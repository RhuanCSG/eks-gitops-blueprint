# 09. AWS Load Balancer Controller

O AWS Load Balancer Controller (AWS LBC) provisiona ALBs automaticamente a partir de recursos `Ingress` com `ingressClassName: alb`.

## Instalação via ArgoCD

O AWS LBC é instalado automaticamente pelo ArgoCD na Wave 1. Confirme que os pods estão Running:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
# Deve mostrar 2 pods Running (HA)
kubectl get ingressclass
# NAME   CONTROLLER            PARAMETERS   AGE
# alb    ingress.k8s.aws/alb   <none>       ...
```

## IAM via EKS Pod Identity

O AWS LBC precisa de uma IAM role com permissões para gerenciar ALBs, Security Groups e Target Groups.

!!! warning "Pré-requisito obrigatório"
    A Pod Identity Association deve ser criada **antes** de aplicar qualquer Ingress. Sem ela, o controller usa a Node Instance Role (sem permissões de ELB) e loga `AccessDenied: elasticloadbalancing:DescribeLoadBalancers`.

### 1. Baixar a policy oficial e criar na AWS

=== "Linux / macOS"

    ```bash
    curl -o lbc-iam-policy.json \
      https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.1/docs/install/iam_policy.json

    LBC_POLICY_ARN=$(aws iam create-policy \
      --policy-name AWSLoadBalancerControllerIAMPolicy \
      --policy-document file://lbc-iam-policy.json \
      --query 'Policy.Arn' --output text)

    echo $LBC_POLICY_ARN
    ```

=== "Windows (PowerShell)"

    ```powershell
    curl -o lbc-iam-policy.json `
      https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.1/docs/install/iam_policy.json

    $LBC_POLICY_ARN = aws iam create-policy `
      --policy-name AWSLoadBalancerControllerIAMPolicy `
      --policy-document file://lbc-iam-policy.json `
      --query 'Policy.Arn' --output text

    Write-Host $LBC_POLICY_ARN
    ```

### 2. Criar a IAM role com trust policy para EKS Pod Identity

=== "Linux / macOS"

    ```bash
    cat > lbc-trust.json << 'EOF'
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "pods.eks.amazonaws.com"},
        "Action": ["sts:AssumeRole", "sts:TagSession"]
      }]
    }
    EOF

    LBC_ROLE_ARN=$(aws iam create-role \
      --role-name ${CLUSTER_NAME}-aws-lbc \
      --assume-role-policy-document file://lbc-trust.json \
      --query 'Role.Arn' --output text)

    aws iam attach-role-policy \
      --role-name ${CLUSTER_NAME}-aws-lbc \
      --policy-arn $LBC_POLICY_ARN
    ```

=== "Windows (PowerShell)"

    ```powershell
    $trustPolicy = '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}'

    $LBC_ROLE_ARN = aws iam create-role `
      --role-name "${env:CLUSTER_NAME}-aws-lbc" `
      --assume-role-policy-document $trustPolicy `
      --query 'Role.Arn' --output text

    aws iam attach-role-policy `
      --role-name "${env:CLUSTER_NAME}-aws-lbc" `
      --policy-arn $LBC_POLICY_ARN
    ```

### 3. Criar a Pod Identity Association

```bash
aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace kube-system \
  --service-account aws-load-balancer-controller \
  --role-arn $LBC_ROLE_ARN
```

### 4. Reiniciar os pods do AWS LBC

A Pod Identity Association é injetada no momento de criação do pod. Pods já existentes precisam ser reiniciados:

```bash
kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system
kubectl rollout status deployment/aws-load-balancer-controller -n kube-system
```

Confirme que o controller está usando a role correta (não deve mais aparecer `NodeInstanceRole` nos logs):

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=10
```

## Expor Harbor, Vault e ArgoCD via Ingress

Crie os arquivos em `manifests/ingresses/` no repositório de infra. O ArgoCD os aplicará automaticamente.

!!! note "Sem domínio customizado"
    Este lab usa os hostnames gerados pelo ALB diretamente (sem domínio próprio). Por isso os Ingresses usam apenas HTTP. TLS com Let's Encrypt exigiria um domínio real configurado no Route 53.

```yaml title="manifests/ingresses/harbor-ingress.yaml"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: harbor
  namespace: harbor
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: harbor
                port:
                  number: 80
```

!!! warning "Harbor: porta real do pod é 8080"
    Com `target-type: ip`, o ALB fala diretamente com o pod na porta do container (8080), não na porta do Service (80). A NetworkPolicy do namespace `harbor` deve permitir ingresso na porta **8080** (além de 80, 443, 5000).

```yaml title="manifests/ingresses/vault-ingress.yaml"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vault-ui
  namespace: vault
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /v1/sys/health
    alb.ingress.kubernetes.io/success-codes: 200,429,472,473
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vault-ui
                port:
                  number: 8200
```

!!! warning "Vault: health check customizado obrigatório"
    O Vault retorna códigos HTTP não-padrão (`429` standby, `472`/`473` recovery mode). Sem as anotações `healthcheck-path` e `success-codes`, o ALB considera todos os targets unhealthy e retorna 502.

### ArgoCD: habilitar modo insecure antes do Ingress

!!! note "Decisão de lab — não use em produção"
    O ArgoCD redireciona HTTP → HTTPS por padrão e exige um certificado TLS válido. Em produção, o correto é configurar um domínio real no Route 53, emitir um certificado via cert-manager + Let's Encrypt e usar `listen-ports: '[{"HTTPS": 443}]'` no Ingress.

    Neste lab optamos por **não usar domínio customizado** para reduzir custo e escopo. Por isso habilitamos o modo `insecure` do ArgoCD, que desativa o redirect HTTPS e permite servir via HTTP puro pelo ALB.

```bash
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge -p '{"data":{"server.insecure":"true"}}'

kubectl rollout restart deployment/argocd-server -n argocd
```

```yaml title="manifests/ingresses/argocd-ingress.yaml"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/success-codes: "200"
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

!!! tip "Health check obrigatório"
    O ArgoCD retorna redirect em `/`. Sem `healthcheck-path: /healthz`, o ALB considera todos os targets unhealthy e retorna 502.

### Obter os hostnames dos ALBs

```bash
kubectl get ingress -A
```

Aguarde ~2 minutos após o push para o ADDRESS ser preenchido pelo AWS LBC.

=== "Linux / macOS"

    ```bash
    HARBOR_ALB=$(kubectl get ingress harbor -n harbor \
      -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
    VAULT_ALB=$(kubectl get ingress vault-ui -n vault \
      -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

    echo "Harbor: http://$HARBOR_ALB"
    echo "Vault:  http://$VAULT_ALB"
    ```

=== "Windows (PowerShell)"

    ```powershell
    $HARBOR_ALB = kubectl get ingress harbor -n harbor `
      -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
    $VAULT_ALB = kubectl get ingress vault-ui -n vault `
      -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

    Write-Host "Harbor: http://$HARBOR_ALB"
    Write-Host "Vault:  http://$VAULT_ALB"
    ```

## Atualizar externalURL do Harbor

Após obter o hostname do ALB do Harbor, atualize o `externalURL` no `charts/harbor/values.yaml`:

```yaml title="charts/harbor/values.yaml (trecho)"
externalURL: http://<HARBOR_ALB_HOSTNAME>
```

Faça o push — o ArgoCD atualizará o Harbor automaticamente. O Harbor usa o `externalURL` para gerar os links de pull/push nas instruções da UI.

## Salvar hostname no arquivo de variáveis

=== "Linux / macOS"

    ```bash
    echo "export HARBOR_ALB=\"$HARBOR_ALB\"" >> ~/eks-lab-vars.env
    echo "export VAULT_ALB=\"$VAULT_ALB\"" >> ~/eks-lab-vars.env
    ```

=== "Windows (PowerShell)"

    ```powershell
    "`$HARBOR_ALB = `"$HARBOR_ALB`"" | Add-Content ~\eks-lab-vars.ps1
    "`$VAULT_ALB  = `"$VAULT_ALB`""  | Add-Content ~\eks-lab-vars.ps1
    ```

## Checklist

- [ ] IAM policy `AWSLoadBalancerControllerIAMPolicy` criada
- [ ] IAM role `<CLUSTER_NAME>-aws-lbc` criada com trust policy para EKS Pod Identity
- [ ] Pod Identity Association configurada para SA `aws-load-balancer-controller` em `kube-system`
- [ ] Pods do AWS LBC reiniciados e `Running` (sem erros `AccessDenied` nos logs)
- [ ] `IngressClass alb` presente no cluster
- [ ] NetworkPolicy do Harbor atualizada para permitir porta 8080 (target-type ip)
- [ ] Ingress `harbor` (namespace `harbor`) com ADDRESS preenchido e UI acessível
- [ ] Ingress `vault-ui` (namespace `vault`) com health check customizado e UI acessível
- [ ] ArgoCD configurado com `server.insecure: true` e reiniciado
- [ ] Ingress `argocd` (namespace `argocd`) com ADDRESS preenchido e UI acessível
- [ ] `externalURL` do Harbor atualizado no `charts/harbor/values.yaml`

---

Com esta etapa concluída, todas as ferramentas estão acessíveis via HTTP pelos hostnames dos ALBs.

| Ferramenta | Acesso | Credenciais |
|---|---|---|
| Harbor | `http://<HARBOR_ALB>` | admin / (do Vault em `secret/harbor/admin`) |
| Vault  | `http://<VAULT_ALB>` | Token root (do vault-init.json) |
| ArgoCD | `http://<ARGOCD_ALB>` | admin / (kubectl get secret argocd-initial-admin-secret) |

Próximo passo: [Destruição dos Recursos](10-cleanup.md).
