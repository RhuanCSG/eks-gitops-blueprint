# 07. External Secrets Operator

O External Secrets Operator (ESO) sincroniza segredos do HashiCorp Vault para Kubernetes Secrets nativos, eliminando a necessidade de gerenciar secrets manualmente no cluster.

## Arquitetura

```
Vault (KV v2)
  └── secret/harbor/admin
        └── HARBOR_ADMIN_PASSWORD
              ↓ (ESO lê via Kubernetes auth)
        ClusterSecretStore: vault-backend
              ↓ (ESO sincroniza)
        ExternalSecret: harbor-admin-secret (namespace: harbor)
              ↓ (ESO cria)
        Secret: harbor-admin-secret (namespace: harbor)
              ↓ (Harbor lê)
        Harbor UI funcionando
```

## Pré-requisitos no Vault

Os itens abaixo são configurados na [Etapa 05 — Vault](05-vault.md). Confirme que existem antes de continuar:

```bash
# Listar métodos de auth (deve mostrar "kubernetes/")
kubectl exec -n vault vault-0 -- \
  sh -c 'VAULT_TOKEN=$VAULT_TOKEN VAULT_ADDR=http://127.0.0.1:8200 vault auth list'

# Confirmar segredo do Harbor
kubectl exec -n vault vault-0 -- \
  sh -c 'VAULT_TOKEN=$VAULT_TOKEN VAULT_ADDR=http://127.0.0.1:8200 vault kv get secret/harbor/admin'

# Confirmar policy e role do ESO
kubectl exec -n vault vault-0 -- \
  sh -c 'VAULT_TOKEN=$VAULT_TOKEN VAULT_ADDR=http://127.0.0.1:8200 vault policy read external-secrets'

kubectl exec -n vault vault-0 -- \
  sh -c 'VAULT_TOKEN=$VAULT_TOKEN VAULT_ADDR=http://127.0.0.1:8200 vault read auth/kubernetes/role/external-secrets'
```

Os valores esperados são:

| Item | Valor esperado |
|---|---|
| Auth method | `kubernetes/` habilitado |
| Secret path | `secret/data/harbor/admin` com chave `HARBOR_ADMIN_PASSWORD` |
| Policy | `external-secrets` — leitura em `secret/data/*` |
| Role | `external-secrets` — bound ao SA `external-secrets` no namespace `external-secrets` |

## Instalação via ArgoCD

O ESO é instalado automaticamente pelo ArgoCD na Wave 2. Confirme que os pods estão Running antes de prosseguir:

```bash
kubectl get pods -n external-secrets
```

Todos os pods devem estar `1/1 Running`:

- `external-secrets-*` (2 réplicas)
- `external-secrets-cert-controller-*`
- `external-secrets-webhook-*` (2 réplicas)

!!! warning "Webhook deve estar Ready"
    O `ExternalSecret` só pode ser criado após o webhook de validação do ESO estar `1/1 Ready`.
    Se os pods do webhook estiverem `0/1`, aguarde e tente novamente.

## Habilitar recursão no cluster-manifests

Por padrão, o ArgoCD **não recorre em subdiretórios**. O `ClusterSecretStore` e os `ExternalSecrets` ficam em subdiretórios de `manifests/`, então é necessário adicionar `directory.recurse: true` na Application `cluster-manifests`:

```yaml title="apps/cluster-manifests.yaml"
spec:
  source:
    repoURL: https://github.com/<GITHUB_USERNAME>/infra-gitops-delivery
    targetRevision: main
    path: manifests
    directory:
      recurse: true   # (1)!
```

1. Sem esta flag, `manifests/network-policies/`, `manifests/external-secrets/` e outros subdiretórios são ignorados.

!!! note
    Esta flag também é necessária para que as NetworkPolicies em `manifests/network-policies/` sejam aplicadas.

## ClusterSecretStore

Crie o arquivo `manifests/cluster-secret-store.yaml` no repositório de infra:

```yaml title="manifests/cluster-secret-store.yaml"
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
            name: "external-secrets"
            namespace: "external-secrets"
```

Após o push, verifique:

```bash
kubectl get clustersecretstore vault-backend
# NAME            AGE   STATUS   CAPABILITIES   READY
# vault-backend   30s   Valid    ReadWrite      True
```

O status deve ser `Valid` e `READY: True`. Se aparecer `InvalidProvider`, verifique se o Vault está acessível e se o role existe.

## ExternalSecret: Harbor admin password

Crie o arquivo `manifests/external-secrets/harbor-admin-secret.yaml`:

```yaml title="manifests/external-secrets/harbor-admin-secret.yaml"
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: harbor-admin-secret
  namespace: harbor
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: harbor-admin-secret
    creationPolicy: Owner
  data:
    - secretKey: HARBOR_ADMIN_PASSWORD
      remoteRef:
        key: harbor/admin        # (1)!
        property: HARBOR_ADMIN_PASSWORD
```

1. Para KV v2, o caminho não inclui `secret/data/` — o ESO adiciona o prefixo automaticamente com base em `path: "secret"` e `version: "v2"` do `ClusterSecretStore`.

Após o ArgoCD sincronizar, verifique:

```bash
kubectl get externalsecret harbor-admin-secret -n harbor
# NAME                  STORE          REFRESH INTERVAL   STATUS
# harbor-admin-secret   vault-backend  1h                 SecretSynced

kubectl get secret harbor-admin-secret -n harbor
# NAME                  TYPE     DATA   AGE
# harbor-admin-secret   Opaque   1      30s
```

## Verificar Harbor

Com o `harbor-admin-secret` criado, o pod `harbor-core` deve iniciar:

```bash
kubectl get pods -n harbor
```

Todos os pods de Harbor devem ficar `Running`. Se `harbor-core` ainda estiver em `CreateContainerConfigError`, verifique se o Secret foi criado no namespace correto:

```bash
kubectl describe pod -n harbor -l component=core | grep -A5 "Events"
```

## NetworkPolicies: correções necessárias

!!! warning "Problema sistêmico — DNS bloqueado em todas as namespaces"
    Ao habilitar `directory.recurse: true`, as NetworkPolicies de todos os namespaces passam a ser aplicadas. Sem uma regra de egresso para DNS (UDP/TCP 53 → `kube-system`), nenhum pod consegue resolver nomes dentro do cluster.

    Sintomas típicos:

    | Namespace | Erro |
    |---|---|
    | `argocd` | `dial udp 172.20.0.10:53: i/o timeout` ao sincronizar |
    | `external-secrets` | `context deadline exceeded` no webhook de validação |
    | `vault` | Falha silenciosa no auto-unseal KMS após restart |
    | `harbor` | Falha ao conectar ao banco de dados interno |
    | `cert-manager` | Falha ao resolver endpoints do Let's Encrypt |

**Solução:** adicionar a regra abaixo em todas as policies de egresso:

```yaml
- ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
```

Todos os arquivos em `manifests/network-policies/` do repositório já incluem esta correção.

### ESO: porta do webhook e comunicação interna

O ESO possui dois problemas adicionais que não afetam outros namespaces:

**1. Porta incorreta no `allow-webhook-ingress`** — o webhook do ESO ouve na porta **10250** (não 9443). A policy deve usar:

```yaml
ingress:
  - ports:
      - protocol: TCP
        port: 10250
```

**2. Comunicação interna entre componentes** — o ESO controller, webhook e cert-controller precisam se comunicar. Adicione:

```yaml title="manifests/network-policies/external-secrets-netpol.yaml (trecho)"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-eso-internal
  namespace: external-secrets
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: external-secrets
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: external-secrets
```

## Forçar sincronização manual de um ExternalSecret

=== "Linux / macOS"

    ```bash
    kubectl annotate externalsecret harbor-admin-secret -n harbor \
      force-sync=$(date +%s) --overwrite
    ```

=== "Windows (PowerShell)"

    ```powershell
    $timestamp = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
    kubectl annotate externalsecret harbor-admin-secret -n harbor `
      "force-sync=$timestamp" --overwrite
    ```

## Checklist

- [ ] Vault com Kubernetes auth habilitado e role `external-secrets` configurado
- [ ] ESO instalado e todos os pods `1/1 Running` (incluindo webhook)
- [ ] `directory.recurse: true` adicionado ao `cluster-manifests` Application
- [ ] `ClusterSecretStore vault-backend` com status `Valid` e `READY: True`
- [ ] `ExternalSecret harbor-admin-secret` com status `SecretSynced`
- [ ] `Secret harbor-admin-secret` criado no namespace `harbor`
- [ ] `harbor-core` pod em `Running` sem erros de secret

Próximo passo: [cert-manager](08-cert-manager.md).
