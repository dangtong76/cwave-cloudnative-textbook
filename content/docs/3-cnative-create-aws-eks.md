---
title: "☁️ 03. Terraform 이용한 EKS 생성"
weight: 3
date: 2025-02-02
draft: false
---
---
테라폼은 인프라 생성과 관리에 있어서 가장 사랑받는 도구 입니다. 적어도 현재 까지는... 
강력한 경쟁자 : [OpenTofu](https://opentofu.org/), [Plumi](https://www.pulumi.com/)

최근에는 많은 회사들이 Git을 기반으로 코드로 인프라 플랫폼을 정의하고 관리하고 있으며, 이러한 기술 트랜드를 GitOPs 라고 부릅니다. 
테라폼은 퍼블릭 클라우드 뿐만아니라, Vmware, Nutanix 같은 온프라미스 환경에서도 잘 동작 합니다. 

<br><br>

## 1. 사전 준비
---
### - AWS 계정 생성 및 권한 할당
### - awscli 계정 설정
```bash
aws configure
aws configure list
cat ~/.aws/credentials
```

## 2. terraform 코드 작성
### - 디렉토리 생성
```bash
mkdir -p infra/cwave-aws-eks
```
### - 테라폼 코드 생성
1. variables.tf
```hcl
variable "aws_region" {
  description = "AWS Region"
  type        = string
  default     = "ap-northeast-2"  # 기본값 설정 (선택사항)
}

variable "vpc_name" {
  description = "name of vpc"
  type        = string
  default     = "cwave"
}

variable "availability_zones" {
  description = "Map of AZ suffixes to full AZ names"
  type        = map(string)
  default = {
    a = "ap-northeast-2a"
    b = "ap-northeast-2b"
    c = "ap-northeast-2c"
  }
}

variable "public_subnet_cidrs" {
  description = "Map of public subnet CIDRs by AZ suffix"
  type        = map(string)
  default = {
    a = "10.1.1.0/24"
    b = "10.1.2.0/24"
    c = "10.1.3.0/24"
  }
}

variable "private_subnet_cidrs" {
  description = "Map of private subnet CIDRs by AZ suffix"
  type        = map(string)
  default = {
    a = "10.1.4.0/24"
    b = "10.1.5.0/24"
    c = "10.1.6.0/24"
  }
}

variable "cluster_name" {
  description = "name of cluster"
  type        = string
  default     = "cwave"
}

variable "cluster_version" {
  description = "version of cluster"
  type        = string
  default     = "1.32"
}

variable "environment" {
  description = "Environment name (e.g., dev, stg, prd)"
  type        = string
  default     = "dev"  # 필요한 경우 기본값 설정
}

variable "eks_namespace_roles" {
  description = "Map of EKS namespace roles and their configurations"
  type = map(object({
    name                = string
    environment         = string
    additional_policies = list(string)
  }))
  default = {}
}
```
2. 000_provider.tf
```hcl
## AWS Provider 설정
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      managed_by = "terraform"
    }
  }
}
## 버전 설정
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.24"
    }
  }
}
```

3. 100_network.tf
```hcl

# Create a VPC

resource "aws_vpc" "vpc" {
  cidr_block           = "10.1.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "${var.cluster_name}-vpc"
  }
}

# Create Public & Private Subnet

resource "aws_subnet" "public" {
  for_each                = var.public_subnet_cidrs
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = each.value
  availability_zone       = var.availability_zones[each.key]
  map_public_ip_on_launch = true

  tags = {
    Name                                        = "public-subnet-${each.key}"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                    = "1"
  }
}

resource "aws_subnet" "private" {
  for_each                = var.private_subnet_cidrs
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = each.value
  availability_zone       = var.availability_zones[each.key]
  map_public_ip_on_launch = true
  tags = {
    Name                                        = "private-subnet-${each.key}"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"           = "1"
  }
}




# Create Internet gateway & Nat gateway

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id
  tags = {
    Name = "${var.cluster_name}-igw"
  }
}
resource "aws_nat_gateway" "nat" {
  for_each      = var.availability_zones
  subnet_id     = aws_subnet.public[each.key].id
  allocation_id = aws_eip.nat[each.key].id

  tags = {
    Name = "${var.cluster_name}-nat-${each.key}"
  }
}

resource "aws_eip" "nat" {
  for_each = var.availability_zones
  domain   = "vpc"
  lifecycle {
    create_before_destroy = true
  }
}

# Create Route Table & Route

resource "aws_route_table" "public-rtb" {
  vpc_id = aws_vpc.vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name = "${var.cluster_name}-public-rtb"
  }
}

resource "aws_route_table" "private-rtb" {
  for_each = var.private_subnet_cidrs
  vpc_id = aws_vpc.vpc.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat[each.key].id
  }
  tags = {
    Name = "${var.cluster_name}-private-rtb-${each.key}"
  }
}

resource "aws_route_table_association" "public-rtb" {
  for_each       = { for k, subnet in aws_subnet.public : k => subnet }
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public-rtb.id
}

resource "aws_route_table_association" "private-rtb" {
  for_each       = { for k, subnet in aws_subnet.private : k => subnet }
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public-rtb.id

}
```
4. 200_sg.tf
```hcl

# Create Security Group
resource "aws_security_group" "allow-ssh-sg" {
  name        = "allow-ssh"
  description = "allow ssh"
  vpc_id      = aws_vpc.vpc.id
}

resource "aws_security_group_rule" "allow-ssh" {
  from_port         = 22
  protocol          = "tcp"
  security_group_id = aws_security_group.allow-ssh-sg.id
  to_port           = 22
  type              = "ingress"
  description       = "ssh"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group" "public-sg" {
  name        = "public-sg"
  description = "allow all ports"
  vpc_id      = aws_vpc.vpc.id
}

resource "aws_security_group" "private-sg" {
  name        = "private-sg"
  description = "allow all ports"
  vpc_id      = aws_vpc.vpc.id
}

resource "aws_security_group_rule" "allow-all-ports" {
  from_port         = 0
  protocol          = "-1"
  security_group_id = aws_security_group.public-sg.id
  to_port           = 0
  type              = "ingress"
  description       = "all ports"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "allow-all-ports-egress" {
  from_port         = 0
  protocol          = "-1"
  security_group_id = aws_security_group.public-sg.id
  to_port           = 0
  type              = "egress"
  description       = "all ports"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group" "allow_nfs" {
  name        = "allow nfs for efs"
  description = "Allow NFS inbound traffic"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description = "NFS from VPC"
    from_port   = 2049
    to_port     = 2049
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.vpc.cidr_block]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
}
```
5. 300_eks.tf
```hcl
# Create eks cluster

data "aws_caller_identity" "current" {}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  version         = "~> 20.37"
  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true
  
  # EBS 관련 정책 추가
  iam_role_additional_policies = {
    AmazonEBSCSIDriverPolicy = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
    AmazonEC2FullAccess      = "arn:aws:iam::aws:policy/AmazonEC2FullAccess"
  }
  
  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      cluster_name = var.cluster_name
      most_recent = true
    }
    aws-ebs-csi-driver = {
      most_recent = true
      service_account_role_arn = module.ebs_csi_irsa.iam_role_arn
    }
    aws-efs-csi-driver = {
      most_recent = true
      service_account_role_arn = module.attach_efs_csi_role.iam_role_arn
    }
    eks-pod-identity-agent = {
      most_recent = true
    }
  }
  enable_cluster_creator_admin_permissions = true
  vpc_id                   = aws_vpc.vpc.id
  subnet_ids               = [for s in aws_subnet.private : s.id]

  # EKS Managed Node Group
  eks_managed_node_group_defaults = {
    instance_types = ["t3.medium"]
  }

  eks_managed_node_groups = {
    green = {
      min_size     = 2
      max_size     = 5
      desired_size = 2

      instance_types = ["t3.medium"]
      iam_role_additional_policies = {
        # AWS 관리형 정책 추가
        AmazonEBSCSIDriverPolicy = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
      }
    }
  }
}

module "ebs_csi_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.59"

  role_name = "${var.cluster_name}-ebs-csi-controller"

  attach_ebs_csi_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:ebs-csi-controller-sa"]
    }
  }
}

module "vpc_cni_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.59"

  role_name_prefix      = "VPC-CNI-IRSA"
  attach_vpc_cni_policy = true
  vpc_cni_enable_ipv4   = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-node"]
    }
    common = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-node"]
    }
  }
}

# EFS 파일시스템 및 마운트 타겟 생성

resource "aws_efs_file_system" "stw_node_efs" {
  creation_token = "efs-for-stw-node"
  
  tags = {
    Name        = "${var.cluster_name}-efs"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# 모든 가용영역에 EFS 마운트 타겟 생성
resource "aws_efs_mount_target" "stw_node_efs_mt" {
  for_each = var.availability_zones
  
  file_system_id  = aws_efs_file_system.stw_node_efs.id
  subnet_id       = aws_subnet.private[each.key].id
  security_groups = [aws_security_group.allow_nfs.id]
}
```
6. 400_iam.tf
```hcl
# EC2 인스턴스용 IAM 역할
# EC2 인스턴스가 AWS 서비스에 접근할 수 있도록 하는 IAM 역할을 생성합니다.

resource "aws_iam_role" "ec2_role" {
  # IAM 역할 이름
  name = "cwave_ec2_role"

  # 신뢰 관계 정책 (Trust Policy)
  # EC2 인스턴스가 이 역할을 수임(assume)할 수 있도록 허용
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"  # 역할 수임 권한
        Effect = "Allow"           # 허용
        Sid    = ""               # 정책 ID (선택사항)
        Principal = {
          Service = "ec2.amazonaws.com"  # EC2 서비스만 이 역할을 수임 가능
        }
      }
    ]
  })
}

# ECR PowerUser 정책 연결
resource "aws_iam_role_policy_attachment" "ecr_poweruser" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser"
}

# IAM Policy 설정

data "http" "iam_policy" {
  url = "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.2/docs/install/iam_policy.json"
}

resource "aws_iam_role_policy" "cwave-eks-controller" {
  name_prefix = "AWSLoadBalancerControllerIAMPolicy"
  role        = module.lb_controller_role.iam_role_name
  policy      = data.http.iam_policy.response_body
}

# EKS Namespace IAM Roles
resource "aws_iam_role" "eks_namespace_role" {
  for_each = var.eks_namespace_roles

  name = "eks-namespace-${each.value.name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Environment = each.value.environment
    ManagedBy   = "terraform"
  }
}

# Attach basic policies to namespace roles
resource "aws_iam_role_policy_attachment" "eks_namespace_policy" {
  for_each = var.eks_namespace_roles

  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_namespace_role[each.key].name
}

# Attach additional policies if specified
resource "aws_iam_role_policy_attachment" "eks_namespace_additional_policies" {
  for_each = {
    for policy in flatten([
      for ns_key, ns in var.eks_namespace_roles : [
        for policy in ns.additional_policies : {
          ns_key = ns_key
          policy = policy
        }
      ]
    ]) : "${policy.ns_key}-${policy.policy}" => policy
  }

  policy_arn = each.value.policy
  role       = aws_iam_role.eks_namespace_role[each.value.ns_key].name
}

# EFS role 설정

module "attach_efs_csi_role" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name             = "efs-csi"
  attach_efs_csi_policy = true

  oidc_providers = {
    ex = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:efs-csi-controller-sa"]
    }
  }
}
```

7. 500_helm.tf
```hcl

# EKS 클러스터 정보 조회
data "aws_eks_cluster" "cluster" {
   name = module.eks.cluster_name
   depends_on = [module.eks.cluster_name]
}

# EKS 클러스터 인증 정보 조회
data "aws_eks_cluster_auth" "cluster" {
   name = module.eks.cluster_name
   depends_on = [module.eks.cluster_name]
}

# Kubernetes Provider 설정
provider "kubernetes" {
  alias                  = "cwave-eks"
  host                   = data.aws_eks_cluster.cluster.endpoint
  # token                  = data.aws_eks_cluster_auth.cluster.token
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks",
      "get-token",
      "--cluster-name",
      var.cluster_name,
      "--region",
      var.aws_region
    ]
  }
}

/*
헬름차트
쿠버네티스 클러스터 추가 될때마다 alias 를 변경해서 추가해주기
*/
provider "helm" {
  alias = "cwave-eks-helm"

  kubernetes {
    host                   = module.eks.cluster_endpoint
    token                  = data.aws_eks_cluster_auth.eks_cluster_auth.token
    cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)

    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args = [
        "eks",
        "get-token",
        "--cluster-name",
        module.eks.cluster_name,
        "--region",
        var.aws_region,
      ]
    }
  }
}


# Helm release : alb

data "aws_eks_cluster_auth" "eks_cluster_auth" {
  name = var.cluster_name
}

# Load Balancer Controller를 위한 IAM Role 생성
module "lb_controller_role" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name = "eks-aws-lb-controller-role"

  attach_load_balancer_controller_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}

resource "helm_release" "eks_common_alb" {
  provider   = helm.cwave-eks-helm
  name       = "aws-load-balancer-controller"
  chart      = "aws-load-balancer-controller"
  version    = "1.6.2"
  repository = "https://aws.github.io/eks-charts"
  namespace  = "kube-system"

  dynamic "set" {
    for_each = {
      "clusterName"                                               = var.cluster_name
      "serviceAccount.create"                                     = "true"
      "serviceAccount.name"                                       = "aws-load-balancer-controller"
      "region"                                                    = var.aws_region
      "vpcId"                                                     = aws_vpc.vpc.id
      "image.repository"                                          = "602401143452.dkr.ecr.${var.aws_region}.amazonaws.com/amazon/aws-load-balancer-controller"
      "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn" = module.lb_controller_role.iam_role_arn
    }

    content {
      name  = set.key
      value = set.value
    }
  }
  depends_on = [
    module.eks,
    module.lb_controller_role
  ]
}
```

### - AWS EKS클러스터 생성하기
```bash
terraform init
terraform plan
terraform apply
```

### - kubectl 컨텍스트 업데이트
```bash
aws eks update-kubeconfig --region ap-northeast-2 --name cwave
```

```bash
kubectl get po
```