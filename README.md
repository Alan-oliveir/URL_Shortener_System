# 🔗 URL Shortener System - Complete Serverless Solution

Sistema completo de encurtamento de URLs desenvolvido com arquitetura serverless na AWS. Composto por dois microserviços independentes que trabalham em conjunto para criar e resolver URLs encurtadas.

![AWS](https://img.shields.io/badge/AWS-Lambda-orange)
![Java](https://img.shields.io/badge/Java-17-blue)
![Maven](https://img.shields.io/badge/Maven-3.6+-green)
![S3](https://img.shields.io/badge/Storage-Amazon%20S3-red)

## Arquitetura do Sistema

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Cliente   │     │   API Gateway   │     │   AWS Lambda    │
│             │───▶|                  │───▶│                 │
│ Navegador/  │     │ - POST /create  │     │  CreateUrl      │
│ App         │     │ - GET /{code}   │     │  RedirectUrl    │
└─────────────┘     └─────────────────┘     └─────────────────┘
                                                    │
                                                    ▼
                                          ┌─────────────────┐
                                          │   Amazon S3     │
                                          │                 │
                                          │ URL Data Store  │
                                          └─────────────────┘
```

## Estrutura do Repositório

```
url-shortener-system/
├── CreateUrlLambda/          # Serviço de criação de URLs
│   ├── src/main/java/
│   ├── pom.xml
│   └── README.md
├── RedirectUrlShortner/      # Serviço de redirecionamento
│   ├── src/main/java/
│   ├── pom.xml
│   └── README.md
└── README.md                 # Este arquivo
```

## Microserviços

### 1. [CreateUrlLambda](./CreateUrlLambda) - Criação de URLs
**Responsabilidade**: Gerar códigos únicos e armazenar URLs encurtadas

- **Endpoint**: `POST /create`
- **Input**: URL original + tempo de expiração
- **Output**: Código da URL encurtada
- **Funcionalidades**:
  - Geração de códigos únicos (8 caracteres)
  - Validação de entrada
  - Armazenamento seguro no S3

### 2. [RedirectUrlShortner](./RedirectUrlShortner) - Resolução de URLs
**Responsabilidade**: Resolver códigos e redirecionar para URLs originais

- **Endpoint**: `GET /{code}`
- **Input**: Código da URL encurtada
- **Output**: Redirecionamento HTTP ou erro
- **Funcionalidades**:
  - Validação de expiração
  - Redirecionamento HTTP 302
  - Tratamento de URLs expiradas (410)

## Tecnologias Utilizadas

| Categoria | Tecnologia | Versão |
|-----------|------------|--------|
| **Linguagem** | Java | 17 |
| **Build Tool** | Maven | 3.6+ |
| **Computação** | AWS Lambda | - |
| **Storage** | Amazon S3 | - |
| **API Gateway** | AWS API Gateway | - |
| **Serialização** | Jackson | 2.12.3 |
| **Utilities** | Lombok | 1.18.34 |

## Fluxo Completo de Funcionamento

### 1. Criação de URL Encurtada
```bash
# Requisição
POST /create
{
  "originalUrl": "https://exemplo.com/artigo-muito-longo-sobre-tecnologia",
  "expirationTime": "1672531200"
}

# Resposta
{
  "code": "a1b2c3d4"
}
```

### 2. Uso da URL Encurtada
```bash
# Requisição
GET /a1b2c3d4

# Resposta (se válida)
HTTP/1.1 302 Found
Location: https://exemplo.com/artigo-muito-longo-sobre-tecnologia

# Resposta (se expirada)
HTTP/1.1 410 Gone
Body: "This URL has expired."
```

## Deploy e Configuração

### Pré-requisitos
- Conta AWS configurada
- AWS CLI instalado
- Java 17+ e Maven 3.6+

### Passos para Deploy

1. **Criar bucket S3**
```bash
aws s3 mb s3://aws-url-shortner-app
```

2. **Build dos projetos**
```bash
# CreateUrlLambda
cd CreateUrlLambda
mvn clean package

# RedirectUrlShortner
cd ../RedirectUrlShortner
mvn clean package
```

3. **Deploy das Lambdas**
- Upload dos JARs via AWS Console ou CLI
- Configurar handlers corretos
- Adicionar permissões IAM necessárias

4. **Configurar API Gateway**
- Criar rotas para ambos os serviços
- Configurar integração com Lambda
- Deploy da API

### Permissões IAM Necessárias

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

## Vantagens da Arquitetura Serverless

- **Escalabilidade Automática**: Ajusta-se automaticamente à demanda
- **Pay-per-Use**: Paga apenas pelas execuções realizadas
- **Zero Maintenance**: Infraestrutura gerenciada pela AWS
- **High Availability**: Disponibilidade garantida pela AWS
- **Microserviços**: Serviços independentes e especializados

## Demonstração de Habilidades

Este projeto demonstra conhecimentos em:

### **Backend Development**
- Java 17 e programação orientada a objetos
- Arquitetura de microserviços
- APIs REST e códigos de status HTTP
- Tratamento de exceções e validação

### **Cloud Computing (AWS)**
- AWS Lambda (Function as a Service)
- Amazon S3 (Object Storage)
- AWS API Gateway
- IAM (Identity and Access Management)

### **DevOps & Build**
- Maven para gerenciamento de dependências
- Build automation e packaging
- Deployment em ambiente cloud

### **Qualidade de Código**
- Lombok para código limpo
- Separação de responsabilidades
- Tratamento robusto de erros

## Melhorias Futuras

- [ ] **Analytics**: Dashboard de métricas de uso
- [ ] **Cache**: Redis para URLs frequentes
- [ ] **Security**: Rate limiting e autenticação
- [ ] **Monitoring**: CloudWatch logs e alerts
- [ ] **Testing**: Testes unitários e de integração
- [ ] **UI**: Interface web para criação de URLs

## Desenvolvedor

Desenvolvido por [Seu Nome] durante evento de programação da Rocketseat.

**Contato**: [seu-email@exemplo.com]
**LinkedIn**: [seu-linkedin]
**GitHub**: [seu-github]

## Licença

Este projeto está sob a licença MIT. Consulte os arquivos LICENSE em cada subprojeto para mais detalhes.

---

> **💡 Nota para Recrutadores**: Este projeto demonstra a capacidade de desenvolver soluções completas utilizando arquitetura serverless moderna, aplicando boas práticas de desenvolvimento e integração com serviços cloud da AWS.
