# 01. Pré-requisitos

Antes de iniciar, certifique-se de que seu ambiente local e sua conta AWS estão corretamente configurados.

## Ferramentas Necessárias

### AWS CLI v2

```bash
# Verificar versão instalada
aws --version
# aws-cli/2.x.x Python/3.x.x ...

# Instalar (Linux/macOS)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verificar
aws --version
```

!!! note "Windows"
    No Windows, use o instalador MSI disponível na [documentação oficial da AWS](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

### kubectl

```bash
# Instalar versão compatível com EKS 1.32+
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verificar
kubectl version --client
```

### Helm

```bash
# Instalar Helm 3
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verificar (requer >= 3.14)
helm version
```

### eksctl

```bash
# Instalar eksctl
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin

# Verificar (requer >= 0.190)
eksctl version
```

## Configuração AWS

### Credenciais

Configure o AWS CLI com suas credenciais:

```bash
aws configure
# AWS Access Key ID: <sua-access-key>
# AWS Secret Access Key: <sua-secret-key>
# Default region name: us-east-1
# Default output format: json
```

!!! warning "Segurança"
    Nunca armazene suas credenciais AWS em arquivos do projeto ou repositórios Git. Use variáveis de ambiente ou o AWS credentials file (`~/.aws/credentials`).

### Permissões IAM Necessárias

O usuário ou role que executará os comandos deste guia precisa das seguintes permissões:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:*",
        "ec2:*",
        "iam:*",
        "kms:*",
        "s3:*",
        "elasticloadbalancing:*",
        "autoscaling:*",
        "cloudformation:*"
      ],
      "Resource": "*"
    }
  ]
}
```

!!! tip "Para produção"
    Em produção, aplique o princípio do menor privilégio e crie roles específicas por responsabilidade. Para este laboratório, a policy acima simplifica o processo.

### Verificar acesso

```bash
aws sts get-caller-identity
```

Saída esperada:

```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "<AWS_ACCOUNT_ID>",
    "Arn": "arn:aws:iam::<AWS_ACCOUNT_ID>:user/<seu-usuario>"
}
```

Salve o valor de `Account` — você precisará dele ao longo do guia como `<AWS_ACCOUNT_ID>`.

## Configuração GitHub

### Personal Access Token (PAT)

O ArgoCD precisará de acesso ao repositório privado `infra-gitops-delivery`. Crie um PAT com escopo `repo`:

1. Acesse **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**
2. Clique em **Generate new token (classic)**
3. Selecione o escopo `repo`
4. Copie o token gerado — você usará na etapa de configuração do ArgoCD

!!! warning "Segurança"
    Armazene o PAT com segurança. Ele será configurado como um Secret no cluster via Vault, nunca diretamente em YAML.

## Variáveis de Referência

Ao longo deste guia, usamos os seguintes placeholders. Substitua pelos valores reais do seu ambiente conforme for avançando:

| Placeholder | Onde obter |
|---|---|
| `<AWS_ACCOUNT_ID>` | `aws sts get-caller-identity --query Account --output text` |
| `<CLUSTER_NAME>` | Definido por você (ex: `eks-gitops-lab`) |
| `<AWS_REGION>` | `us-east-1` |
| `<VPC_ID>` | Obtido na etapa 02 |
| `<KMS_KEY_ARN>` | Obtido na etapa 02 |
| `<HARBOR_S3_BUCKET>` | Definido por você (ex: `harbor-registry-<AWS_ACCOUNT_ID>`) |
| `<GITHUB_USERNAME>` | Seu username no GitHub |
| `<ARGOCD_REPO_URL>` | URL HTTPS do `infra-gitops-delivery` |

## Checklist de Pré-requisitos

- [ ] AWS CLI v2 instalado e configurado
- [ ] kubectl >= 1.32 instalado
- [ ] Helm >= 3.14 instalado
- [ ] eksctl >= 0.190 instalado
- [ ] Credenciais AWS configuradas (`aws sts get-caller-identity` retorna sucesso)
- [ ] GitHub PAT criado com escopo `repo`
- [ ] Repositórios GitHub criados (`eks-gitops-blueprint`, `infra-gitops-delivery-blueprint`, `infra-gitops-delivery`)

Com tudo pronto, avance para a [criação da VPC e Networking](02-vpc-networking.md).
