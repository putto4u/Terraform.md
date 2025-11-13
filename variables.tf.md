 **`main.tf`** 파일에서 사용하는 변수들을 정의하기 위한 **`variables.tf`** 파일을 작성합니다.

`locals` 등의 블록 변수들은 `main.tf` 파일 내에서 고정 값으로 사용되지만, `ansible_user`와 같이 환경에 따라 변경될 수 있는 값을 \*\*입력 변수(`variable`)\*\*로 정의하여 유연성을 높이는 것이 좋습니다.

-----

## 🛠️ `variables.tf` 파일 내용



```terraform
# -----------------------------------------------------------------------------
# INPUT VARIABLES (입력 변수 정의)
# -----------------------------------------------------------------------------

# --- VM 접속 및 환경 관련 변수 ---

variable "ansible_user" {
  description = "VM에 접속하여 Ansible 작업을 수행할 사용자 계정 이름."
  type        = string
  default     = "ubuntu"
}

variable "onprem_cidr" {
  description = "가상 환경(VirtualBox)에서 사용할 네트워크 대역(CIDR)."
  type        = string
  default     = "192.168.1.0/24"
  # 참고: VirtualBox의 NAT Network 설정을 참조하여 필요시 수정합니다.
}


# --- 서비스 IP 주소 변수 ---

variable "k8s_master_ip" {
  description = "Kubernetes Master 서버에 할당할 고정 IP 주소."
  type        = string
  default     = "192.168.1.100"
}

variable "mysql_db_ip" {
  description = "MySQL DB 서버에 할당할 고정 IP 주소."
  type        = string
  default     = "192.168.1.101"
}

# --- 기타 설정 변수 (옵션) ---

/*
variable "vm_image_name" {
  description = "VM 생성에 사용할 VirtualBox 이미지 이름."
  type        = string
  default     = "Ubuntu_22.04"
}

variable "vm_cpu_count" {
  description = "VM에 할당할 가상 CPU 개수."
  type        = number
  default     = 2
}
*/
```

-----

## 💡 `main.tf` 및 `locals` 활용 팁

위에서 정의된 변수들을 기존의 `locals` 블록과 결합하여 사용하는 방법을 고려하여 \*\*`main.tf`\*\*를 다음과 같이 구조화하면 더욱 좋습니다.

### 1\. `main.tf`에서 `var.`와 `local.` 활용

1.  **입력 변수 사용:** `variables.tf`에서 정의된 값은 `main.tf`에서 \*\*`var.ansible_user`\*\*와 같이 **`var.`** 접두사로 참조됩니다.
2.  **지역 변수로 통합:** `locals` 블록을 사용하여 입력 변수와 다른 상수를 한데 모아 관리합니다.

<!-- end list -->

```terraform
# (main.tf 내용 예시)

# 1. 'variables.tf'에서 정의된 입력 변수들은 'var.'로 가져옵니다.
# 2. 고정된 값과 입력 변수를 결합하여 'locals'를 재정의합니다.
locals {
  k8s_master_ip  = var.k8s_master_ip
  mysql_db_ip    = var.mysql_db_ip
  onprem_cidr    = var.onprem_cidr
  ansible_user   = var.ansible_user # 'var.'로 가져온 값을 'local.'로 재정의하여 사용합니다.
}

# ...

resource "null_resource" "onprem_vbox_setup" {
  provisioner "local-exec" {
    # 이제 스크립트에 전달할 때도 'local.'을 사용하여 일관성을 유지합니다.
    command = "K8S_MASTER_IP=${local.k8s_master_ip} MYSQL_DB_IP=${local.mysql_db_ip} sh ./scripts/create_vbox_vms.sh"
  }
}
```

이 구조를 사용하면 IP 주소나 사용자 이름 등을 프로젝트 외부에서 쉽게 변경할 수 있어, 실습 환경을 재구성하거나 다른 환경에 배포할 때 유연성이 매우 높아집니다.
