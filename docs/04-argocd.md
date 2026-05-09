# 04. ArgoCD

Instalamos o ArgoCD via Helm e configuramos o padrão **App of Apps**: uma Application raiz gerencia todas as outras Applications do cluster de forma declarativa.

## Conceito: App of Apps

```
argocd-root-app (Application)
├── vault-app (Application)
├── harbor-app (Application)
├── external-secrets-app (Application)
├── cert-manager-app (Application)
└── aws-lbc-app (Application)
```

## Instalação

### 1. Adicionar repositório Helm

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### 2. Instalar ArgoCD

=== "Linux / macOS"

    ```bash
    ARGOCD_VERSION=$(helm search repo argo/argo-cd --output json | jq -r '.[0].version')

    helm install argocd argo/argo-cd \
      --namespace argocd \
      --version "$ARGOCD_VERSION" \
      --set configs.params."server\.insecure"=true \
      --set server.service.type=ClusterIP \
      --wait
    ```

=== "Windows (PowerShell)"

    ```powershell
    $ARGOCD_VERSION = (helm search repo argo/argo-cd --output json | ConvertFrom-Json)[0].version

    helm install argocd argo/argo-cd `
      --namespace argocd `
      --version "$ARGOCD_VERSION" `
      --set "configs.params.server\.insecure=true" `
      --set server.service.type=ClusterIP `
      --wait
    ```

### 3. Verificar instalação

```bash
kubectl get pods -n argocd
```

Saída esperada:

```
NAME                                               READY   STATUS    RESTARTS
argocd-application-controller-0                    1/1     Running   0
argocd-applicationset-controller-xxx               1/1     Running   0
argocd-dex-server-xxx                              1/1     Running   0
argocd-notifications-controller-xxx                1/1     Running   0
argocd-redis-xxx                                   1/1     Running   0
argocd-repo-server-xxx                             1/1     Running   0
argocd-server-xxx                                  1/1     Running   0
```

### 4. Obter senha inicial do admin

=== "Linux / macOS"

    ```bash
    ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
      -o jsonpath="{.data.password}" | base64 -d)

    echo "ArgoCD admin password: $ARGOCD_PASSWORD"
    ```

=== "Windows (PowerShell)"

    ```powershell
    $ARGOCD_PASSWORD_B64 = kubectl -n argocd get secret argocd-initial-admin-secret `
      -o jsonpath="{.data.password}"

    $ARGOCD_PASSWORD = [System.Text.Encoding]::UTF8.GetString(
      [System.Convert]::FromBase64String($ARGOCD_PASSWORD_B64)
    )

    Write-Host "ArgoCD admin password: $ARGOCD_PASSWORD"
    ```

!!! warning "Troque a senha após o primeiro acesso"
    A senha inicial é gerada automaticamente. Troque-a após o primeiro login.

## Acesso Temporário via Port-forward

=== "Linux / macOS"

    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:80 &
    # Acessar em: http://localhost:8080
    ```

=== "Windows (PowerShell)"

    ```powershell
    Start-Job { kubectl port-forward svc/argocd-server -n argocd 8080:80 }
    # Acessar em: http://localhost:8080
    ```

## Configurar Repositório Privado

!!! info "Scope do GitHub PAT"
    O token precisa de permissão de leitura no repositório privado:

    - **Classic PAT**: scope `repo`
    - **Fine-grained PAT**: repositório `infra-gitops-delivery` → `Contents: Read-only`

=== "Linux / macOS"

    ```bash
    kubectl apply -n argocd -f - << EOF
    apiVersion: v1
    kind: Secret
    metadata:
      name: infra-gitops-delivery-repo
      namespace: argocd
      labels:
        argocd.argoproj.io/secret-type: repository
    type: Opaque
    stringData:
      type: git
      url: https://github.com/RhuanCSG/infra-gitops-delivery
      username: RhuanCSG
      password: <GITHUB_PAT>
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: v1
    kind: Secret
    metadata:
      name: infra-gitops-delivery-repo
      namespace: argocd
      labels:
        argocd.argoproj.io/secret-type: repository
    type: Opaque
    stringData:
      type: git
      url: https://github.com/RhuanCSG/infra-gitops-delivery
      username: RhuanCSG
      password: <GITHUB_PAT>
    '@ | kubectl apply -n argocd -f -
    ```

!!! warning "Segurança"
    Nunca armazene o PAT em YAML versionado. Após o bootstrap inicial, o PAT é gerenciado via Vault + External Secrets Operator.

## App of Apps — Bootstrap

!!! warning "Pré-requisito: IAM antes do root-app"
    No fluxo GitOps, o ArgoCD sobe Vault (Wave 3) e Harbor (Wave 4) automaticamente. Os pods precisam das Pod Identity Associations para acessar KMS e S3 **desde o primeiro start**. Crie todos os recursos IAM antes de aplicar o root-app:

    - **Vault**: role IAM + Pod Identity Association → detalhes na [Etapa 05](05-vault.md)
    - **Harbor**: Service Accounts + role IAM + Pod Identity Associations → detalhes na [Etapa 06](06-harbor.md)
    - **AWS LBC**: role IAM + Pod Identity Association → detalhes na [Etapa 09](09-aws-load-balancer.md)

### Criar a Application raiz

=== "Linux / macOS"

    ```bash
    kubectl apply -n argocd -f - << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: root-app
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      source:
        repoURL: https://github.com/RhuanCSG/infra-gitops-delivery
        targetRevision: main
        path: apps
      destination:
        server: https://kubernetes.default.svc
        namespace: argocd
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: root-app
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      source:
        repoURL: https://github.com/RhuanCSG/infra-gitops-delivery
        targetRevision: main
        path: apps
      destination:
        server: https://kubernetes.default.svc
        namespace: argocd
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    '@ | kubectl apply -n argocd -f -
    ```

## Verificar Sincronização

```bash
kubectl get applications -n argocd
kubectl describe application root-app -n argocd
```

## Checklist

- [ ] ArgoCD instalado no namespace `argocd`
- [ ] Todos os pods em estado `Running`
- [ ] Senha do admin obtida
- [ ] Repositório privado `infra-gitops-delivery` registrado
- [ ] Application `root-app` criada e sincronizando

Próximo passo: [HashiCorp Vault](05-vault.md).
