# IAM Permissions

Este documento detalha todas as permissões IAM necessárias para o funcionamento completo do URL Shortener System na AWS.

## Visão Geral

O sistema requer permissões específicas para cada Lambda function acessar o Amazon S3 e outros serviços AWS necessários.

!!! warning "Princípio do Menor Privilégio"
    Todas as permissões seguem o princípio do menor privilégio, concedendo apenas os acessos mínimos necessários para o funcionamento.

## Permissões por Serviço

### Create URL Lambda

A função de criação de URLs precisa de permissões para **escrever** objetos no S3.

```json title="create-url-lambda-policy.json"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3PutObjectPermission",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::aws-url-shortner-app/*"
    },
    {
      "Sid": "CloudWatchLogsPermission",
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

### Redirect URL Lambda

A função de redirecionamento precisa de permissões para **ler** objetos do S3.

```json title="redirect-url-lambda-policy.json"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3GetObjectPermission",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::aws-url-shortner-app/*"
    },
    {
      "Sid": "CloudWatchLogsPermission",
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

## Roles IAM

### Create URL Lambda Role

```json title="create-url-lambda-role.json"
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
```

### Redirect URL Lambda Role

```json title="redirect-url-lambda-role.json"
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
```

## Configuração via AWS CLI

### 1. Configurar Role para Create URL Lambda

=== "Criar Role"
    ```bash
    # Criar role
    aws iam create-role \
      --role-name CreateUrlLambdaRole \
      --assume-role-policy-document file://create-url-lambda-role.json
    ```

=== "Anexar Policy"
    ```bash
    # Criar policy
    aws iam create-policy \
      --policy-name CreateUrlLambdaPolicy \
      --policy-document file://create-url-lambda-policy.json
    
    # Anexar policy ao role
    aws iam attach-role-policy \
      --role-name CreateUrlLambdaRole \
      --policy-arn arn:aws:iam::ACCOUNT-ID:policy/CreateUrlLambdaPolicy
    ```

### 2. Configurar Role para Redirect URL Lambda

=== "Criar Role"
    ```bash
    # Criar role
    aws iam create-role \
      --role-name RedirectUrlLambdaRole \
      --assume-role-policy-document file://redirect-url-lambda-role.json
    ```

=== "Anexar Policy"
    ```bash
    # Criar policy
    aws iam create-policy \
      --policy-name RedirectUrlLambdaPolicy \
      --policy-document file://redirect-url-lambda-policy.json
    
    # Anexar policy ao role
    aws iam attach-role-policy \
      --role-name RedirectUrlLambdaRole \
      --policy-arn arn:aws:iam::ACCOUNT-ID:policy/RedirectUrlLambdaPolicy
    ```

## Configuração via Console AWS

### Passo 1: Criar Roles

1. Navegue para **IAM** → **Roles** → **Create role**
2. Selecione **AWS service** → **Lambda**
3. Nomeie o role: `CreateUrlLambdaRole` ou `RedirectUrlLambdaRole`

### Passo 2: Criar Policies

1. Navegue para **IAM** → **Policies** → **Create policy**
2. Selecione **JSON** e cole o conteúdo da policy
3. Nomeie adequadamente: `CreateUrlLambdaPolicy` ou `RedirectUrlLambdaPolicy`

### Passo 3: Anexar Policies aos Roles

1. Navegue para **IAM** → **Roles** → Selecione o role
2. **Attach policies** → Selecione a policy correspondente
3. **Attach policy**

## Verificação de Permissões

### Testar Permissões Create URL

```bash
# Testar se pode criar objeto no S3
aws s3 cp test-file.txt s3://aws-url-shortner-app/test-file.txt \
  --profile create-url-lambda
```

### Testar Permissões Redirect URL

```bash
# Testar se pode ler objeto do S3
aws s3 cp s3://aws-url-shortner-app/test-file.txt ./downloaded-file.txt \
  --profile redirect-url-lambda
```

## Matriz de Permissões

| Serviço | Create URL Lambda | Redirect URL Lambda |
|---------|-------------------|---------------------|
| **S3 - GetObject** | ❌ | ✅ |
| **S3 - PutObject** | ✅ | ❌ |
| **S3 - PutObjectAcl** | ✅ | ❌ |
| **CloudWatch Logs** | ✅ | ✅ |

## Troubleshooting

### Erro: AccessDenied no S3

```bash
# Verificar permissões do role
aws iam list-attached-role-policies --role-name CreateUrlLambdaRole

# Verificar conteúdo da policy
aws iam get-policy-version \
  --policy-arn arn:aws:iam::ACCOUNT-ID:policy/CreateUrlLambdaPolicy \
  --version-id v1
```

### Erro: Lambda não consegue escrever logs

```bash
# Verificar se CloudWatch Logs está na policy
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::ACCOUNT-ID:role/CreateUrlLambdaRole \
  --action-names logs:CreateLogGroup logs:CreateLogStream logs:PutLogEvents \
  --resource-arns arn:aws:logs:us-east-1:ACCOUNT-ID:*
```

## Melhores Práticas de Segurança

### 1. Separação de Responsabilidades
- **Create Lambda**: Apenas `PutObject`
- **Redirect Lambda**: Apenas `GetObject`

### 2. Especificidade de Recursos
```json
{
  "Resource": "arn:aws:s3:::aws-url-shortner-app/*"
}
```
!!! tip "Específico vs Genérico"
    Use `aws-url-shortner-app/*` em vez de `*` para limitar acesso apenas ao bucket necessário.

### 3. Monitoramento de Acesso
```json
{
  "Sid": "CloudTrailLogging",
  "Effect": "Allow",
  "Action": [
    "cloudtrail:PutEvents"
  ],
  "Resource": "*"
}
```

## Rotação de Credenciais

Para ambientes de produção, considere implementar:

1. **Rotação automática** de access keys
2. **Temporary credentials** via STS
3. **Cross-account roles** para isolamento

## Checklist de Configuração

- [ ] Roles IAM criados para ambas as Lambdas
- [ ] Policies com permissões mínimas necessárias
- [ ] Policies anexadas aos roles corretos
- [ ] CloudWatch Logs habilitado
- [ ] Bucket S3 criado com nome correto
- [ ] Permissões testadas e validadas
- [ ] Logs de acesso configurados (opcional)

---

!!! success "Configuração Completa"
    Com essas permissões configuradas, o sistema estará pronto para funcionar de forma segura e eficiente na AWS.

**Próximo passo**: Configure o [AWS Setup](aws-setup.md) para deploy completo do sistema.