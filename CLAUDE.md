# EKS GitOps Blueprint — Contexto do Projeto

## Visão Geral

Laboratório de infraestrutura GitOps com AWS EKS, HashiCorp Vault, Harbor e ArgoCD.
Objetivo: servir de modelo para implementações em produção.
Todos os recursos AWS são destruídos ao final do laboratório.

**GitHub:** `RhuanCSG` — https://github.com/RhuanCSG

---

## Repositórios

| Repositório | Visibilidade | URL | Finalidade |
|---|---|---|---|
| `eks-gitops-blueprint` | Público | https://github.com/RhuanCSG/eks-gitops-blueprint | Documentação (GitHub Pages + MkDocs) |
| `infra-gitops-delivery-blueprint` | Público | https://github.com/RhuanCSG/infra-gitops-delivery-blueprint | Manifestos com placeholders — para fork |
| `infra-gitops-delivery` | Privado | https://github.com/RhuanCSG/infra-gitops-delivery | Manifestos reais — observado pelo ArgoCD |
| App repo | A definir | — | Criado na Fase 5 |

**Caminhos locais:**

| Repositório | Caminho |
|---|---|
| `eks-gitops-blueprint` | `C:\projetos\claude\eks-gitops-blueprint` |
| `infra-gitops-delivery-blueprint` | `C:\projetos\claude\infra-gitops-delivery-blueprint` |
| `infra-gitops-delivery` | `C:\projetos\claude\infra-gitops-delivery` |

**GitHub Pages:** https://rhuancsg.github.io/eks-gitops-blueprint/
Build via GitHub Actions (`.github/workflows/docs.yml`) — dispara em push para `main`.

---

## Stack de Tecnologias

| Camada | Tecnologia | Observação |
|---|---|---|
| Cloud | AWS us-east-1 | Menor custo, maior disponibilidade de Spot |
| Kubernetes | EKS 1.32+ | Control plane gerenciado |
| Nodes | t3.medium · Spot · 3 AZs | Custo mínimo com HA |
| SO | Amazon Linux 2023 | Nunca Amazon Linux 2 |
| Storage | EBS gp3 criptografado | Nunca gp2 |
| IAM | EKS Pod Identity | Nunca IRSA |
| Network Policy | VPC CNI nativo | Nunca Calico |
| GitOps | ArgoCD · App of Apps | Sync waves para ordenação |
| Secrets | Vault HA + Raft + KMS | Auto-unseal obrigatório com Spot |
| Secrets sync | External Secrets Operator | ClusterSecretStore → Kubernetes Secrets |
| Registry | Harbor + S3 | Persistência independente de pods |
| Ingress | AWS LBC + ALB | Nativo AWS |
| TLS | cert-manager + Let's Encrypt | Gratuito, automático |
| DNS | Endpoints do ALB | Sem domínio customizado |
| Docs | MkDocs Material | requirements.txt: `mkdocs-material==9.6.14` |

---

## Versões dos Helm Charts (fixadas)

| Ferramenta | Chart Version | Helm Repo |
|---|---|---|
| cert-manager | `v1.15.3` | https://charts.jetstack.io |
| AWS Load Balancer Controller | `1.8.3` | https://aws.github.io/eks-charts |
| External Secrets Operator | `0.10.4` | https://charts.external-secrets.io |
| HashiCorp Vault | `0.28.1` | https://helm.releases.hashicorp.com |
| Harbor | `1.15.1` | https://helm.goharbor.io |

> ArgoCD **não** é gerenciado por ele mesmo — instalado manualmente via Helm antes do bootstrap.

---

## Design da Rede VPC

```
VPC: 10.0.0.0/16

Subnets Públicas (ALB):
  10.0.1.0/24  — us-east-1a
  10.0.2.0/24  — us-east-1b
  10.0.3.0/24  — us-east-1c

Subnets Privadas (EKS Nodes):
  10.0.11.0/24 — us-east-1a
  10.0.12.0/24 — us-east-1b
  10.0.13.0/24 — us-east-1c
```

- NAT Gateway único em us-east-1a (custo de lab — em produção, um por AZ)
- Tags obrigatórias para o AWS LBC:
  - Subnets públicas: `kubernetes.io/role/elb=1`
  - Subnets privadas: `kubernetes.io/role/internal-elb=1`
  - Todas as subnets: `kubernetes.io/cluster/<CLUSTER_NAME>=shared`

---

## Estrutura dos Repositórios

### `eks-gitops-blueprint` (documentação)

```
eks-gitops-blueprint/
├── CLAUDE.md
├── README.md
├── mkdocs.yml
├── requirements.txt                    # mkdocs-material==9.6.14
├── LICENSE
├── .gitignore
├── docs/
│   ├── index.md
│   ├── 01-prerequisites.md
│   ├── 02-vpc-networking.md
│   ├── 03-eks-cluster.md
│   ├── 04-argocd.md
│   ├── 05-vault.md
│   ├── 06-harbor.md
│   ├── 07-external-secrets.md
│   ├── 08-cert-manager.md
│   └── 09-aws-load-balancer.md
└── .github/
    └── workflows/
        └── docs.yml
```

### `infra-gitops-delivery-blueprint` e `infra-gitops-delivery` (manifestos)

Estrutura idêntica. O blueprint tem `<PLACEHOLDERS>`; o privado os tem substituídos.

```
infra-gitops-delivery[-blueprint]/
├── bootstrap/
│   └── root-app.yaml                   # Application raiz — aplicar uma única vez
├── apps/                               # ArgoCD Applications (gerenciadas pelo root-app)
│   ├── cluster-manifests.yaml          # sync-wave: 0
│   ├── cert-manager.yaml               # sync-wave: 1
│   ├── aws-load-balancer-controller.yaml  # sync-wave: 1
│   ├── external-secrets.yaml           # sync-wave: 2
│   ├── vault.yaml                      # sync-wave: 3
│   └── harbor.yaml                     # sync-wave: 4
├── charts/                             # Helm values (sem Chart.yaml — multiple sources no ArgoCD)
│   ├── cert-manager/values.yaml
│   ├── aws-load-balancer-controller/values.yaml
│   ├── external-secrets/values.yaml
│   ├── vault/values.yaml
│   └── harbor/values.yaml
├── manifests/
│   ├── namespaces.yaml
│   ├── storageclass.yaml               # gp3 como default, gp2 removido como default
│   └── network-policies/
│       ├── argocd-netpol.yaml          # default-deny + allow ALB + allow internal
│       ├── cert-manager-netpol.yaml
│       ├── external-secrets-netpol.yaml
│       ├── harbor-netpol.yaml
│       └── vault-netpol.yaml
├── scripts/
│   ├── config.env.example              # Template de valores (Linux)
│   ├── config.ps1.example              # Template de valores (Windows)
│   ├── replace-placeholders.sh         # Substitui placeholders (Linux)
│   └── replace-placeholders.ps1        # Substitui placeholders (Windows)
├── .gitignore                          # ignora scripts/config.env e scripts/config.ps1
├── LICENSE
└── README.md
```

---

## Padrão de Substituição de Placeholders

O repositório público tem `<PLACEHOLDERS>`. Para usar em ambiente real:

1. Copiar `scripts/config.env.example` → `scripts/config.env` (Linux) ou `config.ps1` (Windows)
2. Preencher com valores reais
3. Executar `bash scripts/replace-placeholders.sh` ou `.\scripts\replace-placeholders.ps1`
4. Revisar com `git diff` e fazer push para o repo privado

O `scripts/config.env` está no `.gitignore` — **nunca versionar**.

---

## Placeholders

| Placeholder | Descrição | Valor neste projeto |
|---|---|---|
| `<AWS_ACCOUNT_ID>` | ID da conta AWS (12 dígitos) | a preencher |
| `<CLUSTER_NAME>` | Nome do cluster EKS | a preencher (sugestão: `eks-gitops-lab`) |
| `<AWS_REGION>` | Região AWS | `us-east-1` |
| `<VPC_ID>` | ID da VPC criada | a preencher |
| `<KMS_KEY_ARN>` | ARN da chave KMS para Vault auto-unseal | a preencher |
| `<HARBOR_S3_BUCKET>` | Nome do bucket S3 do Harbor | a preencher (sugestão: `harbor-registry-<AWS_ACCOUNT_ID>`) |
| `<GITHUB_USERNAME>` | Username do GitHub | `RhuanCSG` |
| `<HARBOR_ADMIN_PASSWORD>` | Senha admin do Harbor | gerar com `openssl rand -base64 24` |
| `<HARBOR_ALB_HOSTNAME>` | Hostname do ALB do Harbor | obtido após etapa 09 |
| `<ARGOCD_REPO_URL>` | URL do repo privado de infra | `https://github.com/RhuanCSG/infra-gitops-delivery` |
| `<ARGOCD_REPO_SECRET>` | Nome do Secret no ArgoCD com credenciais do repo | `infra-gitops-delivery-repo` |

---

## ArgoCD: Padrão App of Apps e Sync Waves

O `root-app` observa o diretório `apps/` do repositório privado. Cada arquivo cria uma Application. A ordem de sincronização é controlada por `argocd.argoproj.io/sync-wave`:

```
Wave 0 — cluster-manifests   → namespaces, StorageClass gp3, NetworkPolicies
Wave 1 — cert-manager        → TLS (dependência de todos os Ingresses)
Wave 1 — aws-lbc             → ALB Controller (dependência dos Ingresses)
Wave 2 — external-secrets    → ESO (dependência de quem lê segredos do Vault)
Wave 3 — vault               → Vault HA + Raft (requer ESO para funcionar)
Wave 4 — harbor              → Registry (requer ESO, LBC e cert-manager)
```

As Applications usam **multiple sources** do ArgoCD (2.6+):
- Fonte 1: Helm chart upstream (por URL + versão)
- Fonte 2: Git repo com `values.yaml` customizados

Isso elimina a necessidade de umbrella charts ou `Chart.yaml` no repositório.

---

## Fluxo GitOps Completo

```
Developer → push → infra-gitops-delivery (privado)
                         ↓
                   ArgoCD detecta (30s ou webhook)
                         ↓
              reconcilia cluster em sync waves
                         ↓
         wave 0 → wave 1 → wave 2 → wave 3 → wave 4
```

---

## Sequência de Bootstrap do Cluster

Após criar todos os recursos AWS (Etapas 01-09 da documentação):

1. Preencher `scripts/config.env` (ou `config.ps1`) com os valores reais
2. Executar o script de substituição de placeholders
3. Fazer push para `infra-gitops-delivery` (privado)
4. Registrar credenciais do repo privado no ArgoCD
5. Aplicar o root-app: `kubectl apply -f bootstrap/root-app.yaml`
6. O ArgoCD sincroniza tudo automaticamente nas waves corretas
7. Inicializar o Vault manualmente (vault operator init)
8. Configurar Harbor (projeto, robot account → Vault → ESO)

---

## IAM: EKS Pod Identity Associations necessárias

| ServiceAccount | Namespace | IAM Role | Para quê |
|---|---|---|---|
| `vault` | `vault` | `<CLUSTER_NAME>-vault` | Acesso ao KMS para auto-unseal |
| `harbor-core` | `harbor` | `<CLUSTER_NAME>-harbor` | Acesso ao S3 para imagens |
| `harbor-registry` | `harbor` | `<CLUSTER_NAME>-harbor` | Acesso ao S3 para imagens |
| `aws-load-balancer-controller` | `kube-system` | `<CLUSTER_NAME>-aws-lbc` | Gerenciar ALBs |

---

## Convenções de Documentação

- Todas as páginas de docs têm comandos em **dois formatos** via MkDocs content tabs:
  - `=== "Linux / macOS"` com bloco `bash`
  - `=== "Windows (PowerShell)"` com bloco `powershell`
- Comandos idênticos em ambas as plataformas (`aws cli`, `kubectl`, `helm`, `eksctl`) ficam em bloco único sem abas
- `requirements.txt` usa versão fixada (`mkdocs-material==9.6.14`) — não usar `latest`
- Nunca colocar informações sensíveis em repositórios públicos

---

## Princípios e Boas Práticas

- **Nunca** usar recursos deprecados (gp2, Amazon Linux 2, versões EOL)
- **Nunca** armazenar informações sensíveis em repositórios públicos
- **Nunca** usar IRSA — EKS Pod Identity é o padrão atual
- **Nunca** usar Calico — VPC CNI nativo para Network Policies
- Todo estado do cluster declarado em Git — zero `kubectl apply` manual em produção
- Vault **sempre** em modo HA com Raft e auto-unseal KMS (Spot Instances reiniciam sem aviso)
- Harbor com S3 como backend — imagens não dependem de PVC

---

## Fases do Projeto

| Fase | Status | Entregável |
|---|---|---|
| 1 | Concluída | CLAUDE.md + 9 páginas de documentação (`eks-gitops-blueprint`) |
| 2 | Concluída | Manifestos YAML locais (apps, charts, manifests, scripts) |
| 3 | Concluída | 3 repos criados no GitHub + GitHub Pages habilitado |
| 4 | Em andamento | Executar o laboratório AWS — Etapas 01-05 concluídas, pendente: ESO (07), AWS LBC (09) |
| 5 | Pendente | Repositório de aplicação + deploy end-to-end no cluster |

## Recursos AWS Criados (Fase 4)

| Recurso | ID / ARN |
|---|---|
| VPC | `vpc-0fb73b7357e7d4d53` |
| Cluster EKS | `eks-gitops-lab` — us-east-1 — v1.32 |
| KMS Key (Vault) | `arn:aws:kms:us-east-1:237474125667:key/24126b44-4889-4baa-89e8-bdca0f2e95dc` |
| S3 Harbor | `harbor-registry-237474125667` |
| IAM Role Vault | `eks-gitops-lab-vault` |
| IAM Role Harbor | `eks-gitops-lab-harbor` |
| IAM Role EBS CSI | `eks-gitops-lab-ebs-csi` |
