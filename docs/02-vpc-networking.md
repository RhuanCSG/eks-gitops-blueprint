# 02. VPC e Networking

Criamos uma VPC dedicada com subnets públicas e privadas em 3 Availability Zones. Os nodes EKS ficam nas subnets privadas; os ALBs ficam nas subnets públicas.

## Design da Rede

```
VPC: 10.0.0.0/16
│
├── Subnets Públicas (ALB, NAT Gateway)
│   ├── 10.0.1.0/24  — us-east-1a
│   ├── 10.0.2.0/24  — us-east-1b
│   └── 10.0.3.0/24  — us-east-1c
│
└── Subnets Privadas (EKS Nodes)
    ├── 10.0.11.0/24 — us-east-1a
    ├── 10.0.12.0/24 — us-east-1b
    └── 10.0.13.0/24 — us-east-1c
```

- **Internet Gateway**: saída para internet nas subnets públicas
- **NAT Gateway**: único (custo de lab) em us-east-1a
- **Route Tables**: separadas para subnets públicas e privadas

!!! info "NAT Gateway único"
    Em produção, use um NAT Gateway por AZ. No laboratório, um único NAT reduz o custo (~$0,045/hora + dados transferidos).

## Criação dos Recursos

### 1. Variáveis de ambiente

=== "Linux / macOS"

    ```bash
    export AWS_REGION="us-east-1"
    export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
    export CLUSTER_NAME="eks-gitops-lab"
    export VPC_CIDR="10.0.0.0/16"
    ```

=== "Windows (PowerShell)"

    ```powershell
    $AWS_REGION   = "us-east-1"
    $AWS_ACCOUNT_ID = aws sts get-caller-identity --query Account --output text
    $CLUSTER_NAME = "eks-gitops-lab"
    $VPC_CIDR     = "10.0.0.0/16"
    ```

### 2. Criar a VPC

=== "Linux / macOS"

    ```bash
    VPC_ID=$(aws ec2 create-vpc \
      --cidr-block $VPC_CIDR \
      --region $AWS_REGION \
      --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=${CLUSTER_NAME}-vpc},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" \
      --query 'Vpc.VpcId' \
      --output text)

    echo "VPC_ID=$VPC_ID"

    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support '{"Value":true}'
    ```

=== "Windows (PowerShell)"

    ```powershell
    $VPC_ID = aws ec2 create-vpc `
      --cidr-block $VPC_CIDR `
      --region $AWS_REGION `
      --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=${CLUSTER_NAME}-vpc},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" `
      --query 'Vpc.VpcId' `
      --output text

    Write-Host "VPC_ID=$VPC_ID"

    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support '{"Value":true}'
    ```

### 3. Internet Gateway

=== "Linux / macOS"

    ```bash
    IGW_ID=$(aws ec2 create-internet-gateway \
      --region $AWS_REGION \
      --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=${CLUSTER_NAME}-igw}]" \
      --query 'InternetGateway.InternetGatewayId' \
      --output text)

    aws ec2 attach-internet-gateway \
      --internet-gateway-id $IGW_ID \
      --vpc-id $VPC_ID

    echo "IGW_ID=$IGW_ID"
    ```

=== "Windows (PowerShell)"

    ```powershell
    $IGW_ID = aws ec2 create-internet-gateway `
      --region $AWS_REGION `
      --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=${CLUSTER_NAME}-igw}]" `
      --query 'InternetGateway.InternetGatewayId' `
      --output text

    aws ec2 attach-internet-gateway `
      --internet-gateway-id $IGW_ID `
      --vpc-id $VPC_ID

    Write-Host "IGW_ID=$IGW_ID"
    ```

### 4. Subnets Públicas

=== "Linux / macOS"

    ```bash
    PUB_SUBNET_1A=$(aws ec2 create-subnet \
      --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-public-1a},{Key=kubernetes.io/role/elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" \
      --query 'Subnet.SubnetId' --output text)

    PUB_SUBNET_1B=$(aws ec2 create-subnet \
      --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1b \
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-public-1b},{Key=kubernetes.io/role/elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" \
      --query 'Subnet.SubnetId' --output text)

    PUB_SUBNET_1C=$(aws ec2 create-subnet \
      --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone us-east-1c \
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-public-1c},{Key=kubernetes.io/role/elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" \
      --query 'Subnet.SubnetId' --output text)

    echo "PUB_SUBNET_1A=$PUB_SUBNET_1A | PUB_SUBNET_1B=$PUB_SUBNET_1B | PUB_SUBNET_1C=$PUB_SUBNET_1C"
    ```

=== "Windows (PowerShell)"

    ```powershell
    $PUB_SUBNET_1A = aws ec2 create-subnet `
      --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a `
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-public-1a},{Key=kubernetes.io/role/elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" `
      --query 'Subnet.SubnetId' --output text

    $PUB_SUBNET_1B = aws ec2 create-subnet `
      --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1b `
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-public-1b},{Key=kubernetes.io/role/elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" `
      --query 'Subnet.SubnetId' --output text

    $PUB_SUBNET_1C = aws ec2 create-subnet `
      --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone us-east-1c `
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-public-1c},{Key=kubernetes.io/role/elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" `
      --query 'Subnet.SubnetId' --output text

    Write-Host "PUB_SUBNET_1A=$PUB_SUBNET_1A | PUB_SUBNET_1B=$PUB_SUBNET_1B | PUB_SUBNET_1C=$PUB_SUBNET_1C"
    ```

!!! note "Tags obrigatórias para o ALB"
    A tag `kubernetes.io/role/elb=1` nas subnets públicas é obrigatória para o AWS Load Balancer Controller descobrir e provisionar ALBs.

### 5. Subnets Privadas

=== "Linux / macOS"

    ```bash
    PRIV_SUBNET_1A=$(aws ec2 create-subnet \
      --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 --availability-zone us-east-1a \
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-private-1a},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" \
      --query 'Subnet.SubnetId' --output text)

    PRIV_SUBNET_1B=$(aws ec2 create-subnet \
      --vpc-id $VPC_ID --cidr-block 10.0.12.0/24 --availability-zone us-east-1b \
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-private-1b},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" \
      --query 'Subnet.SubnetId' --output text)

    PRIV_SUBNET_1C=$(aws ec2 create-subnet \
      --vpc-id $VPC_ID --cidr-block 10.0.13.0/24 --availability-zone us-east-1c \
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-private-1c},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" \
      --query 'Subnet.SubnetId' --output text)

    echo "PRIV_SUBNET_1A=$PRIV_SUBNET_1A | 1B=$PRIV_SUBNET_1B | 1C=$PRIV_SUBNET_1C"
    ```

=== "Windows (PowerShell)"

    ```powershell
    $PRIV_SUBNET_1A = aws ec2 create-subnet `
      --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 --availability-zone us-east-1a `
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-private-1a},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" `
      --query 'Subnet.SubnetId' --output text

    $PRIV_SUBNET_1B = aws ec2 create-subnet `
      --vpc-id $VPC_ID --cidr-block 10.0.12.0/24 --availability-zone us-east-1b `
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-private-1b},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" `
      --query 'Subnet.SubnetId' --output text

    $PRIV_SUBNET_1C = aws ec2 create-subnet `
      --vpc-id $VPC_ID --cidr-block 10.0.13.0/24 --availability-zone us-east-1c `
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-private-1c},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared}]" `
      --query 'Subnet.SubnetId' --output text

    Write-Host "PRIV_SUBNET_1A=$PRIV_SUBNET_1A | 1B=$PRIV_SUBNET_1B | 1C=$PRIV_SUBNET_1C"
    ```

### 6. NAT Gateway

=== "Linux / macOS"

    ```bash
    EIP_ALLOC_ID=$(aws ec2 allocate-address \
      --domain vpc \
      --tag-specifications "ResourceType=elastic-ip,Tags=[{Key=Name,Value=${CLUSTER_NAME}-nat-eip}]" \
      --query 'AllocationId' --output text)

    NAT_GW_ID=$(aws ec2 create-nat-gateway \
      --subnet-id $PUB_SUBNET_1A \
      --allocation-id $EIP_ALLOC_ID \
      --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=${CLUSTER_NAME}-nat}]" \
      --query 'NatGateway.NatGatewayId' --output text)

    echo "NAT_GW_ID=$NAT_GW_ID — aguardando disponibilidade..."

    aws ec2 wait nat-gateway-available \
      --filter "Name=nat-gateway-id,Values=$NAT_GW_ID"

    echo "NAT Gateway disponível."
    ```

=== "Windows (PowerShell)"

    ```powershell
    $EIP_ALLOC_ID = aws ec2 allocate-address `
      --domain vpc `
      --tag-specifications "ResourceType=elastic-ip,Tags=[{Key=Name,Value=${CLUSTER_NAME}-nat-eip}]" `
      --query 'AllocationId' --output text

    $NAT_GW_ID = aws ec2 create-nat-gateway `
      --subnet-id $PUB_SUBNET_1A `
      --allocation-id $EIP_ALLOC_ID `
      --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=${CLUSTER_NAME}-nat}]" `
      --query 'NatGateway.NatGatewayId' --output text

    Write-Host "NAT_GW_ID=$NAT_GW_ID — aguardando disponibilidade..."

    aws ec2 wait nat-gateway-available `
      --filter "Name=nat-gateway-id,Values=$NAT_GW_ID"

    Write-Host "NAT Gateway disponível."
    ```

### 7. Route Tables

=== "Linux / macOS"

    ```bash
    # Route table pública
    PUB_RT_ID=$(aws ec2 create-route-table \
      --vpc-id $VPC_ID \
      --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=${CLUSTER_NAME}-public-rt}]" \
      --query 'RouteTable.RouteTableId' --output text)

    aws ec2 create-route --route-table-id $PUB_RT_ID \
      --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

    for SUBNET in $PUB_SUBNET_1A $PUB_SUBNET_1B $PUB_SUBNET_1C; do
      aws ec2 associate-route-table --route-table-id $PUB_RT_ID --subnet-id $SUBNET
      aws ec2 modify-subnet-attribute --subnet-id $SUBNET --map-public-ip-on-launch
    done

    # Route table privada
    PRIV_RT_ID=$(aws ec2 create-route-table \
      --vpc-id $VPC_ID \
      --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=${CLUSTER_NAME}-private-rt}]" \
      --query 'RouteTable.RouteTableId' --output text)

    aws ec2 create-route --route-table-id $PRIV_RT_ID \
      --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_ID

    for SUBNET in $PRIV_SUBNET_1A $PRIV_SUBNET_1B $PRIV_SUBNET_1C; do
      aws ec2 associate-route-table --route-table-id $PRIV_RT_ID --subnet-id $SUBNET
    done
    ```

=== "Windows (PowerShell)"

    ```powershell
    # Route table pública
    $PUB_RT_ID = aws ec2 create-route-table `
      --vpc-id $VPC_ID `
      --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=${CLUSTER_NAME}-public-rt}]" `
      --query 'RouteTable.RouteTableId' --output text

    aws ec2 create-route --route-table-id $PUB_RT_ID `
      --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

    foreach ($SUBNET in @($PUB_SUBNET_1A, $PUB_SUBNET_1B, $PUB_SUBNET_1C)) {
      aws ec2 associate-route-table --route-table-id $PUB_RT_ID --subnet-id $SUBNET
      aws ec2 modify-subnet-attribute --subnet-id $SUBNET --map-public-ip-on-launch
    }

    # Route table privada
    $PRIV_RT_ID = aws ec2 create-route-table `
      --vpc-id $VPC_ID `
      --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=${CLUSTER_NAME}-private-rt}]" `
      --query 'RouteTable.RouteTableId' --output text

    aws ec2 create-route --route-table-id $PRIV_RT_ID `
      --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_ID

    foreach ($SUBNET in @($PRIV_SUBNET_1A, $PRIV_SUBNET_1B, $PRIV_SUBNET_1C)) {
      aws ec2 associate-route-table --route-table-id $PRIV_RT_ID --subnet-id $SUBNET
    }
    ```

### 8. Chave KMS para Vault Auto-unseal

=== "Linux / macOS"

    ```bash
    KMS_KEY_ARN=$(aws kms create-key \
      --description "Vault auto-unseal - ${CLUSTER_NAME}" \
      --key-usage ENCRYPT_DECRYPT \
      --query 'KeyMetadata.Arn' --output text)

    aws kms create-alias \
      --alias-name "alias/${CLUSTER_NAME}-vault-unseal" \
      --target-key-id $KMS_KEY_ARN

    echo "KMS_KEY_ARN=$KMS_KEY_ARN"
    ```

=== "Windows (PowerShell)"

    ```powershell
    $KMS_KEY_ARN = aws kms create-key `
      --description "Vault auto-unseal - ${CLUSTER_NAME}" `
      --key-usage ENCRYPT_DECRYPT `
      --query 'KeyMetadata.Arn' --output text

    aws kms create-alias `
      --alias-name "alias/${CLUSTER_NAME}-vault-unseal" `
      --target-key-id $KMS_KEY_ARN

    Write-Host "KMS_KEY_ARN=$KMS_KEY_ARN"
    ```

### 9. Bucket S3 para Harbor

=== "Linux / macOS"

    ```bash
    export HARBOR_S3_BUCKET="harbor-registry-${AWS_ACCOUNT_ID}"

    aws s3api create-bucket --bucket $HARBOR_S3_BUCKET --region $AWS_REGION

    aws s3api put-public-access-block --bucket $HARBOR_S3_BUCKET \
      --public-access-block-configuration \
        BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

    aws s3api put-bucket-versioning --bucket $HARBOR_S3_BUCKET \
      --versioning-configuration Status=Enabled

    aws s3api put-bucket-encryption --bucket $HARBOR_S3_BUCKET \
      --server-side-encryption-configuration '{
        "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
      }'

    echo "Harbor S3 bucket: $HARBOR_S3_BUCKET"
    ```

=== "Windows (PowerShell)"

    ```powershell
    $HARBOR_S3_BUCKET = "harbor-registry-${AWS_ACCOUNT_ID}"

    aws s3api create-bucket --bucket $HARBOR_S3_BUCKET --region $AWS_REGION

    aws s3api put-public-access-block --bucket $HARBOR_S3_BUCKET `
      --public-access-block-configuration `
        BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

    aws s3api put-bucket-versioning --bucket $HARBOR_S3_BUCKET `
      --versioning-configuration Status=Enabled

    aws s3api put-bucket-encryption --bucket $HARBOR_S3_BUCKET `
      --server-side-encryption-configuration '{\"Rules\":[{\"ApplyServerSideEncryptionByDefault\":{\"SSEAlgorithm\":\"AES256\"}}]}'

    Write-Host "Harbor S3 bucket: $HARBOR_S3_BUCKET"
    ```

## Salvar Variáveis

=== "Linux / macOS"

    ```bash
    cat > ~/eks-lab-vars.env << EOF
    export AWS_REGION="$AWS_REGION"
    export AWS_ACCOUNT_ID="$AWS_ACCOUNT_ID"
    export CLUSTER_NAME="$CLUSTER_NAME"
    export VPC_ID="$VPC_ID"
    export PUB_SUBNET_1A="$PUB_SUBNET_1A"
    export PUB_SUBNET_1B="$PUB_SUBNET_1B"
    export PUB_SUBNET_1C="$PUB_SUBNET_1C"
    export PRIV_SUBNET_1A="$PRIV_SUBNET_1A"
    export PRIV_SUBNET_1B="$PRIV_SUBNET_1B"
    export PRIV_SUBNET_1C="$PRIV_SUBNET_1C"
    export KMS_KEY_ARN="$KMS_KEY_ARN"
    export HARBOR_S3_BUCKET="$HARBOR_S3_BUCKET"
    EOF

    echo "Carregar em sessões futuras: source ~/eks-lab-vars.env"
    ```

=== "Windows (PowerShell)"

    ```powershell
    @"
    `$AWS_REGION        = "$AWS_REGION"
    `$AWS_ACCOUNT_ID    = "$AWS_ACCOUNT_ID"
    `$CLUSTER_NAME      = "$CLUSTER_NAME"
    `$VPC_ID            = "$VPC_ID"
    `$PUB_SUBNET_1A     = "$PUB_SUBNET_1A"
    `$PUB_SUBNET_1B     = "$PUB_SUBNET_1B"
    `$PUB_SUBNET_1C     = "$PUB_SUBNET_1C"
    `$PRIV_SUBNET_1A    = "$PRIV_SUBNET_1A"
    `$PRIV_SUBNET_1B    = "$PRIV_SUBNET_1B"
    `$PRIV_SUBNET_1C    = "$PRIV_SUBNET_1C"
    `$KMS_KEY_ARN       = "$KMS_KEY_ARN"
    `$HARBOR_S3_BUCKET  = "$HARBOR_S3_BUCKET"
    "@ | Set-Content ~\eks-lab-vars.ps1

    Write-Host "Carregar em sessões futuras: . ~\eks-lab-vars.ps1"
    ```

!!! warning "Não versione esse arquivo"
    O arquivo de variáveis contém IDs e ARNs específicos da sua conta. Não adicione ao Git.

## Checklist

- [ ] VPC criada com DNS habilitado
- [ ] 3 subnets públicas com tag `kubernetes.io/role/elb=1`
- [ ] 3 subnets privadas com tag `kubernetes.io/role/internal-elb=1`
- [ ] Internet Gateway criado e associado à VPC
- [ ] NAT Gateway criado na subnet pública 1a
- [ ] Route tables configuradas (pública → IGW, privada → NAT)
- [ ] Chave KMS criada para Vault auto-unseal
- [ ] Bucket S3 criado para Harbor com acesso público bloqueado

Próximo passo: [Criação do Cluster EKS](03-eks-cluster.md).
