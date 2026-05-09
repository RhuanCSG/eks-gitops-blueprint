# 06. Harbor Registry

Instalamos o Harbor como registry privado de imagens com backend S3 para persistência.

## Pré-requisito: Service Accounts

O Helm chart do Harbor usa os Service Accounts `harbor-core` e `harbor-registry` quando os nomes são customizados no values.yaml. Esses SAs precisam existir **antes** da instalação — caso contrário o harbor-core e o harbor-registry não sobem.

```bash
kubectl create serviceaccount harbor-core -n harbor
kubectl create serviceaccount harbor-registry -n harbor
```

## IAM via EKS Pod Identity

=== "Linux / macOS"

    ```bash
    cat > harbor-pod-identity-trust.json << 'EOF'
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

    HARBOR_ROLE_ARN=$(aws iam create-role \
      --role-name "${CLUSTER_NAME}-harbor" \
      --assume-role-policy-document file://harbor-pod-identity-trust.json \
      --query 'Role.Arn' --output text)

    cat > harbor-s3-policy.json << EOF
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "s3:GetObject", "s3:PutObject", "s3:DeleteObject",
            "s3:ListBucket", "s3:GetBucketLocation"
          ],
          "Resource": [
            "arn:aws:s3:::${HARBOR_S3_BUCKET}",
            "arn:aws:s3:::${HARBOR_S3_BUCKET}/*"
          ]
        }
      ]
    }
    EOF

    HARBOR_POLICY_ARN=$(aws iam create-policy \
      --policy-name "${CLUSTER_NAME}-harbor-s3" \
      --policy-document file://harbor-s3-policy.json \
      --query 'Policy.Arn' --output text)

    aws iam attach-role-policy \
      --role-name "${CLUSTER_NAME}-harbor" \
      --policy-arn $HARBOR_POLICY_ARN

    for SA in harbor-core harbor-registry; do
      aws eks create-pod-identity-association \
        --cluster-name $CLUSTER_NAME \
        --namespace harbor \
        --service-account $SA \
        --role-arn $HARBOR_ROLE_ARN \
        --region $AWS_REGION
    done
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
    '@ | Set-Content harbor-pod-identity-trust.json

    $HARBOR_ROLE_ARN = aws iam create-role `
      --role-name "${CLUSTER_NAME}-harbor" `
      --assume-role-policy-document file://harbor-pod-identity-trust.json `
      --query 'Role.Arn' --output text

    @"
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "s3:GetObject", "s3:PutObject", "s3:DeleteObject",
            "s3:ListBucket", "s3:GetBucketLocation"
          ],
          "Resource": [
            "arn:aws:s3:::$HARBOR_S3_BUCKET",
            "arn:aws:s3:::${HARBOR_S3_BUCKET}/*"
          ]
        }
      ]
    }
    "@ | Set-Content harbor-s3-policy.json

    $HARBOR_POLICY_ARN = aws iam create-policy `
      --policy-name "${CLUSTER_NAME}-harbor-s3" `
      --policy-document file://harbor-s3-policy.json `
      --query 'Policy.Arn' --output text

    aws iam attach-role-policy `
      --role-name "${CLUSTER_NAME}-harbor" `
      --policy-arn $HARBOR_POLICY_ARN

    foreach ($SA in @("harbor-core", "harbor-registry")) {
      aws eks create-pod-identity-association `
        --cluster-name $CLUSTER_NAME `
        --namespace harbor `
        --service-account $SA `
        --role-arn $HARBOR_ROLE_ARN `
        --region $AWS_REGION
    }
    ```

## Instalação via Helm

### 1. Adicionar repositório

```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
```

### 2. Criar values do Harbor

=== "Linux / macOS"

    ```bash
    HARBOR_ADMIN_PASSWORD=$(openssl rand -base64 24)
    echo "HARBOR_ADMIN_PASSWORD=$HARBOR_ADMIN_PASSWORD"

    cat > harbor-values.yaml << EOF
    expose:
      type: clusterIP
      tls:
        enabled: false

    externalURL: https://<ALB_DNS_NAME>

    persistence:
      enabled: true
      resourcePolicy: "keep"
      imageChartStorage:
        type: s3
        s3:
          region: ${AWS_REGION}
          bucket: ${HARBOR_S3_BUCKET}
          regionendpoint: https://s3.${AWS_REGION}.amazonaws.com

    database:
      type: internal
      internal:
        storageClass: gp3
        size: 5Gi

    redis:
      type: internal
      internal:
        storageClass: gp3

    trivy:
      enabled: true
      storageClass: gp3

    # Senha gerenciada via Vault + ESO
    existingSecretAdminPassword: harbor-admin-secret
    existingSecretAdminPasswordKey: HARBOR_ADMIN_PASSWORD
    metrics:
      enabled: false
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @"
    expose:
      type: clusterIP
      tls:
        enabled: false

    externalURL: https://<ALB_DNS_NAME>

    persistence:
      enabled: true
      resourcePolicy: "keep"
      imageChartStorage:
        type: s3
        s3:
          region: $AWS_REGION
          bucket: $HARBOR_S3_BUCKET
          regionendpoint: https://s3.${AWS_REGION}.amazonaws.com

    database:
      type: internal
      internal:
        storageClass: gp3
        size: 5Gi

    redis:
      type: internal
      internal:
        storageClass: gp3

    trivy:
      enabled: true
      storageClass: gp3

    # Senha gerenciada via Vault + ESO
    existingSecretAdminPassword: harbor-admin-secret
    existingSecretAdminPasswordKey: HARBOR_ADMIN_PASSWORD
    metrics:
      enabled: false
    "@ | Set-Content harbor-values.yaml
    ```

### 3. Instalar

=== "Linux / macOS"

    ```bash
    HARBOR_VERSION=$(helm search repo harbor/harbor --output json | jq -r '.[0].version')

    helm install harbor harbor/harbor \
      --namespace harbor \
      --version "$HARBOR_VERSION" \
      -f harbor-values.yaml \
      --wait --timeout 10m
    ```

=== "Windows (PowerShell)"

    ```powershell
    $HARBOR_VERSION = (helm search repo harbor/harbor --output json | ConvertFrom-Json)[0].version

    helm install harbor harbor/harbor `
      --namespace harbor `
      --version "$HARBOR_VERSION" `
      -f harbor-values.yaml `
      --wait --timeout 10m
    ```

## Acesso Temporário

=== "Linux / macOS"

    ```bash
    kubectl port-forward svc/harbor-portal -n harbor 8081:80 &
    # http://localhost:8081 | admin / $HARBOR_ADMIN_PASSWORD
    ```

=== "Windows (PowerShell)"

    ```powershell
    Start-Job { kubectl port-forward svc/harbor-portal -n harbor 8081:80 }
    # http://localhost:8081 | admin / $HARBOR_ADMIN_PASSWORD
    ```

## Criar Projeto e Robot Account

=== "Linux / macOS"

    ```bash
    HARBOR_TOKEN=$(echo -n "admin:${HARBOR_ADMIN_PASSWORD}" | base64)

    # Criar projeto
    curl -s -X POST http://localhost:8081/api/v2.0/projects \
      -H "Authorization: Basic ${HARBOR_TOKEN}" \
      -H "Content-Type: application/json" \
      -d '{"project_name": "gitops-lab", "public": false, "metadata": {"auto_scan": "true"}}'

    # Criar robot account
    curl -s -X POST http://localhost:8081/api/v2.0/robots \
      -H "Authorization: Basic ${HARBOR_TOKEN}" \
      -H "Content-Type: application/json" \
      -d '{
        "name": "eks-cluster-pull",
        "description": "Pull de imagens pelo cluster EKS",
        "duration": -1,
        "level": "system",
        "permissions": [{"kind": "project", "namespace": "gitops-lab",
          "access": [{"resource": "repository", "action": "pull"}]}]
      }' | jq '{name: .name, secret: .secret}'
    ```

=== "Windows (PowerShell)"

    ```powershell
    $HARBOR_TOKEN = [System.Convert]::ToBase64String(
      [System.Text.Encoding]::UTF8.GetBytes("admin:${HARBOR_ADMIN_PASSWORD}")
    )

    # Criar projeto
    Invoke-RestMethod -Method Post -Uri http://localhost:8081/api/v2.0/projects `
      -Headers @{ Authorization = "Basic $HARBOR_TOKEN"; "Content-Type" = "application/json" } `
      -Body '{"project_name": "gitops-lab", "public": false, "metadata": {"auto_scan": "true"}}'

    # Criar robot account
    $robot = Invoke-RestMethod -Method Post -Uri http://localhost:8081/api/v2.0/robots `
      -Headers @{ Authorization = "Basic $HARBOR_TOKEN"; "Content-Type" = "application/json" } `
      -Body '{
        "name": "eks-cluster-pull",
        "description": "Pull de imagens pelo cluster EKS",
        "duration": -1,
        "level": "system",
        "permissions": [{"kind": "project", "namespace": "gitops-lab",
          "access": [{"resource": "repository", "action": "pull"}]}]
      }'

    Write-Host "Robot name: $($robot.name)"
    Write-Host "Robot secret: $($robot.secret)"
    ```

!!! tip "Armazenar credenciais no Vault"
    O secret do robot account deve ser armazenado no Vault (`secret/harbor/robot-account`) e sincronizado via ESO.

## Salvar Variáveis

=== "Linux / macOS"

    ```bash
    echo "export HARBOR_ADMIN_PASSWORD=\"$HARBOR_ADMIN_PASSWORD\"" >> ~/eks-lab-vars.env
    echo "export HARBOR_ROLE_ARN=\"$HARBOR_ROLE_ARN\"" >> ~/eks-lab-vars.env
    ```

=== "Windows (PowerShell)"

    ```powershell
    "`$HARBOR_ADMIN_PASSWORD = `"$HARBOR_ADMIN_PASSWORD`"" | Add-Content ~\eks-lab-vars.ps1
    "`$HARBOR_ROLE_ARN       = `"$HARBOR_ROLE_ARN`""       | Add-Content ~\eks-lab-vars.ps1
    ```

## Checklist

- [ ] Service Accounts `harbor-core` e `harbor-registry` pré-criados no namespace `harbor`
- [ ] Role IAM do Harbor criada com acesso ao S3
- [ ] Pod Identity Associations criadas para `harbor-core` e `harbor-registry`
- [ ] Harbor instalado com backend S3
- [ ] Todos os pods em estado `Running`
- [ ] Projeto `gitops-lab` criado
- [ ] Robot account criado e secret armazenado no Vault

Próximo passo: [External Secrets Operator](07-external-secrets.md).
