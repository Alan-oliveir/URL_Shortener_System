# Troubleshooting Guide

Este guia ajuda a identificar e resolver problemas comuns no URL Shortener System.

## Diagnóstico Rápido

### Verificação de Status dos Serviços

=== "Create URL Service"
    ```bash
    # Teste básico de criação
    curl -X POST https://your-api-gateway.com/create \
      -H "Content-Type: application/json" \
      -d '{
        "originalUrl": "https://google.com",
        "expirationTime": "9999999999"
      }'
    ```

=== "Redirect Service"
    ```bash
    # Teste de redirecionamento
    curl -I https://your-api-gateway.com/test-code
    ```

=== "S3 Bucket"
    ```bash
    # Verificar se bucket existe
    aws s3 ls s3://aws-url-shortner-app/
    ```

## Problemas Comuns

### 1. Error 500 - Internal Server Error

#### **Sintomas**
- API retorna erro 500
- Lambda function falha na execução

#### **Possíveis Causas & Soluções**

!!! danger "Permissões IAM Incorretas"
    **Problema**: Lambda não tem permissão para acessar S3
    
    **Solução**:
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "s3:GetObject",
            "s3:PutObject"
          ],
          "Resource": "arn:aws:s3:::aws-url-shortner-app/*"
        }
      ]
    }
    ```

!!! warning "Bucket S3 Não Existe"
    **Problema**: Bucket `aws-url-shortner-app` não foi criado
    
    **Solução**:
    ```bash
    aws s3 mb s3://aws-url-shortner-app
    ```

!!! info "Timeout da Lambda"
    **Problema**: Função excede tempo limite padrão (3s)
    
    **Solução**:
    - Aumentar timeout para 30s no console AWS
    - Ou via CLI: `aws lambda update-function-configuration --function-name YourFunction --timeout 30`

### 2. Error 400 - Bad Request

#### **Sintomas**
- Requisição rejeitada com erro 400
- Mensagem "Error parsing JSON body"

#### **Soluções**

!!! bug "JSON Malformado"
    **Verificar**:
    ```bash
    # ✅ Correto
    curl -X POST https://api.com/create \
      -H "Content-Type: application/json" \
      -d '{"originalUrl":"https://example.com","expirationTime":"1672531200"}'
    
    # ❌ Incorreto (faltam aspas)
    curl -X POST https://api.com/create \
      -H "Content-Type: application/json" \
      -d '{originalUrl:https://example.com,expirationTime:1672531200}'
    ```

!!! warning "Content-Type Ausente"
    Sempre incluir header: `Content-Type: application/json`

### 3. Error 410 - Gone

#### **Sintomas**
- URL retorna "This URL has expired"
- Redirecionamento não funciona

#### **Explicação**
Este é o comportamento esperado para URLs expiradas.

**Verificar tempo de expiração**:
```bash
# Verificar conteúdo do arquivo no S3
aws s3 cp s3://aws-url-shortner-app/your-code.json - | jq .

# Comparar com timestamp atual
date +%s
```

### 4. Error 404 - Not Found

#### **Sintomas**
- Código de URL não encontrado
- Lambda retorna erro ao buscar no S3

#### **Possíveis Causas**

!!! error "Código Inexistente"
    **Problema**: Código nunca foi criado ou foi deletado
    
    **Verificação**:
    ```bash
    aws s3 ls s3://aws-url-shortner-app/ | grep your-code
    ```

!!! warning "Caracteres Especiais na URL"
    **Problema**: Códigos com caracteres especiais podem causar problemas
    
    **Verificação**: Códigos devem conter apenas [a-zA-Z0-9]

### 5. CORS Issues

#### **Sintomas**
- Erro CORS no navegador
- "Access-Control-Allow-Origin" ausente

#### **Solução - API Gateway**
```yaml
# Configurar CORS no API Gateway
Resources:
  ApiGatewayCorsConfig:
    Type: AWS::ApiGateway::Method
    Properties:
      HttpMethod: OPTIONS
      Integration:
        IntegrationResponses:
          - StatusCode: 200
            ResponseParameters:
              method.response.header.Access-Control-Allow-Headers: "'Content-Type,X-Amz-Date,Authorization'"
              method.response.header.Access-Control-Allow-Methods: "'GET,POST,OPTIONS'"
              method.response.header.Access-Control-Allow-Origin: "'*'"
```

## Debugging Detalhado

### Verificar Logs do CloudWatch

=== "Via Console AWS"
    1. Acesse CloudWatch → Log Groups
    2. Procure por `/aws/lambda/your-function-name`
    3. Verifique logs mais recentes

=== "Via CLI"
    ```bash
    # Listar log groups
    aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/"
    
    # Ver logs recentes
    aws logs tail /aws/lambda/your-function-name --follow
    ```

### Testar Lambda Localmente

Para testar sem deploy:

```bash
# Instalar SAM CLI
pip install aws-sam-cli

# Criar template.yaml básico
cat > template.yaml << EOF
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  CreateUrlFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: CreateUrlLambda/target/
      Handler: com.rocketseat.createUrlShortner.Main::handleRequest
      Runtime: java17
      Events:
        CreateUrl:
          Type: Api
          Properties:
            Path: /create
            Method: post
EOF

# Testar localmente
sam local start-api
```

### Validar Configuração do Maven

```bash
# Verificar se JAR foi gerado corretamente
cd CreateUrlLambda
mvn clean package
ls -la target/

# JAR deve existir e ter dependências incluídas (shade plugin)
jar -tf target/CreateUrlLambda-1.0-SNAPSHOT.jar | grep -E "(aws|jackson|lombok)"
```

## Monitoramento e Métricas

### Métricas Importantes

| Métrica | O que indica | Valor normal |
|---------|--------------|--------------|
| **Duration** | Tempo de execução | < 5000ms |
| **Errors** | Falhas na execução | < 1% |
| **Throttles** | Limitação de concorrência | 0 |
| **Invocations** | Número de execuções | Conforme uso |

### Alertas Recomendados

```bash
# Criar alerta para errors > 5%
aws cloudwatch put-metric-alarm \
  --alarm-name "Lambda-High-Error-Rate" \
  --alarm-description "Lambda error rate > 5%" \
  --metric-name "Errors" \
  --namespace "AWS/Lambda" \
  --statistic "Sum" \
  --period 300 \
  --threshold 5 \
  --comparison-operator "GreaterThanThreshold"
```

## Ferramentas de Debug

### 1. Postman Collection

Criar collection para testes:

```json
{
  "info": {
    "name": "URL Shortener Tests"
  },
  "item": [
    {
      "name": "Create URL",
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "application/json"
          }
        ],
        "body": {
          "raw": "{\n  \"originalUrl\": \"https://google.com\",\n  \"expirationTime\": \"9999999999\"\n}"
        },
        "url": {
          "raw": "{{api_url}}/create"
        }
      }
    }
  ]
}
```

### 2. Script de Teste Automatizado

```bash
#!/bin/bash
# test-system.sh

API_URL="https://your-api-gateway.com"

echo "🧪 Testando criação de URL..."
RESPONSE=$(curl -s -X POST $API_URL/create \
  -H "Content-Type: application/json" \
  -d '{"originalUrl":"https://google.com","expirationTime":"9999999999"}')

CODE=$(echo $RESPONSE | jq -r '.code')
echo "✅ Código gerado: $CODE"

echo "🧪 Testando redirecionamento..."
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $API_URL/$CODE)

if [ "$HTTP_STATUS" = "302" ]; then
    echo "✅ Redirecionamento funcionando"
else
    echo "❌ Erro no redirecionamento: $HTTP_STATUS"
fi
```

## Quando Buscar Ajuda

Se após seguir este guia o problema persistir:

1. **Colete informações**:
   - Logs do CloudWatch
   - Código de erro específico
   - Payload da requisição
   - Timestamp do erro

2. **Verifique**:
   - Status dos serviços AWS
   - Quotas e limites da conta
   - Configurações de rede/VPC

3. **Recursos adicionais**:
   - [AWS Lambda Troubleshooting](https://docs.aws.amazon.com/lambda/latest/dg/troubleshooting.html)
   - [S3 Error Codes](https://docs.aws.amazon.com/AmazonS3/latest/API/ErrorResponses.html)
   - [API Gateway Issues](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-troubleshooting.html)
