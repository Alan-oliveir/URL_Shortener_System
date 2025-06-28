# System Architecture

Esta página detalha a arquitetura completa do URL Shortener System, incluindo componentes, fluxos de dados e decisões de design.

## Visão Geral da Arquitetura

O sistema foi projetado seguindo os princípios de **arquitetura serverless** e **microserviços**, proporcionando alta escalabilidade, baixo custo operacional e facilidade de manutenção.

```mermaid
graph TB
    subgraph "Cliente"
        C[Browser/App]
    end
    
    subgraph "AWS Cloud"
        subgraph "API Layer"
            AG[API Gateway]
        end
        
        subgraph "Compute Layer"
            L1[Create URL Lambda]
            L2[Redirect URL Lambda]
        end
        
        subgraph "Storage Layer"
            S3[Amazon S3 Bucket]
        end
        
        subgraph "Security Layer"
            IAM[IAM Roles & Policies]
        end
    end
    
    C -->|HTTP Requests| AG
    AG -->|POST /create| L1
    AG -->|GET /code| L2
    L1 -->|Store URL Data| S3
    L2 -->|Retrieve URL Data| S3
    L1 -.->|Permissions| IAM
    L2 -.->|Permissions| IAM
```

## Princípios de Design

### 1. **Serverless First**
- Sem servidores para gerenciar
- Escalabilidade automática
- Modelo pay-per-use

### 2. **Microserviços**
- Separação de responsabilidades
- Deploy independente
- Resiliência isolada

### 3. **Event-Driven**
- Processamento assíncrono
- Baixo acoplamento
- Alta disponibilidade

### 4. **Stateless**
- Sem estado entre requisições
- Escalabilidade horizontal
- Recuperação rápida

## Componentes Detalhados

### API Gateway
**Responsabilidade**: Ponto de entrada único para todas as requisições

```yaml
Configuration:
  Type: REST API
  Endpoints:
    - POST /create     → Create URL Lambda
    - GET /{code}      → Redirect URL Lambda
  Features:
    - Rate limiting
    - Request validation
    - CORS support
    - API keys (opcional)
```

**Benefícios**:  

- Roteamento centralizado  
- Throttling automático  
- Logs de requisições  
- Transformação de dados  

### Create URL Lambda
**Responsabilidade**: Processar criação de URLs encurtadas

```mermaid
flowchart TD
    A[Receber Requisição] --> B{Validar JSON}
    B -->|Válido| C[Extrair originalUrl e expirationTime]
    B -->|Inválido| D[Retornar Erro 400]
    C --> E[Gerar UUID de 8 caracteres]
    E --> F[Criar objeto UrlData]
    F --> G[Serializar para JSON]
    G --> H[Salvar no S3]
    H -->|Sucesso| I[Retornar código]
    H -->|Erro| J[Retornar Erro 500]
```

**Características Técnicas**:  

- **Runtime**: Java 17  
- **Memory**: 512 MB (recomendado)  
- **Timeout**: 30 segundos  
- **Handler**: `com.rocketseat.createUrlShortner.Main::handleRequest`  

### Redirect URL Lambda
**Responsabilidade**: Resolver códigos e redirecionar usuários

```mermaid
flowchart TD
    A[Receber Requisição] --> B{Código válido?}
    B -->|Não| C[Retornar Erro 400]
    B -->|Sim| D[Buscar no S3]
    D -->|Não encontrado| E[Retornar Erro 404]
    D -->|Encontrado| F[Deserializar JSON]
    F --> G{URL expirada?}
    G -->|Sim| H[Retornar 410 Gone]
    G -->|Não| I[Retornar 302 Redirect]
```

**Características Técnicas**:  

- **Runtime**: Java 17  
- **Memory**: 256 MB (suficiente)  
- **Timeout**: 15 segundos  
- **Handler**: `com.rocketseat.redirectUrlShortner.Main::handleRequest`  

### Amazon S3
**Responsabilidade**: Armazenamento persistente de dados das URLs

```yaml
Bucket Configuration:
  Name: aws-url-shortner-app
  Region: us-east-1 (recomendado)
  Versioning: Disabled
  Encryption: AES-256 (server-side)
  
Object Structure:
  Key: "{8-char-code}.json"
  Content: |
    {
      "originalUrl": "https://exemplo.com/url-longa",
      "expirationTime": 1672531200
    }
```

**Benefícios**:  

- Durabilidade de 99.999999999%  
- Escalabilidade ilimitada  
- Baixo custo por GB  
- Integração nativa com Lambda  

## Fluxos de Dados

### Fluxo de Criação de URL

```mermaid
sequenceDiagram
    participant C as Cliente
    participant AG as API Gateway
    participant L1 as Create Lambda
    participant S3 as Amazon S3
    
    C->>AG: POST /create
    Note over C,AG: originalUrl + expirationTime
    
    AG->>L1: Invoke Lambda
    L1->>L1: Parse JSON
    L1->>L1: Generate UUID code
    L1->>L1: Create UrlData object
    
    L1->>S3: PutObject
    Note over L1,S3: Key: code.json
    
    S3-->>L1: Success
    L1-->>AG: Return code
    AG-->>C: 200 OK + response
```

### Fluxo de Redirecionamento

```mermaid
sequenceDiagram
    participant C as Cliente
    participant AG as API Gateway
    participant L2 as Redirect Lambda
    participant S3 as Amazon S3
    
    C->>AG: GET /a1b2c3d4
    AG->>L2: Invoke Lambda
    L2->>L2: Extract code from path
    
    L2->>S3: GetObject
    Note over L2,S3: Key: a1b2c3d4.json
    
    alt Object exists
        S3-->>L2: UrlData JSON
        L2->>L2: Parse JSON
        L2->>L2: Check expiration
        
        alt Not expired
            L2-->>AG: 302 + Location header
            AG-->>C: Redirect to original URL
        else Expired
            L2-->>AG: 410 Gone
            AG-->>C: URL expired message
        end
    else Object not found
        S3-->>L2: NoSuchKey error
        L2-->>AG: 500 Error
        AG-->>C: Internal server error
    end
```

## Segurança e Permissions

### IAM Roles e Policies

#### Create URL Lambda Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::aws-url-shortner-app/*"
    }
  ]
}
```

#### Redirect URL Lambda Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::aws-url-shortner-app/*"
    }
  ]
}
```

### Princípio do Menor Privilégio
- Cada Lambda tem apenas as permissões mínimas necessárias
- Acesso restrito apenas ao bucket específico
- Logs separados por função

## Performance e Escalabilidade

### Características de Performance

| Componente | Latência Típica | Throughput | Escalabilidade |
|------------|-----------------|------------|----------------|
| **API Gateway** | < 10ms | 10,000 RPS | Automática |
| **Create Lambda** | 100-500ms | 1,000 concurrent | Automática |
| **Redirect Lambda** | 50-200ms | 1,000 concurrent | Automática |
| **S3** | < 100ms | Unlimited | Automática |

### Otimizações Implementadas

**Cold Start Mitigation:** 

- Uso de Lombok para reduzir reflection  
- Jackson ObjectMapper reutilizado  
- Dependências mínimas  

**Memory Optimization:**

- Create Lambda: 512MB (processamento JSON)  
- Redirect Lambda: 256MB (operação mais simples)  

**S3 Performance:**  

- Naming strategy com UUID evita hot-spotting  
- Objetos pequenos (< 1KB) para baixa latência  

## Monitoramento e Observabilidade

### CloudWatch Metrics (Automáticas)
- **Duration**: Tempo de execução das Lambdas
- **Invocations**: Número de execuções
- **Errors**: Contagem de erros
- **Throttles**: Requisições limitadas

### Logs Estruturados
```java
// Exemplo de log estruturado
context.getLogger().log(String.format(
    "URL created: code=%s, originalUrl=%s, expirationTime=%d",
    shortUrlCode, originalUrl, expirationTimeInSeconds
));
```

### Alertas Recomendados
- Error rate > 5%
- Duration > 10 segundos
- Throttles > 0

## Benefícios da Arquitetura

### Escalabilidade
- **Automática**: Sem configuração manual
- **Elástica**: Escala para zero quando não há tráfego
- **Global**: Pode ser replicada em múltiplas regiões

### Confiabilidade
- **Sem single point of failure**
- **Retry automático** em falhas transitórias
- **Durabilidade** garantida pelo S3

### Custo-Efetividade
- **Pay-per-use**: Paga apenas pelo que consome
- **Sem infraestrutura ociosa**
- **Custo previsível** baseado no volume

### Developer Experience
- **Deployment simples**: JAR upload
- **Logs centralizados**: CloudWatch
- **Versionamento**: Lambda versions e aliases

## Próximos Passos

Esta arquitetura base permite evoluções como:

- **Cache Layer**: Redis/ElastiCache para URLs frequentes
- **Analytics**: DynamoDB para métricas de uso
- **CDN**: CloudFront para distribuição global
- **Custom Domains**: Route 53 para domínios personalizados
- **Batch Processing**: Limpeza automática de URLs expiradas

---

A arquitetura apresentada demonstra como construir sistemas modernos, escaláveis e cost-effective utilizando os princípios cloud-native e as melhores práticas de desenvolvimento serverless.