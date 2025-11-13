## 📜 AWS 리전별 인프라 모듈 (Region-Specific Infrastructure Module) - 세 가지 버전

AWS의 세 가지 리전(서울, 버지니아, 프랑크푸르트)에서 사용할 **VPC CIDR 블록**과 **리전 이름**이 명시적으로 포함된 독립적인 Terraform 모듈 코드입니다.

이 모듈들은 최상위 `main.tf`에서 해당 리전에 맞는 `provider` 별칭과 함께 호출되어야 합니다.

### 1\. 🇰🇷 서울 리전 인프라 모듈 (`modules/seoul_infra/main.tf`)

  * **VPC CIDR:** `10.10.0.0/16`
  * **리전 이름:** `Seoul`

<!-- end list -->

```terraform
# modules/seoul_infra/main.tf

# ------------------------------------------
# 1. 변수 정의 (서울 리전 설정)
# ------------------------------------------
variable "vpc_cidr" {
  default = "10.10.0.0/16" 
  type = string
}
variable "region_name" {
  default = "Seoul"
  type = string
}
variable "onprem_cidr" {
  description = "본사(On-premise) 네트워크의 CIDR 블록 (192.168.1.0/24)"
  type = string
}
variable "ssh_key_name" {
  description = "EC2 인스턴스에 접속할 SSH Key 이름"
  type = string
  default = "your-ssh-key" 
}

# ------------------------------------------
# 2. VPC 및 네트워크 구성
# ------------------------------------------
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = { Name = "${var.region_name}-VPC" }
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = cidrsubnet(var.vpc_cidr, 8, 0) 
  map_public_ip_on_launch = true 
  tags = { Name = "${var.region_name}-Public-Subnet" }
}

# (이후 IGW, Route Table, Security Group, EC2, NLB 정의는
# modules/regional_setup/main.tf의 내용과 동일하게 리소스 이름과 태그에 
# 'Seoul'이 반영된 채로 이어집니다.)
# ...
```

-----

### 2\. 🇺🇸 버지니아 리전 인프라 모듈 (`modules/virginia_infra/main.tf`)

  * **VPC CIDR:** `10.20.0.0/16`
  * **리전 이름:** `Virginia`

<!-- end list -->

```terraform
# modules/virginia_infra/main.tf

# ------------------------------------------
# 1. 변수 정의 (버지니아 리전 설정)
# ------------------------------------------
variable "vpc_cidr" {
  default = "10.20.0.0/16" 
  type = string
}
variable "region_name" {
  default = "Virginia"
  type = string
}
variable "onprem_cidr" {
  description = "본사(On-premise) 네트워크의 CIDR 블록 (192.168.1.0/24)"
  type = string
}
variable "ssh_key_name" {
  description = "EC2 인스턴스에 접속할 SSH Key 이름"
  type = string
  default = "your-ssh-key" 
}

# ------------------------------------------
# 2. VPC 및 네트워크 구성
# ------------------------------------------
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = { Name = "${var.region_name}-VPC" }
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = cidrsubnet(var.vpc_cidr, 8, 0) 
  map_public_ip_on_launch = true 
  tags = { Name = "${var.region_name}-Public-Subnet" }
}

# (이후 IGW, Route Table, Security Group, EC2, NLB 정의는
# modules/regional_setup/main.tf의 내용과 동일하게 리소스 이름과 태그에 
# 'Virginia'가 반영된 채로 이어집니다.)
# ...
```

-----

### 3\. 🇪🇺 프랑크푸르트 리전 인프라 모듈 (`modules/frankfurt_infra/main.tf`)

  * **VPC CIDR:** `10.30.0.0/16`
  * **리전 이름:** `Frankfurt`

<!-- end list -->

```terraform
# modules/frankfurt_infra/main.tf

# ------------------------------------------
# 1. 변수 정의 (프랑크푸르트 리전 설정)
# ------------------------------------------
variable "vpc_cidr" {
  default = "10.30.0.0/16" 
  type = string
}
variable "region_name" {
  default = "Frankfurt"
  type = string
}
variable "onprem_cidr" {
  description = "본사(On-premise) 네트워크의 CIDR 블록 (192.168.1.0/24)"
  type = string
}
variable "ssh_key_name" {
  description = "EC2 인스턴스에 접속할 SSH Key 이름"
  type = string
  default = "your-ssh-key" 
}

# ------------------------------------------
# 2. VPC 및 네트워크 구성
# ------------------------------------------
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = { Name = "${var.region_name}-VPC" }
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = cidrsubnet(var.vpc_cidr, 8, 0) 
  map_public_ip_on_launch = true 
  tags = { Name = "${var.region_name}-Public-Subnet" }
}

# (이후 IGW, Route Table, Security Group, EC2, NLB 정의는
# modules/regional_setup/main.tf의 내용과 동일하게 리소스 이름과 태그에 
# 'Frankfurt'가 반영된 채로 이어집니다.)
# ...
```

### 📌 요약 및 사용 방법

위의 세 모듈은 내부 로직이 완전히 동일하나, \*\*`vpc_cidr`\*\*와 **`region_name`** 변수를 **하드코딩**하여 리전별 독립성을 확보했습니다.

이제 최상위 \*\*`main.tf`\*\*에서는 이 모듈들을 리전별 Provider와 함께 호출합니다.

```terraform
# main.tf (최상위 파일)

module "seoul_infra" {
  source    = "./modules/seoul_infra" # 모듈 디렉터리가 변경됨
  providers = { aws = aws.seoul }
  onprem_cidr = local.onprem_cidr
}

module "virginia_infra" {
  source    = "./modules/virginia_infra" # 모듈 디렉터리가 변경됨
  providers = { aws = aws.virginia }
  onprem_cidr = local.onprem_cidr
}

module "frankfurt_infra" {
  source    = "./modules/frankfurt_infra" # 모듈 디렉터리가 변경됨
  providers = { aws = aws.frankfurt }
  onprem_cidr = local.onprem_cidr
}
```

이 방식을 사용하면 각 리전별 설정이 명확하게 분리되어 관리하기 쉽습니다.

**다음으로, 이 모듈들을 호출하는 최상위 `main.tf` 파일의 해당 부분을 업데이트해 드릴까요?**
