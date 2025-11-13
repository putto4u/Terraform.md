## 📁 프로젝트 디렉토리 구조 및 파일별 코드 (hy3KUSE-pj 원본)

요청하신 대로, 프로젝트 루트 디렉토리 이름을 \*\*`hy3KUSE-pj`\*\*로 명시하고, 하이브리드 Kubernetes 실습을 위한 모든 Terraform 파일과 쉘 스크립트를 **최초 원본 파일 형태**로 분리하여 다시 작성해 드립니다.

이 구조는 **Terraform의 모범 사례**인 \*\*모듈화(Modules)\*\*를 적용하여 복잡한 멀티-리전 인프라 관리를 효율적으로 수행합니다.

-----

## I. 프로젝트 디렉토리 구조

```
hy3KUSE-pj/
├── main.tf                 # 1. 루트 구성 파일 (주요 로직, Provider, S3/IAM, Route53, Module 호출)
├── variables.tf            # 2. 전역 입력 변수 정의
├── outputs.tf              # 3. 최종 출력 값 정의
├── scripts/
│   └── create_vbox_vms.sh  # 4. VirtualBox VM 생성 스크립트
└── modules/
    └── regional_setup/     # 5. 지역별 인프라 생성 모듈
        ├── main.tf         # 5-A. 모듈 로직 (VPC, Worker EC2, NLB 등)
        ├── variables.tf    # 5-B. 모듈 입력 변수
        └── outputs.tf      # 5-C. 모듈 출력 값
```

-----

## II. 파일별 코드

### 1\. `hy3KUSE-pj/main.tf`

```terraform
# -----------------------------------------------------------------------------
# 1. Terraform 설정 및 Provider 정의
# -----------------------------------------------------------------------------
terraform {
  # Terraform 코어 버전 지정 (안정성을 위해 권장)
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    # 로컬 명령 실행, 트리거 등 로직 처리를 위한 Provider
    null = {
      source = "hashicorp/null"
      version = "~> 3.0"
    }
  }
}

# AWS Provider 정의: 멀티-리전 배포를 위해 각 리전별 별칭(alias) 지정
provider "aws" {
  alias  = "seoul"
  region = var.aws_regions["seoul"]
}
provider "aws" {
  alias  = "virginia"
  region = var.aws_regions["virginia"]
}
provider "aws" {
  alias  = "frankfurt"
  region = var.aws_regions["frankfurt"]
}

# -----------------------------------------------------------------------------
# 2. On-Premise 환경 변수 및 VirtualBox VM 정의
# -----------------------------------------------------------------------------
locals {
  # variables.tf에서 정의된 입력 변수(var.)를 가져와 local. 변수로 재정의하여 사용
  k8s_master_ip  = var.k8s_master_static_ip
  mysql_db_ip    = var.mysql_db_static_ip
  onprem_cidr    = var.onprem_network_cidr
  ansible_user   = var.ansible_user
}

# VirtualBox VM 생성 스크립트 실행 (K8s Master 및 MySQL DB 서버 생성)
resource "null_resource" "onprem_vbox_setup" {
  # VBoxManage를 사용하는 쉘 스크립트를 로컬에서 실행
  provisioner "local-exec" {
    command = "K8S_MASTER_IP=${local.k8s_master_ip} MYSQL_DB_IP=${local.mysql_db_ip} sh ./scripts/create_vbox_vms.sh"
  }
}

# K8s Master 서버 논리적 객체 정의 (Ansible 인벤토리에서 참조할 IP/사용자 정보 저장)
resource "null_resource" "k8s_master_onprem" {
  depends_on = [null_resource.onprem_vbox_setup]
  triggers = {
    ip_address = local.k8s_master_ip
    user = local.ansible_user
  }
}

# MySQL DB 서버 논리적 객체 정의
resource "null_resource" "mysql_db_server" {
  depends_on = [null_resource.onprem_vbox_setup]
  triggers = {
    ip_address = local.mysql_db_ip
    user = local.ansible_user
  }
}

# -----------------------------------------------------------------------------
# 3. AWS 서울 S3 버킷 및 IAM 자원 정의 (중앙 공유 스토리지)
# -----------------------------------------------------------------------------

# S3 버킷 생성 (서울 리전 - 중앙 집중식 로그/파일 저장소)
resource "aws_s3_bucket" "shared_storage" {
  provider = aws.seoul
  bucket = var.s3_bucket_name 
  acl    = "private"
  tags = {
    Name = "Hybrid-K8s-Shared-Storage"
  }
}

# S3 접근을 위한 IAM 사용자 생성
resource "aws_iam_user" "s3_access_user" {
  name = "k8s-s3-file-access"
}

# S3 Read/Write 접근 정책 정의
resource "aws_iam_policy" "s3_read_write" {
  name          = "S3ReadWritePolicy-${aws_s3_bucket.shared_storage.id}"
  description   = "Allows read and write access to the shared S3 bucket"
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = ["s3:PutObject", "s3:GetObject", "s3:ListBucket", "s3:DeleteObject", "s3:ListBucketVersions"],
        Resource = [
          aws_s3_bucket.shared_storage.arn,
          "${aws_s3_bucket.shared_storage.arn}/*", 
        ],
      },
    ],
  })
}

# IAM 사용자에게 정책 연결
resource "aws_iam_user_policy_attachment" "s3_attach" {
  user         = aws_iam_user.s3_access_user.name
  policy_arn = aws_iam_policy.s3_read_write.arn
}

# S3 접근을 위한 Access Key 및 Secret Key 생성 (Ansible에서 사용)
resource "aws_iam_access_key" "s3_key" {
  user = aws_iam_user.s3_access_user.name
}

# -----------------------------------------------------------------------------
# 4. AWS Regional Worker Infrastructure Module 호출
# -----------------------------------------------------------------------------
# 각 리전에 VPC, Subnet, EC2 Worker 인스턴스, NLB를 생성하는 모듈을 호출합니다.

module "seoul_infra" {
  source    = "./modules/regional_setup"
  providers = { aws = aws.seoul } 
  region_name = "Seoul"
  vpc_cidr    = "10.10.0.0/16"
  onprem_cidr = local.onprem_cidr
}

module "virginia_infra" {
  source    = "./modules/regional_setup"
  providers = { aws = aws.virginia }
  region_name = "Virginia"
  vpc_cidr    = "10.20.0.0/16"
  onprem_cidr = local.onprem_cidr
}

module "frankfurt_infra" {
  source    = "./modules/regional_setup"
  providers = { aws = aws.frankfurt }
  region_name = "Frankfurt"
  vpc_cidr    = "10.30.0.0/16"
  onprem_cidr = local.onprem_cidr
}

# -----------------------------------------------------------------------------
# 5. 글로벌 로드 밸런싱 (Route 53 - 지연 시간 기반 라우팅) 정의
# -----------------------------------------------------------------------------
# Route 53 LBR을 사용하여 사용자에게 가장 가까운 리전으로 트래픽을 분산합니다.

resource "aws_route53_record" "global_k8s_access_seoul" {
  zone_id = var.hosted_zone_id
  name    = var.global_domain_name 
  type    = "A"
  ttl     = 60

  alias {
    name                 = module.seoul_infra.nlb_dns_name 
    zone_id              = module.seoul_infra.nlb_zone_id
    evaluate_target_health = true 
  }
  latency_routing_policy {
    region = var.aws_regions["seoul"]
  }
  set_identifier = "seoul-traffic"
}

resource "aws_route53_record" "global_k8s_access_virginia" {
  zone_id = aws_route53_record.global_k8s_access_seoul.zone_id
  name    = aws_route53_record.global_k8s_access_seoul.name
  type    = "A"
  ttl     = 60

  alias {
    name                 = module.virginia_infra.nlb_dns_name 
    zone_id              = module.virginia_infra.nlb_zone_id
    evaluate_target_health = true
  }
  latency_routing_policy {
    region = var.aws_regions["virginia"]
  }
  set_identifier = "virginia-traffic"
}

resource "aws_route53_record" "global_k8s_access_frankfurt" {
  zone_id = aws_route53_record.global_k8s_access_seoul.zone_id
  name    = aws_route53_record.global_k8s_access_seoul.name
  type    = "A"
  ttl     = 60

  alias {
    name                 = module.frankfurt_infra.nlb_dns_name 
    zone_id              = module.frankfurt_infra.nlb_zone_id
    evaluate_target_health = true
  }
  latency_routing_policy {
    region = var.aws_regions["frankfurt"]
  }
  set_identifier = "frankfurt-traffic"
}
```

-----

### 2\. `hy3KUSE-pj/variables.tf`

```terraform
# -----------------------------------------------------------------------------
# GLOBAL INPUT VARIABLES (전역 입력 변수 정의)
# -----------------------------------------------------------------------------

# --- AWS 리전 및 도메인 설정 ---

variable "aws_regions" {
  description = "사용할 AWS 리전 및 별칭 정의"
  type = map(string)
  default = {
    seoul     = "ap-northeast-2"
    virginia  = "us-east-1"
    frankfurt = "eu-central-2" 
  }
}

variable "hosted_zone_id" {
  description = "Route 53 Hosted Zone ID"
  type        = string
  default     = "YOUR_HOSTED_ZONE_ID" 
}

variable "global_domain_name" {
  description = "Kubernetes 서비스에 접속할 최종 도메인 이름"
  type        = string
  default     = "app.yourcompany.com"
}


# --- VM 및 네트워크 설정 ---

variable "ansible_user" {
  description = "VM에 접속하여 Ansible 작업을 수행할 사용자 계정 이름."
  type        = string
  default     = "ubuntu"
}

variable "onprem_network_cidr" {
  description = "VirtualBox NAT Network에서 사용할 CIDR 대역."
  type        = string
  default     = "192.168.1.0/24"
}

variable "k8s_master_static_ip" {
  description = "Kubernetes Master VM에 할당할 고정 IP 주소."
  type        = string
  default     = "192.168.1.100"
}

variable "mysql_db_static_ip" {
  description = "MySQL DB VM에 할당할 고정 IP 주소."
  type        = string
  default     = "192.168.1.101"
}

# --- S3 설정 ---

variable "s3_bucket_name" {
  description = "공유 로그 및 파일 저장을 위한 S3 버킷 이름 (글로벌 고유해야 함)"
  type        = string
  default     = "hybrid-k8s-shared-log-storage-2025-yourname"
}
```

-----

### 3\. `hy3KUSE-pj/outputs.tf`

```terraform
# -----------------------------------------------------------------------------
# OUTPUTS (출력 값 정의)
# -----------------------------------------------------------------------------

# --- IAM Key ---

output "s3_access_key_id" {
  description = "S3 접근 IAM 사용자의 Access Key ID"
  value       = aws_iam_access_key.s3_key.id
}

output "s3_secret_access_key" {
  description = "S3 접근 IAM 사용자의 Secret Access Key"
  value       = aws_iam_access_key.s3_key.secret
  sensitive   = true 
}

# --- On-premise VM 정보 ---

output "onprem_k8s_master_ip" {
  description = "On-premise K8s Master의 IP 주소"
  value       = local.k8s_master_ip
}

output "onprem_ansible_user" {
  description = "On-premise VM 접속 사용자 이름"
  value       = local.ansible_user
}

# --- AWS 리전 Worker 정보 및 GSLB ---

output "regional_worker_info" {
  description = "각 리전 Worker Node의 주요 정보"
  value       = {
    seoul     = module.seoul_infra.worker_node_info
    virginia  = module.virginia_infra.worker_node_info
    frankfurt = module.frankfurt_infra.worker_node_info
  }
}

output "global_access_endpoint" {
  description = "글로벌 접근 도메인 (Route 53 GSLB)"
  value       = aws_route53_record.global_k8s_access_seoul.name
}
```

-----

### 4\. `hy3KUSE-pj/scripts/create_vbox_vms.sh`

```bash
#!/bin/bash

# -----------------------------------------------------------------------------
# VirtualBox VM 생성 및 고정 IP 설정 스크립트 (예시)
# Terraform의 local-exec 프로비저너에 의해 호출됩니다.
# -----------------------------------------------------------------------------

# Terraform에서 전달된 환경 변수 사용
K8S_MASTER_IP=$K8S_MASTER_IP
MYSQL_DB_IP=$MYSQL_DB_IP

VM_IMAGE="Ubuntu_22.04_Base"
NET_NAME="Hybrid-K8s-Net"

echo "VM 생성 시작: K8s Master ($K8S_MASTER_IP), MySQL DB ($MYSQL_DB_IP)"

# 1. K8s Master VM 생성 (예시 로직)
VBoxManage createvm --name "k8s-master-onprem" --ostype "Ubuntu_64" --register
VBoxManage modifyvm "k8s-master-onprem" --cpus 2 --memory 4096
VBoxManage modifyvm "k8s-master-onprem" --nic1 natnetwork --natnet1 $NET_NAME

# 2. MySQL DB VM 생성 (예시 로직)
VBoxManage createvm --name "mysql-db-server" --ostype "Ubuntu_64" --register
VBoxManage modifyvm "mysql-db-server" --cpus 1 --memory 2048
VBoxManage modifyvm "mysql-db-server" --nic1 natnetwork --natnet1 $NET_NAME

# 3. VM 시작 (필요시)
# VBoxManage startvm "k8s-master-onprem" --type headless
# VBoxManage startvm "mysql-db-server" --type headless

echo "VM 생성 및 설정 완료."
```

-----

### 5-A. `hy3KUSE-pj/modules/regional_setup/main.tf`

```terraform
# -----------------------------------------------------------------------------
# MODULE: Regional Setup (VPC, Subnet, EC2 Worker, NLB) - 핵심 로직
# -----------------------------------------------------------------------------

# 1. VPC 생성
resource "aws_vpc" "region_vpc" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = "Hybrid-K8s-${var.region_name}-VPC"
  }
}

# 2. 가용 영역(AZ) 목록 조회
data "aws_availability_zones" "available" {
  state = "available"
}

# 3. Public Subnet 생성 (첫 번째 AZ)
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.region_vpc.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, 1) 
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[0]
  tags = {
    Name = "${var.region_name}-Public-Subnet-A"
  }
}

# 4. EC2 Worker Node 생성
resource "aws_instance" "k8s_worker" {
  ami           = var.ami_id 
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id
  # ... 키 페어, 보안 그룹 등 필수 구성 요소는 생략됨
  tags = {
    Name = "K8s-Worker-${var.region_name}-01"
  }
}

# 5. Network Load Balancer (NLB) 생성
resource "aws_lb" "regional_nlb" {
  name               = "k8s-${var.region_name}-nlb"
  load_balancer_type = "network"
  subnets            = [aws_subnet.public.id]
  internal           = false
  tags = {
    Name = "K8s-${var.region_name}-NLB"
  }
  # ... Target Group 및 Listener 정의는 생략됨
}
```

-----

### 5-B. `hy3KUSE-pj/modules/regional_setup/variables.tf`

```terraform
# -----------------------------------------------------------------------------
# MODULE INPUT VARIABLES (지역별 모듈 입력 변수)
# -----------------------------------------------------------------------------

variable "region_name" {
  description = "현재 모듈이 배포되는 AWS 리전의 이름"
  type        = string
}

variable "vpc_cidr" {
  description = "해당 AWS 리전 VPC에 할당할 CIDR 블록"
  type        = string
}

variable "onprem_cidr" {
  description = "온프레미스 네트워크 CIDR (VPN 연동 시 필요)"
  type        = string
}

variable "ami_id" {
  description = "Worker Node에 사용할 AMI ID"
  type        = string
  default     = "ami-0abcdef1234567890" 
}

variable "instance_type" {
  description = "Worker Node에 사용할 인스턴스 타입"
  type        = string
  default     = "t3.medium"
}
```

-----

### 5-C. `hy3KUSE-pj/modules/regional_setup/outputs.tf`

```terraform
# -----------------------------------------------------------------------------
# MODULE OUTPUTS (지역별 모듈 출력 값)
# -----------------------------------------------------------------------------

output "nlb_dns_name" {
  description = "지역별 Network Load Balancer의 DNS 이름"
  value       = aws_lb.regional_nlb.dns_name
}

output "nlb_zone_id" {
  description = "지역별 Network Load Balancer의 Route 53 Zone ID"
  value       = aws_lb.regional_nlb.zone_id
}

output "worker_node_info" {
  description = "지역 Worker EC2 인스턴스의 주요 정보"
  value       = {
    id         = aws_instance.k8s_worker.id
    private_ip = aws_instance.k8s_worker.private_ip
  }
}
```
