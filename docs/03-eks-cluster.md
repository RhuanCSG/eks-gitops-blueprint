# 03. Cluster EKS

Criamos o cluster EKS com Managed Node Groups usando Spot Instances em 3 Availability Zones. Habilitamos os addons necessários: EKS Pod Identity, VPC CNI com Network Policy e EBS CSI Driver.

## Pré-requisitos

Certifique-se de que as variáveis do ambiente estão carregadas:

=== "Linux / macOS"

    ```bash
    source ~/eks-lab-vars.env
    echo "Cluster: $CLUSTER_NAME | VPC: $VPC_ID | Região: $AWS_REGION"
    ```

=== "Windows (PowerShell)"

    ```powershell
    . ~\eks-lab-vars.ps1
    Write-Host "Cluster: $CLUSTER_NAME | VPC: $VPC_ID | Região: $AWS_REGION"
    ```

## Criação do Cluster

### 1. Arquivo de configuração do eksctl

=== "Linux / macOS"

    ```bash
    cat > eks-cluster.yaml << EOF
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig

    metadata:
      name: ${CLUSTER_NAME}
      region: ${AWS_REGION}
      version: "1.32"
      tags:
        project: eks-gitops-blueprint
        environment: lab

    iam:
      withOIDC: false

    vpc:
      id: "${VPC_ID}"
      subnets:
        private:
          us-east-1a:
            id: "${PRIV_SUBNET_1A}"
          us-east-1b:
            id: "${PRIV_SUBNET_1B}"
          us-east-1c:
            id: "${PRIV_SUBNET_1C}"
        public:
          us-east-1a:
            id: "${PUB_SUBNET_1A}"
          us-east-1b:
            id: "${PUB_SUBNET_1B}"
          us-east-1c:
            id: "${PUB_SUBNET_1C}"

    managedNodeGroups:
      - name: spot-nodes
        instanceTypes:
          - t3.medium
          - t3a.medium
        spot: true
        minSize: 3
        maxSize: 6
        desiredCapacity: 3
        amiFamily: AmazonLinux2023
        volumeSize: 30
        volumeType: gp3
        volumeEncrypted: true
        privateNetworking: true
        subnets:
          - ${PRIV_SUBNET_1A}
          - ${PRIV_SUBNET_1B}
          - ${PRIV_SUBNET_1C}
        labels:
          role: worker
        tags:
          k8s.io/cluster-autoscaler/enabled: "true"
          k8s.io/cluster-autoscaler/${CLUSTER_NAME}: "owned"

    addons:
      - name: vpc-cni
        version: latest
        configurationValues: |-
          enableNetworkPolicy: "true"
      - name: coredns
        version: latest
      - name: kube-proxy
        version: latest
      - name: aws-ebs-csi-driver
        version: latest
      - name: eks-pod-identity-agent
        version: latest
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @"
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig

    metadata:
      name: $CLUSTER_NAME
      region: $AWS_REGION
      version: "1.32"
      tags:
        project: eks-gitops-blueprint
        environment: lab

    iam:
      withOIDC: false

    vpc:
      id: "$VPC_ID"
      subnets:
        private:
          us-east-1a:
            id: "$PRIV_SUBNET_1A"
          us-east-1b:
            id: "$PRIV_SUBNET_1B"
          us-east-1c:
            id: "$PRIV_SUBNET_1C"
        public:
          us-east-1a:
            id: "$PUB_SUBNET_1A"
          us-east-1b:
            id: "$PUB_SUBNET_1B"
          us-east-1c:
            id: "$PUB_SUBNET_1C"

    managedNodeGroups:
      - name: spot-nodes
        instanceTypes:
          - t3.medium
          - t3a.medium
        spot: true
        minSize: 3
        maxSize: 6
        desiredCapacity: 3
        amiFamily: AmazonLinux2023
        volumeSize: 30
        volumeType: gp3
        volumeEncrypted: true
        privateNetworking: true
        subnets:
          - $PRIV_SUBNET_1A
          - $PRIV_SUBNET_1B
          - $PRIV_SUBNET_1C
        labels:
          role: worker
        tags:
          k8s.io/cluster-autoscaler/enabled: "true"
          k8s.io/cluster-autoscaler/${CLUSTER_NAME}: "owned"

    addons:
      - name: vpc-cni
        version: latest
        configurationValues: |-
          enableNetworkPolicy: "true"
      - name: coredns
        version: latest
      - name: kube-proxy
        version: latest
      - name: aws-ebs-csi-driver
        version: latest
      - name: eks-pod-identity-agent
        version: latest
    "@ | Set-Content eks-cluster.yaml
    ```

!!! tip "Windows"
    Salve o conteúdo acima em `~\eks-cluster.yaml` e referencie pelo caminho ao executar o `eksctl`.

### 2. Criar o cluster

```bash
eksctl create cluster -f eks-cluster.yaml
```

!!! info "Tempo estimado"
    A criação leva entre 15 e 20 minutos via CloudFormation.

### 3. Configurar o kubeconfig

```bash
aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION

kubectl get nodes -o wide
```

Saída esperada:

```
NAME                          STATUS   ROLES    AGE   VERSION
ip-10-0-11-xxx.ec2.internal   Ready    <none>   2m    v1.32.x
ip-10-0-12-xxx.ec2.internal   Ready    <none>   2m    v1.32.x
ip-10-0-13-xxx.ec2.internal   Ready    <none>   2m    v1.32.x
```

!!! note "Comandos multiplataforma"
    `eksctl`, `aws eks update-kubeconfig`, `kubectl` e `helm` funcionam identicamente no Linux, macOS e Windows. Só as construções de shell (variáveis, arquivos) diferem.

## Verificação dos Addons

```bash
aws eks list-addons --cluster-name $CLUSTER_NAME --region $AWS_REGION
```

Todos os addons devem estar com status `ACTIVE`.

## StorageClass padrão (gp3)

=== "Linux / macOS"

    ```bash
    kubectl apply -f - << 'EOF'
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: gp3
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
    provisioner: ebs.csi.aws.com
    parameters:
      type: gp3
      encrypted: "true"
    volumeBindingMode: WaitForFirstConsumer
    allowVolumeExpansion: true
    EOF

    kubectl patch storageclass gp2 \
      -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'
    ```

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: gp3
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
    provisioner: ebs.csi.aws.com
    parameters:
      type: gp3
      encrypted: "true"
    volumeBindingMode: WaitForFirstConsumer
    allowVolumeExpansion: true
    '@ | kubectl apply -f -

    kubectl patch storageclass gp2 `
      -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'
    ```

## Namespaces

=== "Linux / macOS"

    ```bash
    kubectl apply -f - << 'EOF'
    apiVersion: v1
    kind: Namespace
    metadata:
      name: argocd
      labels:
        app.kubernetes.io/name: argocd
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: vault
      labels:
        app.kubernetes.io/name: vault
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: harbor
      labels:
        app.kubernetes.io/name: harbor
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: external-secrets
      labels:
        app.kubernetes.io/name: external-secrets
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: cert-manager
      labels:
        app.kubernetes.io/name: cert-manager
    EOF
    ```

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: v1
    kind: Namespace
    metadata:
      name: argocd
      labels:
        app.kubernetes.io/name: argocd
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: vault
      labels:
        app.kubernetes.io/name: vault
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: harbor
      labels:
        app.kubernetes.io/name: harbor
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: external-secrets
      labels:
        app.kubernetes.io/name: external-secrets
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: cert-manager
      labels:
        app.kubernetes.io/name: cert-manager
    '@ | kubectl apply -f -
    ```

## Checklist

- [ ] Cluster EKS criado com versão 1.32+
- [ ] 3 nodes t3.medium Spot em 3 AZs (status `Ready`)
- [ ] Addons ativos: vpc-cni, coredns, kube-proxy, aws-ebs-csi-driver, eks-pod-identity-agent
- [ ] Network Policy habilitada no vpc-cni
- [ ] StorageClass `gp3` configurada como padrão
- [ ] Namespaces criados: argocd, vault, harbor, external-secrets, cert-manager

Próximo passo: [Instalação do ArgoCD](04-argocd.md).
