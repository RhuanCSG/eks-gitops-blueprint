# 05. HashiCorp Vault

Instalamos o Vault em modo **HA com backend Raft integrado** e configuramos **auto-unseal via AWS KMS**. Após a instalação, habilitamos o método de autenticação Kubernetes para o External Secrets Operator.

## Por que Vault HA + Raft?

- **HA**: 3 réplicas garantem operação mesmo com a perda de um node Spot
- **Raft integrado**: sem dependência de DynamoDB ou Consul
- **Auto-unseal com KMS**: pods reiniciam sem intervenção manual

## IAM via EKS Pod Identity

=== "Linux / macOS"

    ```bash
    cat > vault-pod-identity-trust.json << 'EOF'
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {"Service": "pods.eks.amazonaws.com"},
          "Action": ["sts:AssumeRole", "sts:TagSession"]
        }
      ]
    }
    EOF

    VAULT_ROLE_ARN=$(aws iam create-role \
      --role-name "${CLUSTER_NAME}-vault" \
      --assume-role-policy-document file://vault-pod-identity-trust.json \
      --query 'Role.Arn' --output text)

    echo "VAULT_ROLE_ARN=$VAULT_ROLE_ARN"

    cat > vault-kms-policy.json << EOF
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": ["kms:Encrypt", "kms:Decrypt", "kms:DescribeKey"],
          "Resource": "${KMS_KEY_ARN}"
        }
      ]
    }
    EOF

    VAULT_POLICY_ARN=$(aws iam create-policy \
      --policy-name "${CLUSTER_NAME}-vault-kms" \
      --policy-document file://vault-kms-policy.json \
      --query 'Policy.Arn' --output text)

    aws iam attach-role-policy \
      --role-name "${CLUSTER_NAME}-vault" \
      --policy-arn $VAULT_POLICY_ARN

    aws eks create-pod-identity-association \
      --cluster-name $CLUSTER_NAME \
      --namespace vault \
      --service-account vault \
      --role-arn $VAULT_ROLE_ARN \
      --region $AWS_REGION
    ```

=== "Windows (PowerShell)"

    ```powershell
    @'
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {"Service": "pods.eks.amazonaws.com"},
          "Action": ["sts:AssumeRole", "sts:TagSession"]
        }
      ]
    }
    '@ | Set-Content vault-pod-identity-trust.json

    $VAULT_ROLE_ARN = aws iam create-role `
      --role-name "${CLUSTER_NAME}-vault" `
      --assume-role-policy-document file://vault-pod-identity-trust.json `
      --query 'Role.Arn' --output text

    Write-Host "VAULT_ROLE_ARN=$VAULT_ROLE_ARN"

    @"
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": ["kms:Encrypt", "kms:Decrypt", "kms:DescribeKey"],
          "Resource": "$KMS_KEY_ARN"
        }
      ]
    }
    "@ | Set-Content vault-kms-policy.json

    $VAULT_POLICY_ARN = aws iam create-policy `
      --policy-name "${CLUSTER_NAME}-vault-kms" `
      --policy-document file://vault-kms-policy.json `
      --query 'Policy.Arn' --output text

    aws iam attach-role-policy `
      --role-name "${CLUSTER_NAME}-vault" `
      --policy-arn $VAULT_POLICY_ARN

    aws eks create-pod-identity-association `
      --cluster-name $CLUSTER_NAME `
      --namespace vault `
      --service-account vault `
      --role-arn $VAULT_ROLE_ARN `
      --region $AWS_REGION
    ```

## Instalação via Helm

### 1. Adicionar repositório

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

### 2. Criar values do Vault

=== "Linux / macOS"

    ```bash
    cat > vault-values.yaml << EOF
    global:
      enabled: true

    injector:
      enabled: false

    server:
      serviceAccount:
        create: true
        name: vault
        annotations: {}

      ha:
        enabled: true
        replicas: 3
        raft:
          enabled: true
          setNodeId: true
          config: |
            ui = true

            listener "tcp" {
              tls_disable = 1
              address = "[::]:8200"
              cluster_address = "[::]:8201"
            }

            storage "raft" {
              path = "/vault/data"
              retry_join {
                leader_api_addr = "http://vault-0.vault-internal:8200"
              }
              retry_join {
                leader_api_addr = "http://vault-1.vault-internal:8200"
              }
              retry_join {
                leader_api_addr = "http://vault-2.vault-internal:8200"
              }
            }

            seal "awskms" {
              region     = "${AWS_REGION}"
              kms_key_id = "${KMS_KEY_ARN}"
            }

            service_registration "kubernetes" {}

      dataStorage:
        enabled: true
        size: 10Gi
        storageClass: gp3

      affinity: |
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app.kubernetes.io/name: vault
                  component: server
              topologyKey: kubernetes.io/hostname

    ui:
      enabled: true
      serviceType: ClusterIP
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @"
    global:
      enabled: true

    injector:
      enabled: false

    server:
      serviceAccount:
        create: true
        name: vault
        annotations: {}

      ha:
        enabled: true
        replicas: 3
        raft:
          enabled: true
          setNodeId: true
          config: |
            ui = true

            listener "tcp" {
              tls_disable = 1
              address = "[::]:8200"
              cluster_address = "[::]:8201"
            }

            storage "raft" {
              path = "/vault/data"
              retry_join {
                leader_api_addr = "http://vault-0.vault-internal:8200"
              }
              retry_join {
                leader_api_addr = "http://vault-1.vault-internal:8200"
              }
              retry_join {
                leader_api_addr = "http://vault-2.vault-internal:8200"
              }
            }

            seal "awskms" {
              region     = "$AWS_REGION"
              kms_key_id = "$KMS_KEY_ARN"
            }

            service_registration "kubernetes" {}

      dataStorage:
        enabled: true
        size: 10Gi
        storageClass: gp3

      affinity: |
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app.kubernetes.io/name: vault
                  component: server
              topologyKey: kubernetes.io/hostname

    ui:
      enabled: true
      serviceType: ClusterIP
    "@ | Set-Content vault-values.yaml
    ```

### 3. Instalar

=== "Linux / macOS"

    ```bash
    VAULT_VERSION=$(helm search repo hashicorp/vault --output json | jq -r '.[0].version')

    helm install vault hashicorp/vault \
      --namespace vault \
      --version "$VAULT_VERSION" \
      -f vault-values.yaml \
      --wait
    ```

=== "Windows (PowerShell)"

    ```powershell
    $VAULT_VERSION = (helm search repo hashicorp/vault --output json | ConvertFrom-Json)[0].version

    helm install vault hashicorp/vault `
      --namespace vault `
      --version "$VAULT_VERSION" `
      -f vault-values.yaml `
      --wait
    ```

## Inicialização do Vault

=== "Linux / macOS"

    ```bash
    kubectl exec -n vault vault-0 -- vault operator init \
      -recovery-shares=1 \
      -recovery-threshold=1 \
      -format=json > ~/vault-init.json

    VAULT_ROOT_TOKEN=$(cat ~/vault-init.json | jq -r '.root_token')
    echo "VAULT_ROOT_TOKEN=$VAULT_ROOT_TOKEN"
    ```

=== "Windows (PowerShell)"

    ```powershell
    kubectl exec -n vault vault-0 -- vault operator init `
      -recovery-shares=1 `
      -recovery-threshold=1 `
      -format=json | Set-Content ~\vault-init.json

    $VAULT_ROOT_TOKEN = (Get-Content ~\vault-init.json | ConvertFrom-Json).root_token
    Write-Host "VAULT_ROOT_TOKEN=$VAULT_ROOT_TOKEN"
    ```

!!! danger "Root Token"
    Guarde o root token em um gerenciador de senhas. Nunca versione `vault-init.json`.

### Verificar auto-unseal

```bash
kubectl exec -n vault vault-0 -- vault status
```

Saída esperada (`Sealed: false`):

```
Seal Type      awskms
Initialized    true
Sealed         false
HA Mode        active
```

## Configuração do Vault

```bash
kubectl exec -n vault vault-0 -- sh -c "VAULT_TOKEN=$VAULT_ROOT_TOKEN vault secrets enable -path=secret kv-v2"

kubectl exec -n vault vault-0 -- sh -c "VAULT_TOKEN=$VAULT_ROOT_TOKEN vault auth enable kubernetes"

kubectl exec -n vault vault-0 -- sh -c "VAULT_TOKEN=$VAULT_ROOT_TOKEN vault write auth/kubernetes/config kubernetes_host=https://kubernetes.default.svc"
```

### Política para o ESO

=== "Linux / macOS"

    ```bash
    kubectl exec -n vault vault-0 -- vault policy write external-secrets - << 'EOF'
    path "secret/data/*" {
      capabilities = ["read", "list"]
    }
    path "secret/metadata/*" {
      capabilities = ["read", "list"]
    }
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    'path "secret/data/*" { capabilities = ["read", "list"] } path "secret/metadata/*" { capabilities = ["read", "list"] }' | kubectl exec -i -n vault vault-0 -- sh -c "VAULT_TOKEN=$VAULT_ROOT_TOKEN vault policy write external-secrets -"
    ```

### Role Kubernetes para o ESO

```bash
kubectl exec -n vault vault-0 -- sh -c "VAULT_TOKEN=$VAULT_ROOT_TOKEN vault write auth/kubernetes/role/external-secrets bound_service_account_names=external-secrets bound_service_account_namespaces=external-secrets policies=external-secrets ttl=1h"
```

### Armazenar senha do Harbor

```bash
kubectl exec -n vault vault-0 -- sh -c "VAULT_TOKEN=$VAULT_ROOT_TOKEN vault kv put secret/harbor/admin HARBOR_ADMIN_PASSWORD=<HARBOR_ADMIN_PASSWORD>"
```

## Salvar variável do role ARN

=== "Linux / macOS"

    ```bash
    echo "export VAULT_ROLE_ARN=\"$VAULT_ROLE_ARN\"" >> ~/eks-lab-vars.env
    ```

=== "Windows (PowerShell)"

    ```powershell
    "`$VAULT_ROLE_ARN = `"$VAULT_ROLE_ARN`"" | Add-Content ~\eks-lab-vars.ps1
    ```

## Salvar Root Token

=== "Linux / macOS"

    ```bash
    echo "export VAULT_ROOT_TOKEN=\"$VAULT_ROOT_TOKEN\"" >> ~/eks-lab-vars.env
    ```

=== "Windows (PowerShell)"

    ```powershell
    "`$VAULT_ROOT_TOKEN = `"$VAULT_ROOT_TOKEN`"" | Add-Content ~\eks-lab-vars.ps1
    ```

!!! danger "Preserve o root token"
    O root token é necessário em etapas posteriores (configurar ESO, armazenar segredos). Salve-o no arquivo de variáveis do lab e em um gerenciador de senhas. **Nunca versione `vault-init.json`.**

## Checklist

- [ ] Role IAM criada para Vault com acesso ao KMS
- [ ] Pod Identity Association configurada para namespace `vault`
- [ ] Vault instalado com HA + Raft (3 réplicas)
- [ ] Vault inicializado e sem seal (auto-unseal via KMS)
- [ ] Raft cluster com 3 peers (1 leader, 2 followers)
- [ ] Root token salvo em `eks-lab-vars.ps1` (ou `.env`) e em gerenciador de senhas
- [ ] KV secrets engine habilitado em `secret/`
- [ ] Autenticação Kubernetes habilitada
- [ ] Política `external-secrets` e role Kubernetes criadas
- [ ] Senha do Harbor armazenada em `secret/harbor/admin` com chave `HARBOR_ADMIN_PASSWORD`

Próximo passo: [Harbor Registry](06-harbor.md).
