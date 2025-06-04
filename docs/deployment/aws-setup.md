# AWS Setup Guide

Este guia detalha como configurar e fazer deploy do URL Shortener System na AWS, incluindo todos os recursos necessários.

## Pré-requisitos

Antes de começar, certifique-se de ter:

- [x] Conta AWS ativa
- [x] AWS CLI instalado e configurado
- [x] Java 17+ instalado
- [x] Maven 3.6+ instalado
- [x] Conhecimentos básicos de AWS Lambda e S3

## Recursos AWS Necessários

| Serviço | Finalidade | Estimativa de Custo |
|---------|------------|-------------------|
| **S3 Bucket** | Armazenamento de URLs | ~$0.01/mês |
| **Lambda (2 funções)** | Processamento | Free tier até 1M requests |
| **API Gateway** | Roteamento HTTP | Free tier até 1M requests |
| **CloudWatch** | Logs e monitoramento | ~$0.50/mês |

!!! info "Free Tier"
    Para desenvolvimento e portfólio, o uso ficará dentro do Free Tier da AWS na maioria dos casos.

## 1. Configuração do AWS CLI

### Verificar instalação
```bash
aws --version
# aws-cli/2.x.x Python/3.x.x
```

### Configurar credenciais
```bash
aws configure
# AWS Access Key ID: [sua-access-key]
# AWS Secret Access Key: [sua-secret-key]
# Default region name: us-east-1
# Default output format: json
```

### Testar conectividade
```bash
aws sts get-caller-identity
```

## 2. Criação do S3 Bucket

### 2.1 Criar bucket via CLI
```bash
# Substitua por um nome único
BUCKET_NAME="aws-url-shortner-app-$(date +%s)"

# Criar bucket
aws s3 mb s3://$BUCKET_NAME --region us-east-1

# Verificar criação
aws s3 ls | grep $BUCKET_NAME
```

### 2.2 Configurar política do bucket
```bash
# Criar arquivo de política
cat > bucket-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "LambdaAccessOnly",
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::BUCKET_NAME/*"
        }
    ]
}
EOF

# Substituir nome do bucket na política
sed -i "s/BUCKET_NAME/$BUCKET_NAME/g" bucket-policy.json

# Aplicar política
aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy file://bucket-policy.json
```

!!! warning "Nome do Bucket"
    Lembre-se de atualizar o nome do bucket (`aws-url-shortner-app`) no código das Lambdas para o nome gerado.

## 3. Build dos Projetos

### 3.1 Build do Create URL Service
```bash
cd create-url-service
mvn clean package

# Verificar se o JAR foi gerado
ls -la target/CreateUrlLambda-1.0-SNAPSHOT.jar
```

### 3.2 Build do Redirect Service
```bash
cd ../redirect-url-service
mvn clean package

# Verificar se o JAR foi gerado
ls -la target/RedirectUrlShortner-1.0-SNAPSHOT.jar
```

## 4. Criação das IAM Roles

### 4.1 Role para Create URL Lambda
```bash
# Criar assume role policy
cat > lambda-assume-role-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

# Criar role
aws iam create-role \
    --role-name URLShortenerLambdaRole \
    --assume-role-policy-document file://lambda-assume-role-policy.json

# Anexar política básica do Lambda
aws iam attach-role-policy \
    --role-name URLShortenerLambdaRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

### 4.2 Política personalizada para S3
```bash
# Criar política S3
cat > s3-access-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::BUCKET_NAME/*"
        }
    ]
}
EOF

# Substituir nome do bucket
sed -i "s/BUCKET_NAME/$BUCKET_NAME/g" s3-access-policy.json

# Criar e anexar política
aws iam create-policy \
    --policy-name URLShortenerS3Policy \
    --policy-document file://s3-access-policy.json

# Obter ARN da conta
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Anexar à role
aws iam attach-role-policy \
    --role-name URLShortenerLambdaRole \
    --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/URLShortenerS3Policy
```

## 5. Deploy das Lambda Functions

### 5.1 Deploy Create URL Lambda
```bash
# Obter ARN da role
ROLE_ARN=$(aws iam get-role --role-name URLShortenerLambdaRole --query 'Role.Arn' --output text)

# Criar função Lambda
aws lambda create-function \
    --function-name CreateUrlShortener \
    --runtime java17 \
    --role $ROLE_ARN \
    --handler com.rocketseat.createUrlShortner.Main::handleRequest \
    --zip-file fileb://create-url-service/target/CreateUrlLambda-1.0-SNAPSHOT.jar \
    --timeout 30 \
    --memory-size 512 \
    --environment Variables='{BUCKET_NAME='$BUCKET_NAME'}'
```

### 5.2 Deploy Redirect Lambda
```bash
# Criar função Lambda
aws lambda create-function \
    --function-name RedirectUrlShortener \
    --runtime java17 \
    --role $ROLE_ARN \
    --handler com.rocketseat.redirectUrlShortner.Main::handleRequest \
    --zip-file fileb://redirect-url-service/target/RedirectUrlShortner-1.0-SNAPSHOT.jar \
    --timeout 30 \
    --memory-size 512 \
    --environment Variables='{BUCKET_NAME='$BUCKET_NAME'}'
```

### 5.3 Testar as funções
```bash
# Teste básico da Create URL
aws lambda invoke \
    --function-name CreateUrlShortener \
    --payload '{"body": "{\"originalUrl\":\"https://example.com\",\"expirationTime\":\"999999999999\"}"}' \
    response.json

cat response.json

# Teste básico da Redirect (usar código retornado)
aws lambda invoke \
    --function-name RedirectUrlShortener \
    --payload '{"rawPath": "/CODIGO_RETORNADO"}' \
    redirect-response.json

cat redirect-response.json
```

## 6. Configuração do API Gateway

### 6.1 Criar API REST
```bash
# Criar API
API_ID=$(aws apigateway create-rest-api \
    --name URLShortenerAPI \
    --description "API for URL Shortener System" \
    --query 'id' --output text)

echo "API ID: $API_ID"

# Obter Root Resource ID
ROOT_RESOURCE_ID=$(aws apigateway get-resources \
    --rest-api-id $API_ID \
    --query 'items[0].id' --output text)
```

### 6.2 Configurar rota para criação (/create)
```bash
# Criar resource /create
CREATE_RESOURCE_ID=$(aws apigateway create-resource \
    --rest-api-id $API_ID \
    --parent-id $ROOT_RESOURCE_ID \
    --path-part create \
    --query 'id' --output text)

# Criar método POST
aws apigateway put-method \
    --rest-api-id $API_ID \
    --resource-id $CREATE_RESOURCE_ID \
    --http-method POST \
    --authorization-type NONE

# Integrar com Lambda
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws apigateway put-integration \
    --rest-api-id $API_ID \
    --resource-id $CREATE_RESOURCE_ID \
    --http-method POST \
    --type AWS_PROXY \
    --integration-http-method POST \
    --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:$ACCOUNT_ID:function:CreateUrlShortener/invocations

# Dar permissão ao API Gateway
aws lambda add-permission \
    --function-name CreateUrlShortener \
    --statement-id apigateway-invoke \
    --action lambda:InvokeFunction \
    --principal apigateway.amazonaws.com \
    --source-arn "arn:aws:execute-api:us-east-1:$ACCOUNT_ID:$API_ID/*/POST/create"
```

### 6.3 Configurar rota para redirecionamento (/{code})
```bash
# Criar resource {code}
REDIRECT_RESOURCE_ID=$(aws apigateway create-resource \
    --rest-api-id $API_ID \
    --parent-id $ROOT_RESOURCE_ID \
    --path-part '{code}' \
    --query 'id' --output text)

# Criar método GET
aws apigateway put-method \
    --rest-api-id $API_ID \
    --resource-id $REDIRECT_RESOURCE_ID \
    --http-method GET \
    --authorization-type NONE

# Integrar com Lambda
aws apigateway put-integration \
    --rest-api-id $API_ID \
    --resource-id $REDIRECT_RESOURCE_ID \
    --http-method GET \
    --type AWS_PROXY \
    --integration-http-method POST \
    --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:$ACCOUNT_ID:function:RedirectUrlShortener/invocations

# Dar permissão ao API Gateway
aws lambda add-permission \
    --function-name RedirectUrlShortener \
    --statement-id apigateway-invoke \
    --action lambda:InvokeFunction \
    --principal apigateway.amazonaws.com \
    --source-arn "arn:aws:execute-api:us-east-1:$ACCOUNT_ID:$API_ID/*/GET/*"
```

### 6.4 Deploy da API
```bash
# Criar deployment
aws apigateway create-deployment \
    --rest-api-id $API_ID \
    --stage-name prod

# Obter URL da API
API_URL="https://$API_ID.execute-api.us-east-1.amazonaws.com/prod"
echo "API URL: $API_URL"
```

## 7. Teste da API Completa

### 7.1 Testar criação de URL
```bash
# Criar URL encurtada
curl -X POST $API_URL/create \
    -H "Content-Type: application/json" \
    -d '{
        "originalUrl": "https://github.com/seu-usuario/url-shortener-system",
        "expirationTime": "999999999999"
    }'

# Resposta esperada: {"code":"a1b2c3d4"}
```

### 7.2 Testar redirecionamento
```bash
# Testar redirecionamento (substitua pelo código retornado)
curl -I $API_URL/a1b2c3d4

# Resposta esperada: HTTP/1.1 302 Found
#                   Location: https://github.com/seu-usuario/url-shortener-system
```

## 8. Configuração de Monitoramento

### 8.1 Habilitar logs detalhados
```bash
# Criar role para API Gateway logs
cat > apigateway-logs-role.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "apigateway.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

aws iam create-role \
    --role-name APIGatewayLogsRole \
    --assume-role-policy-document file://apigateway-logs-role.json

aws iam attach-role-policy \
    --role-name APIGatewayLogsRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs
```

### 8.2 Configurar CloudWatch
```bash
# Obter ARN da role de logs
LOGS_ROLE_ARN=$(aws iam get-role --role-name APIGatewayLogsRole --query 'Role.Arn' --output text)

# Configurar logs no API Gateway
aws apigateway update-account \
    --patch-ops op=replace,path=/cloudwatchRoleArn,value=$LOGS_ROLE_ARN
```

## 9. Limpeza de Recursos (Opcional)

Para remover todos os recursos criados:

```bash
# Deletar API Gateway
aws apigateway delete-rest-api --rest-api-id $API_ID

# Deletar Lambda functions
aws lambda delete-function --function-name CreateUrlShortener
aws lambda delete-function --function-name RedirectUrlShortener

# Deletar S3 bucket (deve estar vazio)
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME

# Deletar IAM roles e políticas
aws iam detach-role-policy --role-name URLShortenerLambdaRole --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/URLShortenerS3Policy
aws iam delete-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/URLShortenerS3Policy
aws iam detach-role-policy --role-name URLShortenerLambdaRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name URLShortenerLambdaRole
```

## Próximos Passos

Após completar o setup:

1. **[Configurar permissões IAM detalhadas](iam-permissions.md)**
2. **[Implementar monitoramento](../advanced/monitoring.md)**
3. **[Configurar CI/CD](cicd.md)**

## Troubleshooting

Problemas comuns e soluções:

!!! question "Lambda timeout"
    Aumente o timeout das funções Lambda se necessário:
    ```bash
    aws lambda update-function-configuration \
        --function-name CreateUrlShortener \
        --timeout 60
    ```

!!! question "Permissões S3"
    Verifique se o nome do bucket no código corresponde ao criado.

!!! question "CORS Issues"
    Configure CORS no API Gateway se necessário para chamadas do browser.

---

**Estimativa de tempo**: 30-45 minutos para setup completo
**Custo estimado**: <$1/mês para uso de desenvolvimento