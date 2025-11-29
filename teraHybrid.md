🚀 Terraform 기반 하이브리드 Kubernetes 클러스터 실습 환경 구축 계획
사용자님께서 요청하셨던 Terraform을 활용한 Kubernetes 클러스터 구축 실습 환경에 대한 구체적인 계획을 단계별로 안내해 드립니다. 이 실습은 VirtualBox(로컬)를 마스터 노드로, AWS EC2를 워커 노드로 사용하는 하이브리드 클러스터를 목표로 합니다.
1. ⚙️ 필수 전제 조건 (Local PC)
실습을 시작하기 전에 로컬 PC에 설치해야 할 핵심 도구들입니다.
 * VirtualBox: 로컬 환경에 마스터 VM을 생성하기 위한 하이퍼바이저.
 * Vagrant (선택 사항): VirtualBox VM의 초기 설정(OS 설치, 네트워크 구성)을 스크립트로 자동화하는 데 유용합니다. (Terraform의 Local Provisioner로 대체 가능)
 * Terraform: 인프라 배포를 자동화하는 핵심 도구.
 * AWS CLI: AWS 리소스를 관리하고, Terraform AWS Provider가 인증할 수 있도록 설정.
 * kubectl: 클러스터 배포 후 관리 및 모니터링을 위한 도구.
2. 🏗️ 아키텍처 개요 및 핵심 과제
구축할 하이브리드 클러스터의 구조와 각 환경에서 Terraform이 수행할 주요 역할입니다.
 * 하이브리드 아키텍처:
   * Master Node: Local (VirtualBox VM)
   * Worker Nodes: Cloud (AWS EC2 Instances)
 * 핵심 과제:
   * 네트워크 연결: VirtualBox VM이 AWS 워커 노드와 통신하고, 퍼블릭 인터넷을 통해 외부 AWS 서비스에 접근할 수 있도록 네트워크를 구성해야 합니다. (예: VirtualBox VM에 NAT 및 Host-Only 네트워크 설정 후 AWS VPC와 VPN/터널링 구성 또는 가장 간단하게 Public IP를 통한 통신 설정).
   * Terraform 멀티 프로바이더: hashicorp/aws 프로바이더와 master-node/virtualbox 프로바이더(혹은 Vagrant를 통한 로컬 프로비저닝)를 동시에 설정해야 합니다.
3. 📝 Terraform 구축 단계별 계획
단계 A: 초기 설정 및 AWS 인프라 구성
 * AWS Provider 설정: AWS 인증 정보 및 리전 설정.
 * VPC 및 Subnet: AWS에서 워커 노드가 배포될 VPC, 서브넷, 인터넷 게이트웨이 등을 Terraform으로 구성합니다.
 * Security Group: 마스터(로컬)와 워커(AWS) 간 통신을 허용하는 보안 그룹(특히 NodePort, Calico/Flannel 통신 포트 등)을 정의합니다.
 * AWS Worker 노드 생성: EC2 인스턴스를 워커 노드 수만큼 생성하고, 초기 OS 설정(예: Ubuntu 22.04 LTS), 도커 설치 및 실행을 위한 user_data 스크립트를 정의합니다.
단계 B: VirtualBox Master 노드 구성
 * VirtualBox Provider: VirtualBox VM 이미지를 불러와 마스터 노드 VM을 생성합니다. (IP 주소는 고정 IP로 설정하여 AWS와 통신 가능하도록 구성)
 * OS 및 도커 설치: 생성된 VM에 Terraform Local Provisioner 또는 Remote-Exec Provisioner를 사용하여 OS 업데이트 및 **도커(혹은 Containerd)**를 설치합니다.
단계 C: Kubernetes 클러스터 설치 및 통합 (kubeadm 활용)
 * Kubeadm 설치: Master 및 Worker 노드 모두에 Kubeadm, Kubelet, Kubectl을 Terraform의 Remote-Exec을 통해 설치합니다.
 * Master 초기화:
   * 마스터 노드에서 kubeadm init 명령을 실행합니다.
   * 이때 생성된 kubeadm join 토큰 및 명령어를 변수로 저장합니다.
 * Worker 노드 통합:
   * AWS 워커 노드에서 마스터 노드 IP와 저장된 토큰을 사용하여 kubeadm join 명령어를 실행합니다.
   * Worker 노드가 Master 노드와 통신할 수 있도록 네트워크 설정(보안 그룹)을 반드시 확인해야 합니다.
 * CNI (네트워크 플러그인) 배포: 마스터 노드에서 Calico 또는 Flannel과 같은 CNI를 배포하여 Pod 간 통신을 활성화합니다.
단계 D: 로컬 PC 모니터링 환경 구축
 * kubeconfig 복사: 마스터 노드의 .kube/config 파일을 로컬 PC로 안전하게 복사하여 로컬 kubectl이 클러스터를 관리할 수 있도록 합니다.
 * 모니터링 스택 배포:
   * Prometheus와 Grafana를 클러스터 내에 Helm 또는 YAML 파일로 배포합니다.
   * 로컬 PC에서 Grafana 웹 UI에 접속할 수 있도록 NodePort 또는 Ingress를 설정합니다.
이 실습은 Terraform의 멀티 프로바이더 사용, Remote-Exec 프로비저닝, 네트워크 구성 등 여러 고급 기술을 요구합니다.
가장 먼저 Terraform 코드를 AWS와 VirtualBox 프로바이더를 중심으로 어떻게 구조화할지부터 시작해 볼까요?
