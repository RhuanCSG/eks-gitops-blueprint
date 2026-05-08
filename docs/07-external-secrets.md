# 07. External Secrets Operator

O External Secrets Operator (ESO) sincroniza segredos do Vault para Kubernetes Secrets nativos.

## Instalação via Helm

### 1. Adicionar repositório

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
```

### 2. Instalar

=== "Linux / macOS"

    ```bash
    ESO_VERSION=$(helm search repo external-secrets/external-secrets --output json | jq -r '.[0].version')

    helm install external-secrets external-secrets/external-secrets \
      --namespace external-secrets \
      --version "$ESO_VERSION" \
      --set installCRDs=true \
      --wait
    ```

=== "Windows (PowerShell)"

    ```powershell
    $ESO_VERSION = (helm search repo external-secrets/external-secrets --output json | ConvertFrom-Json)[0].version

    helm install external-secrets external-secrets/external-secrets `
      --namespace external-secrets `
      --version "$ESO_VERSION" `
      --set installCRDs=true `
      --wait
    ```

## Configurar ClusterSecretStore

=== "Linux / macOS"

    ```bash
    kubectl apply -f - << 'EOF'
    apiVersion: external-secrets.io/v1beta1
    kind: ClusterSecretStore
    metadata:
      name: vault-backend
    spec:
      provider:
        vault:
          server: "http://vault.vault.svc.cluster.local:8200"
          path: "secret"
          version: "v2"
          auth:
            kubernetes:
              mountPath: "kubernetes"
              role: "external-secrets"
              serviceAccountRef:
                name: external-secrets
                namespace: external-secrets
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: external-secrets.io/v1beta1
    kind: ClusterSecretStore
    metadata:
      name: vault-backend
    spec:
      provider:
        vault:
          server: "http://vault.vault.svc.cluster.local:8200"
          path: "secret"
          version: "v2"
          auth:
            kubernetes:
              mountPath: "kubernetes"
              role: "external-secrets"
              serviceAccountRef:
                name: external-secrets
                namespace: external-secrets
    '@ | kubectl apply -f -
    ```

```bash
kubectl get clustersecretstore vault-backend
# STATUS deve ser "Valid"
```

## Criar ExternalSecrets

### Exemplo 1: Segredo genérico

=== "Linux / macOS"

    ```bash
    kubectl apply -f - << 'EOF'
    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
      name: app-example-secret
      namespace: default
    spec:
      refreshInterval: 1h
      secretStoreRef:
        name: vault-backend
        kind: ClusterSecretStore
      target:
        name: app-example-secret
        creationPolicy: Owner
      data:
        - secretKey: database-password
          remoteRef:
            key: secret/app/example
            property: database-password
        - secretKey: api-key
          remoteRef:
            key: secret/app/example
            property: api-key
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
      name: app-example-secret
      namespace: default
    spec:
      refreshInterval: 1h
      secretStoreRef:
        name: vault-backend
        kind: ClusterSecretStore
      target:
        name: app-example-secret
        creationPolicy: Owner
      data:
        - secretKey: database-password
          remoteRef:
            key: secret/app/example
            property: database-password
        - secretKey: api-key
          remoteRef:
            key: secret/app/example
            property: api-key
    '@ | kubectl apply -f -
    ```

### Exemplo 2: Pull secret do Harbor

=== "Linux / macOS"

    ```bash
    kubectl apply -f - << 'EOF'
    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
      name: harbor-pull-secret
      namespace: default
    spec:
      refreshInterval: 24h
      secretStoreRef:
        name: vault-backend
        kind: ClusterSecretStore
      target:
        name: harbor-pull-secret
        creationPolicy: Owner
        template:
          type: kubernetes.io/dockerconfigjson
          data:
            .dockerconfigjson: |
              {
                "auths": {
                  "{{ .server }}": {
                    "username": "{{ .username }}",
                    "password": "{{ .password }}",
                    "auth": "{{ printf "%s:%s" .username .password | b64enc }}"
                  }
                }
              }
      data:
        - secretKey: username
          remoteRef:
            key: secret/harbor/robot-account
            property: username
        - secretKey: password
          remoteRef:
            key: secret/harbor/robot-account
            property: password
        - secretKey: server
          remoteRef:
            key: secret/harbor/robot-account
            property: server
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
      name: harbor-pull-secret
      namespace: default
    spec:
      refreshInterval: 24h
      secretStoreRef:
        name: vault-backend
        kind: ClusterSecretStore
      target:
        name: harbor-pull-secret
        creationPolicy: Owner
        template:
          type: kubernetes.io/dockerconfigjson
          data:
            .dockerconfigjson: |
              {
                "auths": {
                  "{{ .server }}": {
                    "username": "{{ .username }}",
                    "password": "{{ .password }}",
                    "auth": "{{ printf "%s:%s" .username .password | b64enc }}"
                  }
                }
              }
      data:
        - secretKey: username
          remoteRef:
            key: secret/harbor/robot-account
            property: username
        - secretKey: password
          remoteRef:
            key: secret/harbor/robot-account
            property: password
        - secretKey: server
          remoteRef:
            key: secret/harbor/robot-account
            property: server
    '@ | kubectl apply -f -
    ```

## Forçar Sincronização Manual

=== "Linux / macOS"

    ```bash
    kubectl annotate externalsecret app-example-secret \
      force-sync=$(date +%s) --overwrite
    ```

=== "Windows (PowerShell)"

    ```powershell
    $timestamp = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
    kubectl annotate externalsecret app-example-secret `
      "force-sync=$timestamp" --overwrite
    ```

## Checklist

- [ ] ESO instalado com CRDs
- [ ] `ClusterSecretStore vault-backend` com status `Valid`
- [ ] `ExternalSecret` de exemplo criado e sincronizado
- [ ] Kubernetes Secret criado automaticamente pelo ESO
- [ ] Pull secret do Harbor criado via ExternalSecret

Próximo passo: [cert-manager](08-cert-manager.md).
