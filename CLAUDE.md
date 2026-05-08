# EKS GitOps Blueprint — Contexto do Projeto

## Visão Geral

Laboratório de infraestrutura GitOps com AWS EKS, HashiCorp Vault, Harbor e ArgoCD.
Objetivo: servir de modelo para implementações em produção.
Todos os recursos AWS são destruídos ao final do laboratório.

## Repositórios

| Repositório | Visibilidade | Finalidade |
|---|---|---|
| `eks-gitops-blueprint` | Público | Documentação passo a passo (GitHub Pages + MkDocs Material) |
| `infra-gitops-delivery-blueprint` | Público | Manifestos YAML com placeholders para fork |
| `infra-gitops-delivery` | Privado | Manifestos reais — observado pelo ArgoCD |

> O repositório de aplicação será criado na fase 3 (após o cluster estar operacional).

## Stack de Tecnologias

- **Cloud**: AWS us-east-1
- **Kubernetes**: EKS (versão estável mais recente, mínimo 1.32)
- **Nodes**: Managed Node Groups, t3.medium, Spot Instances, 3 nodes em 3 AZs
- **Sistema Operacional**: Amazon Linux 2023
- **Storage**: EBS gp3 (nunca gp2 ou deprecated)
- **IAM**: EKS Pod Identity (nunca usar IRSA em implementações novas)
- **Rede**: VPC CNI com Network Policies nativas (nunca usar Calico)
- **GitOps**: ArgoCD com padrão App of Apps
- **Secrets**: HashiCorp Vault (HA + Raft) + External Secrets Operator
- **Vault Unseal**: Auto-unseal via AWS KMS (obrigatório com Spot Instances)
- **Registry**: Harbor com backend S3
- **Ingress**: AWS Load Balancer Controller + ALB
- **TLS**: cert-manager + Let's Encrypt
- **DNS**: Endpoints gerados pelo ALB (sem domínio customizado)
- **Docs**: MkDocs Material + GitHub Pages

## Princípios e Boas Práticas

- **Nunca** usar recursos deprecados (gp2, Amazon Linux 2, versões EOL, etc.)
- **Nunca** armazenar informações sensíveis em repositórios públicos
- Usar placeholders explícitos (`<AWS_ACCOUNT_ID>`, `<CLUSTER_NAME>`, etc.) nos manifestos públicos
- Todo estado do cluster deve ser declarado em Git — nenhum `kubectl apply` manual em produção
- EKS Pod Identity é o padrão de autenticação IAM (não IRSA)
- VPC CNI Network Policy para isolamento de rede entre namespaces
- Vault em modo HA com Raft e auto-unseal KMS — obrigatório mesmo em lab
- Harbor com S3 para persistência de imagens independente dos pods
- Toda documentação deve usar `<PLACEHOLDER>` para valores específicos de ambiente

## Estrutura de Diretórios

### `eks-gitops-blueprint` (este repositório — documentação)

```
eks-gitops-blueprint/
├── CLAUDE.md
├── README.md
├── mkdocs.yml
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

### `infra-gitops-delivery-blueprint` (manifestos públicos)

```
infra-gitops-delivery-blueprint/
├── bootstrap/          # Configuração inicial do ArgoCD e App of Apps raiz
├── apps/               # Applications do ArgoCD (uma por ferramenta)
├── charts/             # Helm values customizados por ferramenta
│   ├── vault/
│   ├── harbor/
│   ├── argocd/
│   ├── external-secrets/
│   ├── aws-load-balancer-controller/
│   └── cert-manager/
├── manifests/          # Recursos Kubernetes não-Helm (namespaces, RBAC, NetworkPolicies)
├── scripts/
│   └── replace-placeholders.sh
└── README.md
```

## Fluxo GitOps

```
Developer → push → infra-gitops-delivery (privado)
                          ↓
                    ArgoCD (no cluster)
                          ↓
              reconcilia estado do cluster
```

## Placeholders usados nos manifestos públicos

| Placeholder | Descrição |
|---|---|
| `<AWS_ACCOUNT_ID>` | ID da conta AWS (12 dígitos) |
| `<CLUSTER_NAME>` | Nome do cluster EKS |
| `<AWS_REGION>` | Região AWS (us-east-1) |
| `<VPC_ID>` | ID da VPC criada |
| `<KMS_KEY_ARN>` | ARN da chave KMS para Vault auto-unseal |
| `<HARBOR_S3_BUCKET>` | Nome do bucket S3 do Harbor |
| `<GITHUB_USERNAME>` | Username do GitHub (RhuanCSG neste projeto) |
| `<ARGOCD_REPO_URL>` | URL HTTPS do repositório privado de infra |
| `<ARGOCD_REPO_SECRET>` | Nome do Secret com credenciais do repo no ArgoCD |

## Decisões Arquiteturais (resumo)

| Categoria | Decisão | Motivo |
|---|---|---|
| Ambientes | Cluster único | Custo mínimo para lab |
| Região | us-east-1 | Menor custo, maior disponibilidade |
| Nodes | t3.medium Spot | Custo mínimo viável |
| Topologia | 3 nodes, 3 AZs | HA para Vault Raft quorum |
| IAM | EKS Pod Identity | Padrão atual, mais simples que IRSA |
| Rede | VPC CNI Network Policy | Nativo AWS, sem dependência extra |
| Vault | HA Raft + KMS unseal | Requer quorum (3 pods), unseal automático para Spot |
| Secrets | External Secrets Operator | Desacoplado, padrão de mercado |
| Ingress | AWS LBC + ALB | Nativo AWS, integração com ACM e WAF |
| TLS | cert-manager + Let's Encrypt | Gratuito, automático |
| Registry | Harbor + S3 | Persistência independente de pods |
| GitOps | ArgoCD App of Apps | Escala para múltiplos apps, UI clara |
| Observabilidade | Não nesta fase | Foco no núcleo do blueprint |

## Localização Local dos Repositórios

| Repositório | Caminho local |
|---|---|
| `eks-gitops-blueprint` | `C:\projetos\claude\eks-gitops-blueprint` |
| `infra-gitops-delivery-blueprint` | `C:\projetos\claude\infra-gitops-delivery-blueprint` |
| `infra-gitops-delivery` | `C:\projetos\claude\infra-gitops-delivery` |

## Fases do Projeto

- **Fase 1 (concluída)**: Decisões arquiteturais, CLAUDE.md e repositório de documentação (`eks-gitops-blueprint`)
- **Fase 2 (concluída)**: Repositório de manifestos criado localmente (`infra-gitops-delivery-blueprint` + `infra-gitops-delivery`) — pendente push para GitHub
- **Fase 3**: Push dos repositórios para GitHub e execução do laboratório (VPC → EKS → ArgoCD → Vault → Harbor)
- **Fase 4**: Repositório de aplicação e deploy end-to-end no cluster
