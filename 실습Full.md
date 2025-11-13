## 📚 하이브리드 본사-3개국 하이브리드 클라우드 인프라 구축 IAC 프로젝트 : hy3KUSE-pj

이 실습 프로젝트는 **Terraform**을 사용하여 **온프레미스 (VirtualBox) 환경의 마스터 노드**와 **AWS 클라우드 멀티-리전 (서울, 버지니아, 프랑크푸르트)의 워커 노드**를 통합하는 하이브리드 Kubernetes 인프라를 구축하는 것을 목표로 합니다.

-----

## 🎯 1. 프로젝트 목표 및 학습 목표

### 1.1. 구성 목표

  * **하이브리드 환경 구축:** 온프레미스 VM (K8s Master, MySQL DB)과 클라우드 EC2 (K8s Workers) 통합.
  * **멀티-리전 배포:** AWS의 서울, 버지니아, 프랑크푸르트 3개 리전에 동일 인프라를 모듈화하여 배포.
  * **글로벌 접근성:** AWS Route 53 GSLB (지연 시간 기반 라우팅)를 통한 글로벌 서비스 엔드포인트 구성.
  * **중앙 관리:** 서울 AWS에 중앙 집중식 S3 저장소 및 IAM 사용자/키를 생성하여 하이브리드 환경 간 공유 자원 마련.
  * **운영 안정성:** 코드 로직과 상태 파일 관리를 환경별로 엄격하게 분리하여 실무 안정성 확보.

### 1.2. 학습 목표

  * Terraform의 \*\*모듈(Modules)\*\*을 활용한 인프라 코드의 **재사용성** 극대화 방법 이해.
  * `local-exec` 프로비저너를 이용한 **온프레미스(로컬) 리소스** 연동 및 관리.
  * \*\*멀티-프로바이더(Multi-Provider) 및 별칭(Alias)\*\*을 활용한 멀티-리전 배포 방법 숙달.
  * Terraform **`environments`** 구조를 이용한 **개발/운영 환경 분리** 및 상태 파일(State) 관리 능력 습득.

-----

## 🏗️ 2. 프로젝트 디렉토리 구조 (hy3KUSE-pj)

프로젝트 루트 디렉토리: **`hy3KUSE-pj`**

```
hy3KUSE-pj/
├── main.tf                 # 1. 루트 구성 (Provider 정의, Locals, S3/IAM, Module 호출)
├── variables.tf            # 2. 전역 입력 변수 정의
├── outputs.tf              # 3. 최종 출력 값 정의
├── scripts/
│   └── create_vbox_vms.sh  # 4. VirtualBox VM 생성 쉘 스크립트
├── environments/           # 5. 환경별 구성 (dev/prod)
│   ├── dev/
│   │   ├── backend.tf      # 5-A. 개발 환경 상태 파일 백엔드 설정
│   │   └── main.tfvars     # 5-B. 개발 환경 전용 변수 값
│   └── prod/
│       ├── backend.tf      # 5-C. 운영 환경 상태 파일 백엔드 설정
│       └── main.tfvars     # 5-D. 운영 환경 전용 변수 값
└── modules/
    └── regional_setup/     # 6. 지역별 인프라 생성 모듈 (VPC, Worker EC2, NLB)
        ├── main.tf         # 6-A. 모듈 로직
        ├── variables.tf    # 6-B. 모듈 입력 변수
        └── outputs.tf      # 6-C. 모듈 출력 값
```

-----

## 💻 3. 파일별 코드 작성

### 3.1. `hy3KUSE-pj/main.tf`

```terraform
# -----------------------------------------------------------------------------
# 1. Terraform 설정 및 Provider 정의
# -----------------------------------------------------------------------------
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
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
  # variables.tf에서 정의된 입력 변수(var.)를 가져와 local. 변수로 재정의
  k8s_master_ip  = var.k8s_master_static_ip
  mysql_db_ip    = var.mysql_db_static_ip
  onprem_cidr    = var.onprem_network_cidr
  ansible_user   = var.ansible_user
}

# VirtualBox VM 생성 스크립트 실행 (온프레미스 리소스 연동)
resource "null_resource" "onprem_vbox_setup" {
  provisioner "local-exec" {
    # 환경 변수를 통해 IP를 쉘 스크립트에 전달
    command = "K8S_MASTER_IP=${local.k8s_master_ip} MYSQL_DB_IP=${local.mysql_db_ip} sh ./scripts/create_vbox_vms.sh"
  }
}

# K8s Master 서버 논리적 객체 정의 (Ansible/출력용)
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
  name          = "S3ReadWritePolicy-${var.s3_bucket_name}"
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
# 동일한 모듈을 각 리전별 프로바이더를 지정하여 3회 호출

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
# GSLB 구현을 위해 각 리전별 레코드 셋을 생성

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

### 3.2. `hy3KUSE-pj/variables.tf`

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
    frankfurt = "eu-central-2" # 리전 코드: eu-central-1 또는 eu-central-2 사용 가능
  }
}

variable "hosted_zone_id" {
  description = "Route 53 Hosted Zone ID"
  type        = string
  default     = "YOUR_HOSTED_ZONE_ID" # 실제 Zone ID로 변경 필요
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
  default     = "hybrid-k8s-shared-log-storage-2025-hy3kuse"
}
```

### 3.3. `hy3KUSE-pj/outputs.tf`

```terraform
# -----------------------------------------------------------------------------
# OUTPUTS (출력 값 정의)
# -----------------------------------------------------------------------------

# --- IAM Key (보안상 민감) ---

output "s3_access_key_id" {
  description = "S3 접근 IAM 사용자의 Access Key ID"
  value       = aws_iam_access_key.s3_key.id
}

output "s3_secret_access_key" {
  description = "S3 접근 IAM 사용자의 Secret Access Key"
  value       = aws_iam_access_key.s3_key.secret
  sensitive   = true # 보안을 위해 출력 시 숨김 처리
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
  description = "각 리전 Worker Node의 주요 정보 (Private IP, EC2 ID 등)"
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

### 3.4. `hy3KUSE-pj/scripts/create_vbox_vms.sh`

```bash
#!/bin/bash

# -----------------------------------------------------------------------------
# VirtualBox VM 생성 및 고정 IP 설정 스크립트 (실습 예제)
# -----------------------------------------------------------------------------

# Terraform에서 전달된 환경 변수 사용
K8S_MASTER_IP=$K8S_MASTER_IP
MYSQL_DB_IP=$MYSQL_DB_IP

VM_IMAGE="Ubuntu_22.04_Base"
NET_NAME="Hybrid-K8s-Net"

echo "=== VirtualBox VM 생성 시작 ==="
echo "K8s Master: $K8S_MASTER_IP, MySQL DB: $MYSQL_DB_IP"

# 1. K8s Master VM 생성 및 설정
VBoxManage createvm --name "k8s-master-onprem" --ostype "Ubuntu_64" --register
VBoxManage modifyvm "k8s-master-onprem" --cpus 2 --memory 4096
VBoxManage modifyvm "k8s-master-onprem" --nic1 natnetwork --natnet1 $NET_NAME
echo "K8s Master VM 생성 완료."

# 2. MySQL DB VM 생성 및 설정
VBoxManage createvm --name "mysql-db-server" --ostype "Ubuntu_64" --register
VBoxManage modifyvm "mysql-db-server" --cpus 1 --memory 2048
VBoxManage modifyvm "mysql-db-server" --nic1 natnetwork --natnet1 $NET_NAME
echo "MySQL DB VM 생성 완료."

# 3. VM 시작 (필요시 주석 해제)
# VBoxManage startvm "k8s-master-onprem" --type headless
# VBoxManage startvm "mysql-db-server" --type headless

echo "=== VM 인프라 구축 완료 ==="
```

### 3.5. `hy3KUSE-pj/environments/dev/backend.tf`

```terraform
# -----------------------------------------------------------------------------
# 5-A. 개발 환경 상태 파일 백엔드 설정
# -----------------------------------------------------------------------------
terraform {
  backend "s3" {
    bucket = "tf-state-bucket-hy3kuse-dev" # 개발 환경 전용 S3 버킷 이름 (고유해야 함)
    key    = "dev/terraform.tfstate"       # 개발 환경 상태 파일 경로
    region = "ap-northeast-2"              # 상태 파일을 저장할 AWS 리전
    # dynamodb_table = "terraform-lock-dev" # 상태 파일 잠금용 테이블 (실무 권장)
  }
}
```

### 3.6. `hy3KUSE-pj/environments/dev/main.tfvars`

```terraform
# -----------------------------------------------------------------------------
# 5-B. 개발 환경 전용 변수 값
# -----------------------------------------------------------------------------
# 이 파일의 값은 루트 variables.tf의 기본값을 덮어씁니다.

s3_bucket_name = "hybrid-k8s-shared-log-storage-dev"
# dev 환경에서는 저렴한 인스턴스 타입 등 필요시 추가 변수 설정 가능
```

### 3.7. `hy3KUSE-pj/environments/prod/backend.tf`

```terraform
# -----------------------------------------------------------------------------
# 5-C. 운영 환경 상태 파일 백엔드 설정
# -----------------------------------------------------------------------------
terraform {
  backend "s3" {
    bucket = "tf-state-bucket-hy3kuse-prod" # 운영 환경 전용 S3 버킷 이름
    key    = "prod/terraform.tfstate"       # 운영 환경 상태 파일 경로
    region = "ap-northeast-2"
    # dynamodb_table = "terraform-lock-prod" # 상태 파일 잠금용 테이블 (운영 필수)
  }
}
```

### 3.8. `hy3KUSE-pj/environments/prod/main.tfvars`

```terraform
# -----------------------------------------------------------------------------
# 5-D. 운영 환경 전용 변수 값
# -----------------------------------------------------------------------------
# 운영 환경에서는 보안과 안정성이 최우선입니다.

s3_bucket_name       = "hybrid-k8s-shared-log-storage-prod"
# 실제 Route 53 호스팅 영역 ID 사용
hosted_zone_id       = "PROD_HOSTED_ZONE_ID_12345" 
# 운영 환경에 맞는 Worker Node 인스턴스 타입 등 추가 설정 가능
```

### 3.9. `hy3KUSE-pj/modules/regional_setup/main.tf`

```terraform
# -----------------------------------------------------------------------------
# 6-A. MODULE: Regional Setup (VPC, Subnet, EC2 Worker, NLB) - 핵심 로직
# -----------------------------------------------------------------------------

# 1. VPC 생성
resource "aws_vpc" "region_vpc" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = "Hybrid-K8s-${var.region_name}-VPC"
  }
}

# 2. 가용 영역(AZ) 목록 조회 (동적 AZ 사용)
data "aws_availability_zones" "available" {
  state = "available"
}

# 3. Public Subnet 생성 (첫 번째 AZ)
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.region_vpc.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, 1) # 10.x.1.0/24
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
  # security_groups = [aws_security_group.worker_sg.id] # SG 필요
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
}
```

### 3.10. `hy3KUSE-pj/modules/regional_setup/variables.tf`

```terraform
# -----------------------------------------------------------------------------
# 6-B. MODULE INPUT VARIABLES (지역별 모듈 입력 변수)
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
  description = "온프레미스 네트워크 CIDR (VPN/Direct Connect 등 연동 시 사용)"
  type        = string
}

variable "ami_id" {
  description = "Worker Node에 사용할 AMI ID"
  type        = string
  default     = "ami-0abcdef1234567890" # 리전별/OS별 적절한 AMI ID 필요
}

variable "instance_type" {
  description = "Worker Node에 사용할 인스턴스 타입"
  type        = string
  default     = "t3.medium"
}
```

### 3.11. `hy3KUSE-pj/modules/regional_setup/outputs.tf`

```terraform
# -----------------------------------------------------------------------------
# 6-C. MODULE OUTPUTS (지역별 모듈 출력 값)
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

-----

## ⚙️ 4. 실습 구현 절차 및 실행 명령

### 4.1. 초기 환경 설정

1.  **디렉토리 생성:** 프로젝트 루트 디렉토리와 하위 구조를 생성합니다.
    ```bash
    mkdir -p hy3KUSE-pj/{scripts,environments/dev,environments/prod,modules/regional_setup}
    # 위에서 작성된 모든 파일을 해당 경로에 저장합니다.
    ```
2.  **AWS 자격 증명 설정:** Terraform이 AWS에 접근할 수 있도록 환경 변수를 설정합니다.
    ```bash
    export AWS_ACCESS_KEY_ID="YOUR_AWS_ACCESS_KEY"
    export AWS_SECRET_ACCESS_KEY="YOUR_AWS_SECRET_KEY"
    # 또는 AWS CLI configure 명령 사용
    ```

### 4.2. 보안 키 값 처리 (IAM Key)

IAM Access Key와 Secret Key는 **민감한 정보**이므로 Terraform State 파일에 저장되지만, 출력될 때도 노출됩니다.

  * `outputs.tf`에서 `s3_secret_access_key` 필드에 \*\*`sensitive = true`\*\*를 설정하여 **`terraform apply`** 완료 시 화면에 노출되지 않도록 처리했습니다.
  * **실무 팁:** 생성된 키는 `terraform output` 명령으로 조회 후 Ansible 등 외부 시스템에 전달해야 합니다. 보안을 위해 이 값을 **AWS Secrets Manager**나 **HashiCorp Vault**에 저장하고 사용하는 것을 강력히 권장합니다.

### 4.3. 개발 환경 배포 (dev)

1.  **디렉토리 이동:** 개발 환경 디렉토리로 이동합니다.
    ```bash
    cd hy3KUSE-pj/environments/dev
    ```
2.  **초기화 (Init):** 루트 디렉토리의 코드와 `backend.tf`를 로드합니다.
    ```bash
    terraform init --reconfigure # 백엔드가 분리되었으므로 --reconfigure 사용
    ```
3.  **계획 확인 (Plan):** 실행 전에 어떤 리소스가 생성될지 확인합니다.
    ```bash
    terraform plan -var-file=main.tfvars
    ```
4.  **배포 실행 (Apply):** 인프라를 실제로 구축합니다.
    ```bash
    terraform apply -var-file=main.tfvars
    # 확인을 위해 -auto-approve 플래그는 사용하지 않습니다.
    ```

### 4.4. 운영 환경 배포 (prod)

**주의:** 운영 환경 배포 전에는 `main.tfvars`의 변수 값을 반드시 검토해야 합니다.

1.  **디렉토리 이동:** 운영 환경 디렉토리로 이동합니다.
    ```bash
    cd ../prod
    ```
2.  **초기화 (Init):** 운영 환경의 백엔드 설정(`prod/backend.tf`)을 로드합니다.
    ```bash
    terraform init --reconfigure
    ```
3.  **배포 실행 (Apply):** 운영 인프라를 구축합니다.
    ```bash
    terraform apply -var-file=main.tfvars
    ```
    이 실행은 **개발 환경의 상태 파일과 분리**되어 운영 환경만의 독립적인 상태 파일을 S3에 생성합니다.
