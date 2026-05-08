# 08. cert-manager

O cert-manager automatiza a emissão e renovação de certificados TLS via Let's Encrypt.

## Instalação via Helm

### 1. Adicionar repositório

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

### 2. Instalar

=== "Linux / macOS"

    ```bash
    CM_VERSION=$(helm search repo jetstack/cert-manager --output json | jq -r '.[0].version')

    helm install cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --version "$CM_VERSION" \
      --set crds.enabled=true \
      --wait
    ```

=== "Windows (PowerShell)"

    ```powershell
    $CM_VERSION = (helm search repo jetstack/cert-manager --output json | ConvertFrom-Json)[0].version

    helm install cert-manager jetstack/cert-manager `
      --namespace cert-manager `
      --version "$CM_VERSION" `
      --set crds.enabled=true `
      --wait
    ```

## Configurar ClusterIssuers

=== "Linux / macOS"

    ```bash
    kubectl apply -f - << 'EOF'
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-staging
    spec:
      acme:
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        email: <SEU_EMAIL>
        privateKeySecretRef:
          name: letsencrypt-staging-key
        solvers:
          - http01:
              ingress:
                ingressClassName: alb
    ---
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: <SEU_EMAIL>
        privateKeySecretRef:
          name: letsencrypt-prod-key
        solvers:
          - http01:
              ingress:
                ingressClassName: alb
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-staging
    spec:
      acme:
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        email: <SEU_EMAIL>
        privateKeySecretRef:
          name: letsencrypt-staging-key
        solvers:
          - http01:
              ingress:
                ingressClassName: alb
    ---
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: <SEU_EMAIL>
        privateKeySecretRef:
          name: letsencrypt-prod-key
        solvers:
          - http01:
              ingress:
                ingressClassName: alb
    '@ | kubectl apply -f -
    ```

Substitua `<SEU_EMAIL>` por um endereço válido.

```bash
kubectl get clusterissuer
# letsencrypt-prod    True
# letsencrypt-staging True
```

!!! tip "Staging primeiro, produção depois"
    Sempre teste com `letsencrypt-staging` antes de usar `letsencrypt-prod`. O staging não tem rate limits e confirma que o processo funciona.

## Checklist

- [ ] cert-manager instalado com CRDs
- [ ] Todos os pods em estado `Running`
- [ ] `ClusterIssuer letsencrypt-staging` com status `True`
- [ ] `ClusterIssuer letsencrypt-prod` com status `True`

Próximo passo: [AWS Load Balancer Controller](09-aws-load-balancer.md).
