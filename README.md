<div align="center">

<br/>
```
 ██████╗ ██╗     ██╗   ██╗███████╗██████╗ ██████╗ ██╗███╗   ██╗████████╗
 ██╔══██╗██║     ██║   ██║██╔════╝██╔══██╗██╔══██╗██║████╗  ██║╚══██╔══╝
 ██████╔╝██║     ██║   ██║█████╗  ██████╔╝██████╔╝██║██╔██╗ ██║   ██║
 ██╔══██╗██║     ██║   ██║██╔══╝  ██╔═══╝ ██╔══██╗██║██║╚██╗██║   ██║
 ██████╔╝███████╗╚██████╔╝███████╗██║     ██║  ██║██║██║ ╚████║   ██║
 ╚═════╝ ╚══════╝ ╚═════╝ ╚══════╝╚═╝     ╚═╝  ╚═╝╚═╝╚═╝  ╚═══╝   ╚═╝
```

### EKS GitOps Blueprint

**Guia completo para construir um cluster EKS production-grade com GitOps**

<br/>

[![Documentação](https://img.shields.io/badge/Documentação-GitHub%20Pages-0969DA?style=flat-square&logo=github&logoColor=white)](https://RhuanCSG.github.io/eks-gitops-blueprint/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)
[![AWS EKS](https://img.shields.io/badge/AWS-EKS%201.32+-FF9900?style=flat-square&logo=amazonaws&logoColor=white)](https://aws.amazon.com/eks/)
[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?style=flat-square&logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)
[![Vault](https://img.shields.io/badge/Secrets-Vault%20HA-000000?style=flat-square&logo=vault&logoColor=white)](https://developer.hashicorp.com/vault)
[![Harbor](https://img.shields.io/badge/Registry-Harbor-60B932?style=flat-square)](https://goharbor.io/)

<br/>

**[→ Abrir Documentação Completa](https://RhuanCSG.github.io/eks-gitops-blueprint/)**

<br/>

</div>

---

## Sobre o Projeto

Este repositório documenta, passo a passo, a construção de um laboratório de infraestrutura GitOps utilizando AWS EKS. Cada decisão arquitetural é justificada, cada comando é explicado, e boas práticas de produção são aplicadas desde a primeira linha — mesmo sendo um laboratório de estudos.

Ao final do guia, você terá um cluster completo operacional e um modelo reutilizável para ambientes reais.

---

## O que você vai construir

```mermaid
graph TB
    Dev([Developer])
    Internet((Internet))

    subgraph AWS["AWS us-east-1"]
        GH[(GitHub\ninfra-gitops-delivery)]
        KMS[(KMS\nVault unseal)]
        S3[(S3\nHarbor images)]

        subgraph VPC["VPC  10.0.0.0/16"]
            ALB["AWS ALB\ninternet-facing\n10.0.1-3.0/24"]

            subgraph EKS["EKS Managed Nodes — t3.medium Spot  —  10.0.11-13.0/24"]
                ArgoCD[ArgoCD]
                Vault[Vault HA+Raft]
                Harbor[Harbor]
                ESO[External Secrets]
                LBC[AWS LBC]
                CM[cert-manager]
            end
        end
    end

    Dev -->|git push| GH
    Internet --> ALB
    GH -->|sync 30s| ArgoCD
    ALB --> ArgoCD
    ALB --> Vault
    ALB --> Harbor
    LBC -.->|provisiona| ALB
    ESO --> Vault
    Vault --> KMS
    Harbor --> S3
```

**Fluxo GitOps com sync waves:**

```mermaid
sequenceDiagram
    actor Dev as Developer
    participant GH as GitHub
    participant CD as ArgoCD
    participant EKS as EKS Cluster

    Dev->>GH: git push
    GH-->>CD: webhook / poll 30s
    rect rgb(240, 248, 255)
        Note over CD,EKS: Wave 0 — Namespaces · NetworkPolicies · StorageClass gp3
        CD->>EKS: cluster-manifests
    end
    rect rgb(240, 248, 255)
        Note over CD,EKS: Wave 1 — cert-manager · AWS Load Balancer Controller
        CD->>EKS: cert-manager, aws-lbc
    end
    rect rgb(240, 248, 255)
        Note over CD,EKS: Wave 2 — External Secrets Operator
        CD->>EKS: external-secrets
    end
    rect rgb(240, 248, 255)
        Note over CD,EKS: Wave 3 — Vault HA + Raft + KMS auto-unseal
        CD->>EKS: vault
    end
    rect rgb(240, 248, 255)
        Note over CD,EKS: Wave 4 — Harbor + S3 backend
        CD->>EKS: harbor
    end
    EKS-->>Dev: cluster operacional
```

---

## Guia em 10 Etapas

| # | Etapa | O que você faz |
|:---:|---|---|
| **01** | [Pré-requisitos](https://RhuanCSG.github.io/eks-gitops-blueprint/01-prerequisites/) | Instalar ferramentas, configurar AWS CLI e credenciais |
| **02** | [VPC e Networking](https://RhuanCSG.github.io/eks-gitops-blueprint/02-vpc-networking/) | Criar VPC, subnets públicas/privadas, NAT, KMS, S3 |
| **03** | [Cluster EKS](https://RhuanCSG.github.io/eks-gitops-blueprint/03-eks-cluster/) | Provisionar EKS com Spot Instances, addons e StorageClass gp3 |
| **04** | [ArgoCD](https://RhuanCSG.github.io/eks-gitops-blueprint/04-argocd/) | Instalar ArgoCD e configurar o padrão App of Apps |
| **05** | [HashiCorp Vault](https://RhuanCSG.github.io/eks-gitops-blueprint/05-vault/) | Vault HA com Raft, KMS auto-unseal e Kubernetes auth |
| **06** | [Harbor Registry](https://RhuanCSG.github.io/eks-gitops-blueprint/06-harbor/) | Registry privado com backend S3 e robot accounts |
| **07** | [External Secrets](https://RhuanCSG.github.io/eks-gitops-blueprint/07-external-secrets/) | Sincronizar segredos do Vault para Kubernetes Secrets |
| **08** | [cert-manager](https://RhuanCSG.github.io/eks-gitops-blueprint/08-cert-manager/) | Certificados TLS automáticos com Let's Encrypt |
| **09** | [AWS Load Balancer](https://RhuanCSG.github.io/eks-gitops-blueprint/09-aws-load-balancer/) | ALBs automáticos e exposição das ferramentas via HTTP |
| **10** | [Destruição dos Recursos](https://RhuanCSG.github.io/eks-gitops-blueprint/10-cleanup/) | Remover todos os recursos AWS para evitar cobranças |

> [!TIP]
> Todos os comandos têm versões para **Linux/macOS** e **Windows (PowerShell)** — sem necessidade de WSL.

---

## Stack

<table>
<thead>
<tr><th>Camada</th><th>Tecnologia</th><th>Decisão</th></tr>
</thead>
<tbody>
<tr><td>Cloud</td><td>AWS us-east-1</td><td>Menor custo, maior disponibilidade de Spot</td></tr>
<tr><td>Kubernetes</td><td>Amazon EKS 1.32+</td><td>Control plane gerenciado</td></tr>
<tr><td>Nodes</td><td>t3.medium · Spot · 3 AZs</td><td>Custo mínimo com alta disponibilidade</td></tr>
<tr><td>SO</td><td>Amazon Linux 2023</td><td>Suporte ativo, sem versões deprecadas</td></tr>
<tr><td>Storage</td><td>EBS gp3 (criptografado)</td><td>Padrão atual — sem gp2</td></tr>
<tr><td>IAM</td><td>EKS Pod Identity</td><td>Padrão mais recente — sem IRSA</td></tr>
<tr><td>Network Policy</td><td>VPC CNI nativo</td><td>Sem CNI adicional (Calico)</td></tr>
<tr><td>GitOps</td><td>ArgoCD · App of Apps</td><td>Sync waves, UI, reconciliação contínua</td></tr>
<tr><td>Secrets</td><td>Vault HA + Raft + KMS</td><td>HA sem backend externo, unseal automático</td></tr>
<tr><td>Secrets sync</td><td>External Secrets Operator</td><td>Desacoplado, auditável</td></tr>
<tr><td>Registry</td><td>Harbor + S3</td><td>Persistência independente dos pods</td></tr>
<tr><td>Ingress</td><td>AWS LBC + ALB</td><td>Nativo AWS, integra com ACM e WAF</td></tr>
<tr><td>TLS</td><td>cert-manager + Let's Encrypt</td><td>Gratuito, automatizado</td></tr>
<tr><td>Docs</td><td>MkDocs Material + GitHub Pages</td><td>Documentação navegável, sem custo</td></tr>
</tbody>
</table>

---

## Boas Práticas Aplicadas

Este laboratório não abre mão de padrões de produção — mesmo sendo um ambiente de estudos:

- ✅ **EKS Pod Identity** — substituição definitiva do IRSA
- ✅ **Vault auto-unseal via KMS** — essencial com Spot Instances (pods reiniciam sem aviso)
- ✅ **NetworkPolicy default-deny** — isolamento de rede entre namespaces desde o início
- ✅ **EBS gp3 criptografado** — sem volumes em texto claro, sem `gp2`
- ✅ **Amazon Linux 2023** — SO com suporte ativo nos nodes
- ✅ **Sem secrets em repositórios** — tudo gerenciado via Vault + ESO
- ✅ **ArgoCD App of Apps** — infraestrutura 100% declarativa em Git
- ✅ **Sync waves** — ordem de deploy correta, sem race conditions

---

## Pré-requisitos

<details>
<summary>Ferramentas necessárias (expandir)</summary>

| Ferramenta | Versão mínima | Instalação |
|---|---|---|
| AWS CLI | v2.x | [docs.aws.amazon.com](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |
| kubectl | 1.32 | [kubernetes.io/docs](https://kubernetes.io/docs/tasks/tools/) |
| Helm | 3.14 | [helm.sh](https://helm.sh/docs/intro/install/) |
| eksctl | 0.190 | [eksctl.io](https://eksctl.io/installation/) |
| Git | qualquer | — |

</details>

**Conta AWS** com permissões para criar VPC, EKS, KMS, S3 e IAM roles.  
**Conta GitHub** com acesso a repositórios privados (para o ArgoCD acessar os manifestos).

---

## Repositórios do Projeto

| Repositório | Visibilidade | Finalidade |
|---|---|---|
| [`eks-gitops-blueprint`](https://github.com/RhuanCSG/eks-gitops-blueprint) | Público | Este repositório — documentação e guia |
| [`infra-gitops-delivery-blueprint`](https://github.com/RhuanCSG/infra-gitops-delivery-blueprint) | Público | Manifestos com placeholders — para fork |

> [!NOTE]
> Nunca armazene valores reais de contas AWS, ARNs, senhas ou credenciais em repositórios públicos. O repositório de manifestos deve permanecer **privado** após a substituição dos placeholders.

---

## Custo Estimado do Laboratório

| Recurso | Custo estimado |
|---|---|
| EKS Control Plane | ~$2,40/dia |
| 3× t3.medium Spot | ~$0,50/dia |
| NAT Gateway | ~$1,08/dia |
| ALB (por ferramenta) | ~$0,20/dia cada |
| KMS + S3 | < $0,10/dia |
| **Total** | **~$5–6/dia** |

> [!IMPORTANT]
> Destrua os recursos AWS ao finalizar o laboratório. O guia inclui os passos de limpeza na etapa final.

---

<div align="center">

<br/>

**[Começar pela Etapa 01 →](https://RhuanCSG.github.io/eks-gitops-blueprint/01-prerequisites/)**
&nbsp;&nbsp;|&nbsp;&nbsp;
**[Fork dos Manifestos →](https://github.com/RhuanCSG/infra-gitops-delivery-blueprint)**

<br/>

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)
[![Documentação](https://img.shields.io/badge/docs-GitHub%20Pages-0969DA?style=flat-square&logo=github)](https://RhuanCSG.github.io/eks-gitops-blueprint/)

<br/>

</div>
