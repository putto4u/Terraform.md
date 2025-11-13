## 🖥️ VirtualBox VM 생성 및 구성 쉘 스크립트 (`scripts/create_vbox_vms.sh`)

요청하신 대로, **VirtualBox**의 **`VBoxManage`** 명령어를 사용하여 K8s Master와 MySQL DB 서버 VM을 생성하고, **고정 IP 주소를 할당**하는 쉘 스크립트를 작성합니다.

이 스크립트는 **Ubuntu Server 22.04 LTS (Jammy Jellyfish)** 이미지의 **OVF/OVA 템플릿 파일**이 `$VBOX_IMAGE_PATH`에 존재한다고 가정하며, \*\*`Host-Only Adapter`\*\*를 사용하여 호스트와 VM 간 통신 및 고정 IP를 보장합니다.

### ⚠️ 필수 전제 조건 및 준비 사항

1.  **Host-Only 네트워크 생성:** VirtualBox 설정에서 \*\*`vboxnet0`\*\*이라는 이름의 **Host-Only Adapter**가 **`192.168.1.1`** 게이트웨이와 **`192.168.1.0/24`** 대역으로 이미 생성되어 있어야 합니다.
2.  **VM 템플릿 파일:** 사용할 Ubuntu VM의 `.ovf` 또는 `.ova` 파일 경로를 `VBOX_IMAGE_PATH` 변수에 지정해야 합니다.

### 📜 `scripts/create_vbox_vms.sh`

```bash
#!/bin/bash
# ----------------------------------------------------------------------
# VirtualBox VM 생성 및 고정 IP 설정 스크립트
# 사용법: ./scripts/create_vbox_vms.sh <K8S_MASTER_IP> <MYSQL_DB_IP>
# 예: ./scripts/create_vbox_vms.sh 192.168.1.100 192.168.1.101
# ----------------------------------------------------------------------

# --- 변수 설정 ---
MASTER_IP="$1"
DB_IP="$2"
HOST_ONLY_IFACE="vboxnet0"
NETMASK="255.255.255.0"
GATEWAY="192.168.1.1" # Host-Only 어댑터의 IP
VBOX_IMAGE_PATH="/path/to/your/Ubuntu-Server-22.04.ova" # <-- 실제 템플릿 경로로 수정 필수
MASTER_NAME="K8s-Master-OnPrem"
DB_NAME="MySQL-DB-Server"
VM_CPUS=2
VM_RAM=4096 # MB

if [ -z "$MASTER_IP" ] || [ -z "$DB_IP" ]; then
    echo "오류: K8s Master IP와 MySQL DB IP를 인수로 제공해야 합니다."
    echo "사용법: $0 <K8S_MASTER_IP> <MYSQL_DB_IP>"
    exit 1
fi

echo "--- 1. K8s Master VM 생성 ($MASTER_NAME, IP: $MASTER_IP) ---"

# 1.1. VM Import (기존 템플릿 사용)
VBoxManage import "$VBOX_IMAGE_PATH" --vsys 0 --vmname "$MASTER_NAME" --memory "$VM_RAM" --cpus "$VM_CPUS" --options "skipovfversion"
if [ $? -ne 0 ]; then
    echo "오류: VM Import에 실패했습니다. 이미지 경로를 확인하십시오."
    exit 1
fi

# 1.2. 네트워크 어댑터 설정 (Host-Only Adapter 사용)
VBoxManage modifyvm "$MASTER_NAME" --nic1 hostonly --hostonlyadapter1 "$HOST_ONLY_IFACE"
VBoxManage modifyvm "$MASTER_NAME" --cableconnected1 on

# 1.3. VM 시작 (최초 시작 시 OS 내부 설정이 필요한 경우)
VBoxManage startvm "$MASTER_NAME" --type headless

# ----------------------------------------------------------------------

echo "--- 2. MySQL DB VM 생성 ($DB_NAME, IP: $DB_IP) ---"

# 2.1. VM Import (두 번째 VM 생성을 위해 다른 이름으로 다시 Import)
VBoxManage import "$VBOX_IMAGE_PATH" --vsys 0 --vmname "$DB_NAME" --memory 2048 --cpus 1 --options "skipovfversion" # DB는 자원 소폭 축소 예시

# 2.2. 네트워크 어댑터 설정 (Host-Only Adapter 사용)
VBoxManage modifyvm "$DB_NAME" --nic1 hostonly --hostonlyadapter1 "$HOST_ONLY_IFACE"
VBoxManage modifyvm "$DB_NAME" --cableconnected1 on

# 2.3. VM 시작
VBoxManage startvm "$DB_NAME" --type headless

# ----------------------------------------------------------------------

echo "--- 3. OS 내부 고정 IP 설정 (Ansible을 위한 준비) ---"
# VBoxManage로 OS 내부 네트워크 설정은 불가능합니다.
# VM 시작 후, Ansible을 사용하여 SSH 접속 후 고정 IP 설정을 완료해야 합니다.

cat << EOF
#########################################################################
#                                                                       #
# [중요 안내] VM 생성 완료.                                             #
# 실제 고정 IP 설정 (192.168.1.100, 192.168.1.101)은                  #
# VM 내부 OS (Ubuntu)의 netplan 설정을 Ansible로 완료해야 합니다.       #
#                                                                       #
# 이 스크립트는 VM이 Host-Only 네트워크에 연결된 것만 보장합니다.       #
# Ansible playbook 실행 전, SSH를 통해 임시 DHCP IP로 접속 가능해야 합니다. #
#                                                                       #
#########################################################################
EOF

# ----------------------------------------------------------------------
```

### 📝 실전 팁: VM 내부 IP 설정

**자주 오해하거나 실수하는 부분:** `VBoxManage`는 VM의 **하드웨어** 설정(NIC 연결 방식)만 제어할 수 있습니다. VM **내부 OS**의 IP 주소 설정(Netplan, NetworkManager 등)은 VM이 부팅된 후 **Ansible**과 같은 구성 관리 도구를 사용하여 완료해야 합니다.

**권장되는 다음 단계 (Ansible을 통한 고정 IP 설정):**

1.  VM이 DHCP로 임시 IP를 할당받아 부팅됩니다.
2.  Ansible 플레이북을 실행하여 각 서버에 SSH 접속합니다.
3.  Ansible의 `ansible.builtin.template` 모듈을 사용하여 **`/etc/netplan/01-netcfg.yaml`** 파일을 아래 내용으로 덮어씁니다.

<!-- end list -->

```yaml
# Ansible로 적용할 netplan 설정 예시 (Master/DB 서버에서 실행)
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3: # 실제 NIC 이름으로 변경 필요
      dhcp4: no
      addresses: [ '{{ MASTER_IP 또는 DB_IP }}/24' ] 
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```
