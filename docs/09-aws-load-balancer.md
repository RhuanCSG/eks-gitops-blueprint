# 09. AWS Load Balancer Controller

O AWS Load Balancer Controller provisiona ALBs automaticamente a partir de recursos `Ingress`.

## IAM via EKS Pod Identity

=== "Linux / macOS"

    ```bash
    cat > lbc-pod-identity-trust.json << 'EOF'
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

    LBC_ROLE_ARN=$(aws iam create-role \
      --role-name "${CLUSTER_NAME}-aws-lbc" \
      --assume-role-policy-document file://lbc-pod-identity-trust.json \
      --query 'Role.Arn' --output text)

    curl -o lbc-iam-policy.json \
      https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

    LBC_POLICY_ARN=$(aws iam create-policy \
      --policy-name "${CLUSTER_NAME}-AWSLoadBalancerControllerIAMPolicy" \
      --policy-document file://lbc-iam-policy.json \
      --query 'Policy.Arn' --output text)

    aws iam attach-role-policy \
      --role-name "${CLUSTER_NAME}-aws-lbc" \
      --policy-arn $LBC_POLICY_ARN

    aws eks create-pod-identity-association \
      --cluster-name $CLUSTER_NAME \
      --namespace kube-system \
      --service-account aws-load-balancer-controller \
      --role-arn $LBC_ROLE_ARN \
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
    '@ | Set-Content lbc-pod-identity-trust.json

    $LBC_ROLE_ARN = aws iam create-role `
      --role-name "${CLUSTER_NAME}-aws-lbc" `
      --assume-role-policy-document file://lbc-pod-identity-trust.json `
      --query 'Role.Arn' --output text

    Invoke-WebRequest `
      -Uri https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json `
      -OutFile lbc-iam-policy.json

    $LBC_POLICY_ARN = aws iam create-policy `
      --policy-name "${CLUSTER_NAME}-AWSLoadBalancerControllerIAMPolicy" `
      --policy-document file://lbc-iam-policy.json `
      --query 'Policy.Arn' --output text

    aws iam attach-role-policy `
      --role-name "${CLUSTER_NAME}-aws-lbc" `
      --policy-arn $LBC_POLICY_ARN

    aws eks create-pod-identity-association `
      --cluster-name $CLUSTER_NAME `
      --namespace kube-system `
      --service-account aws-load-balancer-controller `
      --role-arn $LBC_ROLE_ARN `
      --region $AWS_REGION
    ```

## Instalação via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

=== "Linux / macOS"

    ```bash
    LBC_VERSION=$(helm search repo eks/aws-load-balancer-controller --output json | jq -r '.[0].version')

    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
      --namespace kube-system \
      --version "$LBC_VERSION" \
      --set clusterName=$CLUSTER_NAME \
      --set serviceAccount.create=true \
      --set serviceAccount.name=aws-load-balancer-controller \
      --wait
    ```

=== "Windows (PowerShell)"

    ```powershell
    $LBC_VERSION = (helm search repo eks/aws-load-balancer-controller --output json | ConvertFrom-Json)[0].version

    helm install aws-load-balancer-controller eks/aws-load-balancer-controller `
      --namespace kube-system `
      --version "$LBC_VERSION" `
      --set clusterName=$CLUSTER_NAME `
      --set serviceAccount.create=true `
      --set serviceAccount.name=aws-load-balancer-controller `
      --wait
    ```

## Expor as Ferramentas via Ingress

### ArgoCD

=== "Linux / macOS"

    ```bash
    kubectl apply -f - << 'EOF'
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: argocd-ingress
      namespace: argocd
      annotations:
        kubernetes.io/ingress.class: alb
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
        alb.ingress.kubernetes.io/ssl-redirect: "443"
        cert-manager.io/cluster-issuer: letsencrypt-staging
    spec:
      tls:
        - hosts:
            - <ARGOCD_ALB_HOSTNAME>
          secretName: argocd-tls
      rules:
        - host: <ARGOCD_ALB_HOSTNAME>
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: argocd-server
                    port:
                      number: 80
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: argocd-ingress
      namespace: argocd
      annotations:
        kubernetes.io/ingress.class: alb
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
        alb.ingress.kubernetes.io/ssl-redirect: "443"
        cert-manager.io/cluster-issuer: letsencrypt-staging
    spec:
      tls:
        - hosts:
            - <ARGOCD_ALB_HOSTNAME>
          secretName: argocd-tls
      rules:
        - host: <ARGOCD_ALB_HOSTNAME>
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: argocd-server
                    port:
                      number: 80
    '@ | kubectl apply -f -
    ```

!!! note "Hostname do ALB"
    Aplique o Ingress, aguarde ~2 minutos e obtenha o hostname com `kubectl get ingress argocd-ingress -n argocd`. Atualize o campo `host` com o valor real e reaplicar.

### Obter hostname após provisionamento

=== "Linux / macOS"

    ```bash
    ARGOCD_ALB=$(kubectl get ingress argocd-ingress -n argocd \
      -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

    echo "ArgoCD URL: https://$ARGOCD_ALB"
    ```

=== "Windows (PowerShell)"

    ```powershell
    $ARGOCD_ALB = kubectl get ingress argocd-ingress -n argocd `
      -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

    Write-Host "ArgoCD URL: https://$ARGOCD_ALB"
    ```

### Vault e Harbor (mesmo padrão)

Repita o mesmo padrão de Ingress para `vault-ingress` (namespace `vault`, service `vault-ui:8200`) e `harbor-ingress` (namespace `harbor`, service `harbor-portal:80`), substituindo `<ARGOCD_ALB_HOSTNAME>` pelo hostname gerado para cada serviço.

## Migrar de Staging para Produção

Após validar que os certificados staging funcionam:

```bash
kubectl patch ingress argocd-ingress -n argocd \
  --type=json \
  -p='[{"op": "replace", "path": "/metadata/annotations/cert-manager.io~1cluster-issuer", "value": "letsencrypt-prod"}]'

kubectl delete secret argocd-tls -n argocd
# Repetir para vault e harbor
```

## Salvar URL Final

=== "Linux / macOS"

    ```bash
    echo "export ARGOCD_URL=\"https://$ARGOCD_ALB\"" >> ~/eks-lab-vars.env
    ```

=== "Windows (PowerShell)"

    ```powershell
    "`$ARGOCD_URL = `"https://$ARGOCD_ALB`"" | Add-Content ~\eks-lab-vars.ps1
    ```

## Checklist

- [ ] Role IAM do AWS LBC criada com a policy oficial
- [ ] Pod Identity Association configurada para `aws-load-balancer-controller`
- [ ] AWS LBC instalado e pod em `Running`
- [ ] Ingress criado para ArgoCD, Vault e Harbor com ALBs provisionados
- [ ] Certificados emitidos pelo cert-manager (status `True`)
- [ ] Certificados migrados de staging para letsencrypt-prod

---

## Cluster Completo

Com esta etapa concluída, o cluster EKS está totalmente operacional:

| Ferramenta | Acesso |
|---|---|
| ArgoCD | `https://<ARGOCD_ALB>` |
| Vault | `https://<VAULT_ALB>` |
| Harbor | `https://<HARBOR_ALB>` |

O próximo passo é o repositório de manifestos `infra-gitops-delivery`, transferindo o controle de toda a infraestrutura para o GitOps.
