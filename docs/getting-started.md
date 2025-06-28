# Getting Started

Este guia vai te ajudar a configurar e executar o URL Shortener System do zero. Em poucos minutos você terá um sistema completo de encurtamento de URLs funcionando na AWS.

## Pré-requisitos

Antes de começar, certifique-se de ter os seguintes requisitos:

### Ferramentas Necessárias

- **Java 17+** - [Download aqui](https://adoptium.net/)
- **Maven 3.6+** - [Instalação](https://maven.apache.org/install.html)
- **AWS CLI** - [Configuração](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- **Git** - Para clonar os repositórios

### Conta AWS

!!! warning "Importante"
    Você precisará de uma conta AWS ativa. Alguns serviços podem gerar custos mínimos.

=== "Verificar Java"
    ```bash
    java --version
    # Deve retornar Java 17 ou superior
    ```

=== "Verificar Maven"
    ```bash
    mvn --version
    # Deve retornar Maven 3.6 ou superior
    ```

=== "Verificar AWS CLI"
    ```bash
    aws --version
    # Configure com: aws configure
    ```

## Configuração Rápida

### 1. Clonar o Projeto

```bash
# Clonar repositório principal
git clone https://github.com/seu-usuario/url-shortener-system.git
cd url-shortener-system

# Inicializar submodules
git submodule init
git submodule update
```

### 2. Configurar AWS

#### Criar Bucket S3
```bash
# Criar bucket (nome deve ser único globalmente)
aws s3 mb s3://aws-url-shortner-app-SEU-NOME

# Verificar se foi criado
aws s3 ls | grep url-shortner
```

#### Configurar Permissões IAM

Crie uma role IAM com as seguintes permissões:

```json title="lambda-s3-policy.json"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::aws-url-shortner-app-SEU-NOME/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

### 3. Build dos Microserviços

=== "Create URL Service"
    ```bash
    cd create-url-service
    mvn clean compile
    mvn package
    
    # JAR será gerado em target/CreateUrlLambda-1.0-SNAPSHOT.jar
    ```

=== "Redirect Service"
    ```bash
    cd redirect-url-service
    mvn clean compile
    mvn package
    
    # JAR será gerado em target/RedirectUrlShortner-1.0-SNAPSHOT.jar
    ```

## Deploy na AWS

### 1. Deploy das Funções Lambda

#### Create URL Lambda

1. **Acessar AWS Lambda Console**

2. **Criar Nova Função**
    - Nome: `url-shortener-create`
    - Runtime: `Java 17`
    - Arquitetura: `x86_64`

3. **Upload do Código**
    - Fazer upload do JAR: `CreateUrlLambda-1.0-SNAPSHOT.jar`
    - Handler: `com.rocketseat.createUrlShortner.Main::handleRequest`

4. **Configurações**
    - Timeout: `30 segundos`
    - Memória: `256 MB`
    - Variáveis de ambiente (se necessário)

#### Redirect URL Lambda

1. **Criar Nova Função**
    - Nome: `url-shortener-redirect`
    - Runtime: `Java 17`
    - Arquitetura: `x86_64`

2. **Upload do Código**
    - Fazer upload do JAR: `RedirectUrlShortner-1.0-SNAPSHOT.jar`
    - Handler: `com.rocketseat.redirectUrlShortner.Main::handleRequest`  

### 2. Configurar API Gateway

#### Criar API REST

```bash
# Usando AWS CLI
aws apigateway create-rest-api \
    --name url-shortener-api \
    --description "API para sistema de encurtamento de URLs"
```

#### Configurar Rotas

=== "Rota CREATE"
    - **Método**: `POST`
    - **Path**: `/create`
    - **Integração**: Lambda `url-shortener-create`
    - **CORS**: Habilitado

=== "Rota REDIRECT"
    - **Método**: `GET`
    - **Path**: `/{code}`
    - **Integração**: Lambda `url-shortener-redirect`
    - **CORS**: Habilitado

### 3. Deploy da API

```bash
# Fazer deploy da API
aws apigateway create-deployment \
    --rest-api-id SEU-API-ID \
    --stage-name prod
```

## Testando o Sistema

### 1. Criar URL Encurtada

```bash
# Substitua pela sua URL da API
export API_URL="https://sua-api-id.execute-api.us-east-1.amazonaws.com/prod"

# Criar URL encurtada
curl -X POST $API_URL/create \
  -H "Content-Type: application/json" \
  -d '{
    "originalUrl": "https://github.com/seu-usuario/url-shortener-system",
    "expirationTime": "1735689600"
  }'
```

**Resposta esperada:**
```json
{
  "code": "a1b2c3d4"
}
```

### 2. Testar Redirecionamento

```bash
# Testar redirecionamento
curl -I $API_URL/a1b2c3d4
```

**Resposta esperada:**
```http
HTTP/2 302 
location: https://github.com/seu-usuario/url-shortener-system
```

## Resolução de Problemas Comuns

### Lambda não está executando

!!! bug "Erro: Timeout ou erro de permissão"
    **Solução:**
    
    1. Verificar se a role IAM tem as permissões corretas
    2. Aumentar timeout da função (máximo 15 minutos)
    3. Verificar logs no CloudWatch

### Bucket S3 não encontrado

!!! bug "Erro: NoSuchBucket"
    **Solução:**
    
    1. Verificar se o nome do bucket no código coincide com o criado
    2. Verificar se o bucket está na mesma região da Lambda
    3. Confirmar se o bucket foi criado com sucesso

### API Gateway retorna erro 500

!!! bug "Erro: Internal Server Error"
    **Solução:**
    
    1. Verificar logs da Lambda no CloudWatch
    2. Confirmar se o handler está correto
    3. Testar a Lambda diretamente (sem API Gateway)

## Monitoramento

### CloudWatch Logs

Acesse os logs das suas funções em: 

- **Create URL**: `/aws/lambda/url-shortener-create`  
- **Redirect URL**: `/aws/lambda/url-shortener-redirect`  

Métricas Importantes:

- **Invocations**: Número de execuções
- **Duration**: Tempo de execução
- **Errors**: Número de erros
- **Throttles**: Execuções limitadas

## Próximos Passos

Agora que o sistema está funcionando, você pode:

1. **Explorar a [Arquitetura](architecture.md)** - Entender como tudo funciona
2. **Consultar a [API Reference](api/create-url.md)** - Documentação completa das APIs
3. **Configurar [Monitoramento](advanced/monitoring.md)** - Logs e alertas avançados

---

!!! success "Parabéns!"
    Seu URL Shortener System está funcionando! Agora você tem um sistema serverless completo rodando na AWS.