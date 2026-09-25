# AWS Serverless Static Website + Contact API

## 📌 Descrizione del progetto

Questo progetto realizza una piccola architettura **serverless su AWS**
per pubblicare una pagina web statica e gestire l'invio di un form di
contatto.

L'intera infrastruttura viene creata tramite **AWS CloudFormation**,
quindi le risorse possono essere create e gestite come codice
(**Infrastructure as Code - IaC**).

L'architettura è composta da:

-   **Amazon S3** → contiene i file statici del frontend.
-   **Amazon CloudFront** → distribuisce il frontend tramite HTTPS e
    utilizza un **Origin Access Control (OAC)** per accedere
    privatamente al bucket S3.
-   **AWS Lambda** → riceve i dati del form di contatto.
-   **Lambda Function URL** → espone pubblicamente l'endpoint HTTP della
    Lambda.
-   **Amazon DynamoDB** → salva i dati inviati dal form.
-   **AWS IAM** → definisce i permessi necessari alla Lambda.
-   **AWS CloudFormation** → crea tutte le risorse automaticamente.

------------------------------------------------------------------------

## 🏗️ Architettura

``` text
                         INTERNET
                            │
                            │ HTTPS
                            ▼
                  ┌─────────────────────┐
                  │    CloudFront       │
                  │                     │
                  │  Distribution       │
                  └─────────┬───────────┘
                            │
                ┌───────────┴────────────┐
                │                        │
          /*    │                        │ /api/*
                ▼                        ▼
      ┌─────────────────┐       ┌─────────────────────┐
      │   S3 Bucket     │       │ Lambda Function URL │
      │                 │       │                     │
      │ Static Website  │       │ POST /api/*         │
      └─────────────────┘       └──────────┬──────────┘
                                           │
                                           │ PutItem
                                           ▼
                                  ┌──────────────────┐
                                  │    DynamoDB      │
                                  │                  │
                                  │ Contact table    │
                                  └──────────────────┘
```

### Flusso del frontend

Il browser richiede la pagina tramite il dominio CloudFront:

``` text
Browser
   │
   │ HTTPS
   ▼
CloudFront
   │
   │ OAC + SigV4
   ▼
S3
   │
   ▼
home_page.html
```

Il bucket S3 **non è pubblico**. CloudFront accede agli oggetti tramite
**Origin Access Control (OAC)**.

### Flusso del form di contatto

Quando l'utente invia il form:

``` text
Browser
   │
   │ HTTPS POST
   ▼
CloudFront /api/*
   │
   ▼
Lambda Function URL
   │
   │ Invoke
   ▼
Lambda
   │
   │ dynamodb:PutItem
   ▼
DynamoDB
```

La Lambda valida i dati ricevuti e salva un elemento nella tabella
`contact`.

------------------------------------------------------------------------

# ☁️ Risorse AWS

## 1. Amazon S3

CloudFormation crea il bucket:

``` text
emanuele-bucket-web-047719652891
```

Il bucket viene utilizzato per conservare i file statici del frontend,
ad esempio:

``` text
home_page.html
CSS
JavaScript
immagini
```

Il bucket ha il **Public Access Block** completamente abilitato:

``` yaml
BlockPublicAcls: true
BlockPublicPolicy: true
IgnorePublicAcls: true
RestrictPublicBuckets: true
```

Questo significa che il bucket non viene esposto direttamente a
Internet.

L'accesso agli oggetti viene consentito tramite CloudFront.

------------------------------------------------------------------------

# 🌐 Amazon CloudFront

CloudFront rappresenta il punto di ingresso pubblico dell'applicazione.

La distribution utilizza due origin:

``` text
S3Origin
LambdaOrigin
```

### S3 Origin

Per il traffico normale:

``` text
/*
```

CloudFront utilizza il bucket S3 come origin.

La configurazione:

``` yaml
DefaultRootObject: home_page.html
```

permette di servire `home_page.html` come pagina principale.

Il comportamento predefinito permette:

``` text
GET
HEAD
```

e utilizza la policy AWS Managed:

``` text
CachingOptimized
```

con ID:

``` text
658327ea-f89d-4fab-a63d-7e88639e58f6
```

------------------------------------------------------------------------

## 🔐 CloudFront Origin Access Control

Il progetto utilizza:

``` text
AWS::CloudFront::OriginAccessControl
```

con:

``` yaml
OriginAccessControlOriginType: s3
SigningBehavior: always
SigningProtocol: sigv4
```

CloudFront firma quindi le richieste verso S3 utilizzando **AWS
Signature Version 4**.

La bucket policy permette a CloudFront di leggere gli oggetti:

``` text
s3:GetObject
```

ma limita l'accesso alla specifica CloudFront Distribution tramite:

``` text
AWS:SourceArn
```

Il risultato è:

``` text
Internet
   │
   ▼
CloudFront
   │
   │ autorizzato tramite OAC
   ▼
S3
```

e non:

``` text
Internet ───────► S3
```

------------------------------------------------------------------------

# ⚡ AWS Lambda

La funzione Lambda viene creata con:

``` text
FunctionName: emanuele-contact-api
Runtime: Python 3.12
Handler: index.lambda_handler
Timeout: 10 seconds
```

La funzione riceve il payload del form, ad esempio:

``` json
{
  "email": "utente@example.com",
  "name": "Mario",
  "message": "Messaggio di prova"
}
```

La Lambda estrae:

``` text
email
name
message
```

e verifica che siano presenti:

``` text
email
message
```

Se mancano i campi obbligatori restituisce:

``` http
400 Bad Request
```

con:

``` json
{
  "ok": false,
  "error": "Missing fields"
}
```

Se la scrittura su DynamoDB ha successo:

``` http
200 OK
```

con:

``` json
{
  "ok": true
}
```

In caso di errore:

``` http
500 Internal Server Error
```

------------------------------------------------------------------------

# 🔗 Lambda Function URL

La Lambda viene esposta tramite una **Lambda Function URL**:

``` yaml
AuthType: NONE
```

Questo permette di invocare la funzione tramite un endpoint HTTPS senza
dover utilizzare API Gateway.

Il progetto configura anche CORS:

``` yaml
AllowOrigins:
  - "*"

AllowMethods:
  - POST

AllowHeaders:
  - "*"
```

L'endpoint viene utilizzato come origin di CloudFront.

CloudFront definisce infatti:

``` text
LambdaOrigin
```

e utilizza:

``` yaml
OriginProtocolPolicy: https-only
```

per comunicare con la Function URL tramite HTTPS.

------------------------------------------------------------------------

# 🔀 CloudFront Routing

La distribution utilizza due comportamenti principali.

## Frontend

Tutte le richieste che non corrispondono al comportamento `/api/*`
utilizzano:

``` text
S3Origin
```

Schema:

``` text
https://<cloudfront-domain>/
             │
             ▼
          S3Origin
```

## API

Le richieste:

``` text
/api/*
```

vengono indirizzate a:

``` text
LambdaOrigin
```

Schema:

``` text
https://<cloudfront-domain>/api/*
             │
             ▼
       LambdaOrigin
             │
             ▼
    Lambda Function URL
```

Per l'API il caching è disabilitato tramite la AWS Managed Cache Policy:

``` text
CachingDisabled
```

ID:

``` text
4135ea2d-6df8-44a3-9df3-4b5a84be39ad
```

Sono consentiti:

``` text
GET
HEAD
OPTIONS
POST
PUT
PATCH
DELETE
```

mentre vengono memorizzati in cache solamente:

``` text
GET
HEAD
```

------------------------------------------------------------------------

# 🗄️ Amazon DynamoDB

La tabella viene creata con:

``` text
TableName: contact
BillingMode: PAY_PER_REQUEST
```

La chiave primaria è composta da:

``` text
Partition Key:
email (String)

Sort Key:
timestamp (String)
```

Schema:

``` text
contact
├── email       ← Partition Key
├── timestamp   ← Sort Key
├── name
└── message
```

La modalità:

``` text
PAY_PER_REQUEST
```

permette di pagare in base alle richieste effettuate senza configurare
capacità di lettura/scrittura provisioned.

La Lambda utilizza:

``` text
dynamodb:PutItem
```

per inserire i dati.

------------------------------------------------------------------------

# 🔐 IAM

La Lambda utilizza il ruolo:

``` text
Emanuele-artisana-lambda-role
```

Il ruolo può essere assunto dal servizio Lambda tramite:

``` text
lambda.amazonaws.com
```

## Permessi DynamoDB

La policy permette esclusivamente:

``` text
dynamodb:PutItem
```

sulla tabella `contact`.

La risorsa non è `*`, ma viene ricavata direttamente dalla tabella
CloudFormation:

``` yaml
Resource: !GetAtt ContactTable.Arn
```

Questo limita il permesso alla specifica tabella.

## Permessi CloudWatch Logs

La Lambda dispone inoltre dei permessi necessari per scrivere i propri
log:

``` text
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
```

------------------------------------------------------------------------

# 📝 CloudFormation

Tutte le risorse vengono definite nel template CloudFormation.

La struttura principale è:

``` text
CloudFormation Stack
│
├── S3 Bucket
├── CloudFront Origin Access Control
├── DynamoDB Table
├── IAM Role
├── Lambda Function
├── Lambda Function URL
├── Lambda Permissions
├── CloudFront Distribution
└── S3 Bucket Policy
```

Questo permette di creare l'infrastruttura senza configurare manualmente
ogni servizio dalla AWS Console.

CloudFormation gestisce inoltre le dipendenze tra le risorse.

Ad esempio:

``` yaml
Role: !GetAtt LambdaRole.Arn
```

fa utilizzare alla Lambda l'ARN del ruolo creato dallo stesso stack.

Analogamente:

``` yaml
Resource: !GetAtt ContactTable.Arn
```

permette alla policy IAM di riferirsi direttamente alla tabella DynamoDB
creata dallo stack.

------------------------------------------------------------------------

# 🚀 Deploy

Il progetto può essere distribuito tramite AWS CloudFormation.

Esempio con AWS CLI:

``` bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name project-1 \
  --capabilities CAPABILITY_NAMED_IAM
```

La capability:

``` text
CAPABILITY_NAMED_IAM
```

è necessaria perché il template crea un IAM Role con un nome specifico:

``` text
Emanuele-artisana-lambda-role
```

Al termine del deploy è possibile recuperare l'URL CloudFront dagli
output dello stack.

``` bash
aws cloudformation describe-stacks \
  --stack-name project-1 \
  --query "Stacks[0].Outputs"
```

L'output `WebsiteURL` restituisce:

``` text
https://<cloudfront-domain>
```

------------------------------------------------------------------------

# 🧪 Test

## Test del frontend

Aprire:

``` text
https://<cloudfront-domain>
```

CloudFront dovrebbe restituire:

``` text
home_page.html
```

proveniente dal bucket S3.

## Test del form

Dal frontend viene effettuata una richiesta:

``` http
POST /api/...
```

La richiesta segue il percorso:

``` text
Browser
   ↓
CloudFront
   ↓
Lambda Function URL
   ↓
Lambda
   ↓
DynamoDB
```

Dopo l'invio è possibile verificare la presenza del record nella
tabella:

``` text
contact
```



------------------------------------------------------------------------

# 🔒 Considerazioni di sicurezza

Il progetto utilizza diversi meccanismi di sicurezza:

### S3

Il bucket non è pubblico:

``` text
Public Access Block = enabled
```

L'accesso viene effettuato tramite CloudFront OAC.

### CloudFront → S3

L'accesso è autorizzato tramite:

``` text
Origin Access Control
+
AWS Signature V4
+
S3 Bucket Policy
```

### Lambda → DynamoDB

La Lambda dispone solamente del permesso:

``` text
dynamodb:PutItem
```

sulla tabella specifica.

### HTTPS

CloudFront espone il frontend tramite HTTPS.

La comunicazione CloudFront → Lambda Function URL è configurata con:

``` text
https-only
```

### CORS

Il progetto attualmente utilizza:

``` text
AllowOrigins: "*"
```

per semplificare il test del progetto.

In un ambiente reale sarebbe preferibile limitare `AllowOrigins` al
dominio effettivamente utilizzato dal frontend.

------------------------------------------------------------------------

# ⚠️ Nota sulla Function URL

La Function URL è configurata con:

``` yaml
AuthType: NONE
```

quindi l'endpoint Lambda è pubblico.

Il progetto è pensato principalmente come esercizio dimostrativo di:

-   CloudFormation
-   CloudFront
-   S3
-   OAC
-   Lambda
-   Lambda Function URL
-   DynamoDB
-   IAM
-   CORS
-   routing CloudFront

Per un ambiente produttivo sarebbe opportuno valutare ulteriori
controlli di sicurezza e protezione dell'endpoint API.

------------------------------------------------------------------------

# 🎯 Obiettivi didattici

Questo progetto permette di esercitarsi con:

-   Infrastructure as Code tramite **CloudFormation**
-   Hosting di contenuti statici con **Amazon S3**
-   Distribuzione globale tramite **CloudFront**
-   Accesso privato a S3 tramite **Origin Access Control**
-   Routing di CloudFront verso più origin
-   API serverless tramite **Lambda Function URL**
-   Elaborazione di richieste HTTP con Lambda
-   Persistenza dati con **DynamoDB**
-   IAM Least Privilege
-   CORS
-   HTTPS
-   CloudWatch Logs
-   Output e riferimenti tra risorse CloudFormation

------------------------------------------------------------------------

## 📁 Struttura concettuale del progetto

``` text
project-1/
│
├── template.yaml
├── README.md
└── frontend/
    ├── home_page.html
    ├── css/
    ├── js/
    └── ...
```

Il file `template.yaml` contiene la definizione dell'infrastruttura AWS,
mentre il frontend contiene i file statici caricati nel bucket S3.

------------------------------------------------------------------------

## 🧠 Sintesi

Il progetto implementa una semplice applicazione web serverless:

``` text
                    FRONTEND
                       │
                       ▼
                 CloudFront
                  /       \
                 /         \
                ▼           ▼
               S3         Lambda
                            │
                            ▼
                         DynamoDB
```

CloudFormation permette di creare l'intera infrastruttura come codice,
mentre CloudFront rappresenta il punto di ingresso dell'applicazione e
instrada le richieste verso l'origin appropriato.
