# **OpenShift v4.8 BareMetal Disconnected Installation Guide (UPI)**

<br>

## **Guide History**
|   작성일   | 버전 |   내용   | 비고  |
| :--------: | :--: | :------: | :---: |
| 2020/08/17 | v1.0 | 초기작성 | mykim |
| 2021/06.03 | v1.1 | ocp 4.7 테스트 | mykim |
| 2021/09/18 | v1.2 | ocp 4.8 테스트 | mykim |
| 2022/01/04 | v1.3 | ocp 4.9 테스트 | mykim |

<br>

## **테스트 환경**
> 테스트 환경은 OpenShift `4.8.0` 버전에서 진행하였습니다. 

|   구분    |          hostname          |                  IP 주소                  | 수량 | 용도 |
| :-------: | :------------------------: | :---------------------------------------: | :--: | :--: |
| bootstrap |  `bootstrap.rhocp4.cloud.com`  |               192.168.35.208               |  1   |임시  |
|  bastion  |   `bastion.rhocp4.cloud.com`   |               192.168.35.200               |  1   |bastion, 사전준비, nfs      |
|  master   | `master[1-3].rhocp4.cloud.com` |             192.168.35.201~203             |  3   |Control Plan      |
|   infra   | `infra[1-2].rhocp4.cloud.com` |             192.168.35.204~205             |  2   |Compute     |
|  router  | `router[1-2].rhocp4.cloud.com` |             192.168.35.206~207             |  2   |Compute      |
| worker | `worker[1-2].rhocp4.cloud.com` | 192.168.35.208~209 | 2 |Compute |

<br>

## **Guide 진행 절차**
> **인터넷이 안되는 환경에서 설치하기 위해 필요한 리소스(package, image, 설정파일등)들을 다운받고 Script를 통해 실 설치환경에서 편리하게 작업하기 위함입니다.**

[1. 사전 준비 작업](#1.-사전-준비-작업)

  - repository(RPM Package)
  - Image/Operator Mirror
  - Tools 및 설치에 필요한 ISO(oc, openshift-install, rhcos)

[2. 설치에 필요한 환경 구성 및 bastion 설정](#2.-설치에-필요한-환경-구성-및-bastion-설정)
  - dns   
  - haproxy
  - HTTPD
  - nfs-server
  - 설치에 필요한 ignition 파일 생성

[3. OpenShift 설치](#3.-OpenShift-설치)

  - openshift-install을 통한 설치

[4. OpenShift 기본 설정](#4.-OpenShift-기본-설정)

  - 계정생성
  - infra/worker node 구분
  - nfs-client-provisioner

<br>

설치에 사용된 스크립트는 아래와 같습니다.
```bash
v4/
├── 00.prepare
│   ├── 00_sshkey_copy.sh
│   ├── 01_hostname_change.sh
│   ├── 02_repo_setting.sh
│   ├── 03_dns_setting.sh
│   ├── 04_network_change.sh
│   ├── 05_chrony_setting.sh
│   ├── 06_haproxy_setting.sh
│   ├── 07_nfs_setting.sh
│   ├── 08_tool_setting.sh
│   ├── 09_dhcp_setting.sh
│   ├── 10_prepare_check.sh
│   ├── config
│   │   ├── dns
│   │   │   ├── cloud.com.rr.zone
│   │   │   ├── cloud.com.zone
│   │   │   ├── named.conf
│   │   │   ├── named.sample
│   │   │   ├── nirs.go.kr.rr.zone
│   │   │   ├── nirs.go.kr.zone
│   │   │   ├── rr-zone.sample
│   │   │   └── zone.sample
│   │   ├── firewalld.txt
│   │   ├── haproxy.cfg.sample
│   │   ├── install-config.org
│   │   ├── install-config.org2
│   │   ├── install-config.sample
│   │   ├── install-config.yaml
│   │   ├── local.sample
│   │   ├── local.sample.rhel7
│   │   ├── local.sample.rhel8
│   │   ├── net.sample
│   │   ├── nodes-ign.sample
│   │   ├── ocp4.sample
│   │   ├── ocp4.sample.rhel7
│   │   ├── ocp4.sample.rhel8
│   │   └── openshift_nodes
│   ├── convert.sh
│   └── httpd.sh
├── 01.ext-registry
│   ├── 00_podman.sh
│   ├── 01_mirror_registry.sh
│   ├── 02_mirror_operator.sh
│   ├── clean.sh
│   ├── config
│   │   ├── ext-registry.sample
│   │   ├── ext-registry.sample.org
│   │   ├── ext-secret.json
│   │   ├── ext-secret.sample
│   │   ├── pull-secret.json
│   │   └── pull-secret.org
│   ├── create_pullsecret.sh
│   ├── service
│   │   └── ext-registry.service
│   └── tar.sh
├── 10.default-setting
│   ├── 00_htpasswd_identity_provider.sh
│   ├── 01_machine_config_poll.sh
│   ├── 02_node_label_change.sh
│   ├── 03_router_registry_move.sh
│   ├── 04_taint_tolerations.sh
│   ├── 05_nfs_client_provisioner.sh
│   ├── 06_registry_pv_setting.sh
│   ├── 07_operator_catalogsource.sh
│   ├── 08_cluster_logging.sh
│   ├── 09_quay.sh
│   ├── no-provisioner
│   │   └── no-provisioner.yaml
│   ├── pod_check.sh
│   ├── prj_delete.sh
│   ├── quay-cert
│   └── set-time
│       ├── 99-master-chrony.bu
│       ├── 99-master-chrony.yaml
│       ├── 99-master-timezone.yaml
│       ├── 99-worker-chrony.bu
│       ├── 99-worker-chrony.yaml
│       ├── 99-worker-timezone.yaml
│       ├── my_registry.conf
│       └── worker-mirror-by-digest-registries.yaml
├── 99.download
│   ├── clean.sh
│   ├── client
│   ├── install
│   ├── repos
│   │   └── README
│   ├── rhcos
│   ├── sync.sh
│   ├── tar.sh
│   └── tools_download.sh
├── install.sh
├── inventory
│   └── hosts
└── openshift.env
```

<br>

Script중에 수정이 필요한 파일은 아래와 같습니다.

- openshift.env<br>
설치에 필요한 전반적인 설정파일
- 00.prepare/config/openshift_nodes<br>
구성하고자 하는 node 정보
- 01.ext-registry/pull-secret.org<br>
image다운을 위한 pull-scret (https://cloud.redhat.com/openshift/install/metal)

---

## 1. 사전 준비 작업
> 인터넷이 안되는 환경에서 OpenShift를 구성하기 위하여 다음과 같은 준비과정이 필요합니다.
- yum을 사용하기 위한 repository 설정 (package 다운 및 createrepo) 
- 설치에 필요한 tools 및 rhcos
  - client : openshift-client-linux-4.9.10.tar.gz
  - installer : openshift-install-linux-4.9.10.tar.gz
  - rhcos : rhcos-live.x86_64.iso

**스크립트를 사용하기 위한 설정파일을 기준으로 사전 준비 shell을 수행하여 다운로드 합니다.**<br>
`${BASTION_DIR}/openshift.env`

```bash
...
#== general variables ==#
SCRIPT_VERSION=v1.3
DATETIME=`date +%Y%m%d%H%M%S`
INSTALL_USER=root
BASTION_IP=192.168.35.60
BASTION_DIR=/root/.openshift/v4
CLUSTER_NAME=rhocp4
INSTALL_DIR=${BASTION_DIR}/${CLUSTER_NAME}
BASE_DOMAIN=cloud.com

#INVENTORY_PATH=${BASTION_DIR}/inventory_file
OPENSHIFT_VERSION=4.9
OPENSHIFT_RELEASE=4.9.10
RHCOS_VERSION=4.9.0
OPENSHIFT_NODES=${BASTION_DIR}/00.prepare/config/openshift_nodes
CONVERTSHL=convert.sh
...
```

### 설치를 위한 package 다운
shell을 실행하여 `createrepo`를 수행하기 전 `subscription-manager`를 통해 등록이 필요합니다. 
```bash
subscription-manager register
subscription-manager list --available --matches '*OpenShift*'
subscription-manager attach --pool=<pool-id>
subscription-manager repos --disable="*"
subscription-manager repos \
    --enable="rhel-8-for-x86_64-baseos-rpms" \
    --enable="rhel-8-for-x86_64-appstream-rpms" \
    --enable="rhocp-4.9-for-rhel-8-x86_64-rpms" \
    --enable="fast-datapath-for-rhel-8-x86_64-rpms"
yum -y update
```

<br>

등록이 완료되면 아래 shell을 실행하여 선택한 rpms를 다운 받습니다. 

```bash
/root/.openshift/v4/99.download/sync.sh
```
`creadrepo`가 완료되면 `/root/.openshift/v4/99.download/repos`경로에서 확인 가능합니다. 

<br>

다음으로 설치 시 필요한 도구 및 설치파일을 다운로드 합니다. 
```bash
/root/.openshift/v4/99.download/tools_download.sh
```
shell이 수행되면 `/root/.openshift/v4/99.download`하위에 `client/, install/, rhcos/` 디렉토리에 해당 파일을 다운로드합니다.

<br>

마지막으로 설치 시 필요한 image와 operator를 mirror를 통해 저장합니다. <br>
다운을 위해 다음 순서로 진행됩니다. 

1. podman 설치 및 ext-registry 구성
2. image mirror
3. operator mirror

shell를 실행하기 전 image 및 operator를 다운받기 위한 pull-scret이 필요합니다. <br>
pull-scret를 복사하여 `/root/.openshift/v4/01.ext-registry/config/pull-secret.org` 파일에 붙여넣어줍니다.<br>
pull-scret은 다음 경로에서 얻을수 있습니다.<br>
(https://cloud.redhat.com/openshift/install/metal)

<br>

podman을 설치하고 ext-registry를 구성합니다. 
```bash
/root/.openshift/v4/01.ext-registry/00_podman.sh
```

<br>

ext-registry에 image mirror를 통해 다운받습니다. 
```bash
/root/.openshift/v4/01.ext-registry/01_mirror_registry.sh
```

<br>

ext-registry에 operator mirror를 통해 다운받습니다. 
```bash
/root/.openshift/v4/01.ext-registry/02_mirror_operator.sh
```


완료되면 `/opt/registry`경로 하위에 `auth/  certs/ data/` 디렉토리가 생성되고 `data/` 디렉토리에 image 및 operator가 다운됩니다. 

<br>

위에서 다운로드 받은 `tools, repos, image/operator`를 압축하여 인터넷이 안되는 환경으로 복사 후 설치를 진행합니다. 
```bash
/root/.openshift/v4/01.ext-registry/tar.sh
/root/.openshift/v4/99.download/tar.sh
```

<br>

지금까지 설치를 위한 파일에 대하여 준비하는 과정이였습니다. <br>
다음 step에서는 인터넷이 안되는 환경에 설치를 위한 bastion서버를 구성하도록 하겠습니다. 

---

## 2. 설치에 필요한 환경 구성 및 bastion 설정
> 앞선 step에서 준비한 파일을 가지고 bastion서버를 합니다. <br>
이번 step에서는 다음과 같은 작업을 수행하고 준비합니다. 

- ssh key 생성
- bastion서버 hostname 변경
- repository 설정
- dns 설정
- network 설정
- chrony 설정
- haproxy 설정
- nfs-server 설정 (Option)
- tools 설정
- dhcp 설정 (Option)
- 위 모든 설정에 대하여 check

<br>

shell을 실행하기 전 환경에 맞도록 `/root/.openshift/v4/openshift.env` 파일을 수정합니다. 
```bash
...
#== general variables ==#
SCRIPT_VERSION=v1.3
DATETIME=`date +%Y%m%d%H%M%S`
INSTALL_USER=root
BASTION_IP=192.168.35.60
BASTION_DIR=/root/.openshift/v4
CLUSTER_NAME=rhocp4
INSTALL_DIR=${BASTION_DIR}/${CLUSTER_NAME}
BASE_DOMAIN=cloud.com

#INVENTORY_PATH=${BASTION_DIR}/inventory_file
OPENSHIFT_VERSION=4.9
OPENSHIFT_RELEASE=4.9.10
RHCOS_VERSION=4.9.0
OPENSHIFT_NODES=${BASTION_DIR}/00.prepare/config/openshift_nodes
CONVERTSHL=convert.sh

#== repository variables ==#
REPO_IP=192.168.35.60
REPO_PORT=9000
LOCAL_REPO_PATH="\/root\/.openshift\/v4\/99.download"

#== dns variables ==#
DNS_IP=192.168.35.60
DNS_REVERSE=35.168.192
DNS_DOMAIN=${CLUSTER_NAME}.${BASE_DOMAIN}

#== network variables ==#
NW_NAME="ens3"
NW_PREFIX=24
NW_GATEWAY=192.168.35.1
NW_NETMASK=255.255.255.0
NW_DNS1=${DNS_IP}
NW_DNS_SEARCH=${DNS_DOMAIN}
NODE_NW_NAME=ens3
NODE_NW_PREFIX=${NW_PREFIX}
NODE_NW_GATEWAY=${NW_GATEWAY}
NODE_NW_NETMASK=255.255.255.0

#== chrony variables ==#
CHRONY_SERVER=192.168.35.60
CHRONY_ALLOW=192.168.35.0\/23
CHRONY_STRATUM=3

#== external registry ==#
REGISTRY_PATH=
REGISTRY_NAME=ext-registry
REGISTRY_USER="admin"
REGISTRY_PASSWD="passwd"
REGISTRY_PASSWD_ECD=`echo -n ${REGISTRY_USER}:${REGISTRY_PASSWD} | base64`
REGISTRY_URL="ext-registry.${DNS_DOMAIN}"
#REGISTRY_URL="bastion.${DNS_DOMAIN}"
REGISTRY_PORT=5000

LOCAL_REGISTRY=${REGISTRY_URL}:${REGISTRY_PORT}
LOCAL_REPOSITORY='ocp4/openshift4'
PRODUCT_REPO='openshift-release-dev'
LOCAL_SECRET_JSON=${ABSOLUTE_PATH}/config/pull-secret.json
RELEASE_NAME="ocp-release"
RELEASE_INDEX_NAME="redhat/redhat-operator-index:v${OPENSHIFT_VERSION}"
LOCAL_INDEX_NAME="openshift4/redhat-operator-index:v${OPENSHIFT_VERSION}"
FILTER='linux/amd64'
#FILTER='.*'

#== haproxy variables ==#
HAPROXY_IP=192.168.35.60
HAPROXY_NAME=ocp4-lb

#== nfs server ==#
NFS_IP=192.168.35.60
NFS_HOST=nfs.${DNS_DOMAIN}
NFS_STORAGE="/nfs_storage/volumes"
NFS_PORT1=111
NFS_PORT2=2049
NFS_PORT3=20048

#== DHCP server ==#
#DHCP_RANGE_S=192.168.35.201
#DHCP_RANGE_E=192.168.35.208

#== OCP Default Setting ==#
OCP_CLUSTER_ADMIN=admin
OCP_CLUSTER_ADMIN_PW=r3dh4t1!
OCP_IDENTITY_NAME=ocp-admin

OPM_VERSION=4.6.1
GRPCURL_VERSION=1.8.5
HELM_VERSION=3.7.1

NFS_STORAGE_CLASS_NAME=managed-nfs-storage
NFS_PROJECT_NAME=nfs-client-provisioner
#PROVISIONER_NAME=fuseim.pri/ifs
#PROVISIONER_NAME=storage.io/nfs
PROVISIONER_NAME=k8s-sigs.io/nfs-subdir-external-provisioner

ELASTICSEARCH_COUNT=1
ELASTICSEARCH_MEM=4Gi

KIBANA_COUNT=1
CLUSTER_LOGGING_SIZE=100G
RETENTION_APP_D=7D
RETENTION_AUD_D=7D
RETENTION_INF_D=7D

#== quay ==#
QUAY_PROJECT=quay-registry
QUAY_REGISTRY_NAME=quay-registry
QUAY_URL=${QUAY_PROJECT}.apps.${DNS_DOMAIN}
BACKINGSTORE_SIZE=100Gi

#== helm ==#
CHART_PROJECT_NAME=chartmuseum
CHART_MUSEUM_SIZE=10Gi
HELMCHARTREPOSITORY_NAME=chartmuseum

#== compliance ==#
COMPLIANCE_PROJECT_NAME=openshift-compliance
HTTPD_APP_NAME=httpd-result
HTTPD_PV_SIZE=10Gi
```

<br>

ssh key를 생성 및 추가 합니다. 
```bash
/root/.openshift/v4/00.prepare/00_sshkey_copy.sh
```

<br>

다운 받은 package를 가지고 local repo를 설정을 합니다. 
```bash
/root/.openshift/v4/00.prepare/02_repo_setting.sh
```

<br>

Cluster구성에 필요한 dns 설정을 합니다. 
```bash
/root/.openshift/v4/00.prepare/03_dns_setting.sh
```

<br>

bastion서버의 네트워스 설정을 합니다. 
```bash
/root/.openshift/v4/00.prepare/04_network_change.sh
```

<br>

bastion서버의 NTP설정을 합니다. 
```bash
/root/.openshift/v4/00.prepare/05_chrony_setting.sh
```

<br>

Cluster 구성시에 Load-Balance 역할의 haproxy 설정을 합니다.
```bash
/root/.openshift/v4/00.prepare/06_haproxy_setting.sh
```

<br>

Cluster 구성 후 nfs storage를 사용하기 위한 설정을 합니다. 
```bash
/root/.openshift/v4/00.prepare/07_nfs_setting.sh
```

<br>

다운받은 tools(oc, openshift-install)을 PATH 없이 사용하기 위해 설정을 합니다. 
```bash
/root/.openshift/v4/00.prepare/08_tool_setting.sh
```

<br>

DHCP로 구성 및 설정을 합니다. 

```bash
/root/.openshift/v4/00.prepare/09_dhcp_setting.sh
```

<br>

마지막으로 위 설정들이 정상적으로 구성되었는지 확인 합니다. 
```bash
/root/.openshift/v4/00.prepare/10_prepare_check.sh
```

<br>

OpenShift를 설치 하기 전 사전 준비작업이 완료되었습니다. <br>
다음 step에서는 `openshift-install`을 통한 ignition파일 생성 및 실 설치를 진행합니다. 

---

## 3. OpenShift 설치
> bastion서버의 사전 준비 작업이 완료되었으면 `opoenshift-install` 명령어를 통해 `ignition`파일을 생성 후 설치를 진행합니다. 

- `openshift-install` 명령어를 통해 manifests 생성
- `openshift-install` 명령어를 통해 ignition 생성
- 생성된 ignition 파일을 가지고 설치하고자 하는 OpenShift 환경에 맞게 설정파일을 생성
- 각 node 설치 진행
- 로그를 통한 정상 설치 여부 확인
- 인증서 서명 승인

<br>

`openshift-install` 명령어를 통해 manifests 파일을 생성합니다.
```bash
# Usage: ./install.sh [ clean | rm | manifests | ignition | custom | coreos | mon | time | cert ]
/root/.openshift/v4/install.sh manifests
```

<br>

`openshift-install` 명령어를 통해 ignition 파일을 생성합니다.
```bash
/root/.openshift/v4/install.sh ignition
```

<br>

설치 환경에 맞게 설정파일을 수정/생성 합니다. 
```bash
/root/.openshift/v4/install.sh custom
```

<br>

coreos 설치시 참고할 명령어를 출력합니다. 
```bash
/root/.openshift/v4/install.sh coreos
```

<br>

bootstrap의 로그를 확인합니다. 
```bash
# Usage: ./install mon {bootstrap|nodes}
/root/.openshift/v4/install.sh mon bootstrap

# 출력되는 메시지 확인
INFO Waiting up to 20m0s for the Kubernetes API at https://api.ocp4.cloud.com:6443...
INFO API v1.18.3+b74c5ed up
INFO Waiting up to 40m0s for bootstrapping to complete...
INFO It is now safe to remove the bootstrap resources
INFO Time elapsed: 0s
```

<br>

master가 모두 구성되고 나머지 node들의 로그를 확인합니다. 
```bash
/root/.openshift/v4/install.sh mon nodes

# 출력되는 메시지 확인
INFO Waiting up to 30m0s for the cluster at https://api.ocp4.cloud.com:6443 to initialize...
INFO Waiting up to 10m0s for the openshift-console route to be created...
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/.openshift/v4/ocp4/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp4.cloud.com
INFO Login to the console with user: "kubeadmin", and password: "XhaaZ-IbkHC-twRg6-ACr7v"
INFO Time elapsed: 0s
```

<br>

설치된 RHCOS의 `TimeZone`을 `Asia/Seoul`로 변경합니다. 
```bash
/root/.openshift/v4/install.sh time
```

<br>

추가된 노드의 인증서 서명을 승인합니다. 
```bash
/root/.openshift/v4/install.sh cert
```

<br>

OpenShift 기본 구성이 완료되었고, 다음 setp에서는 기본적인 설정을 진행합니다. 

---

## 4. OpenShift 기본 설정
> 구성된 OpenShift를 사용하기 위한 최소한의 기본적인 설정을 수행합니다. 

- htpasswd 기반의 provider 생성 및 계정 생성
- machine config pool 생성
- node label 변경
- router/registry 각 설정 node로 이동
- taint/tolerations 설정
- nfs client provisioner 설정
- Internal Image Registry PV 설정
- catalogsource 설정
- cluster-logging 구성 설정

<br>

htpasswd probider를 생성 하고 `openshift.env`에 정의된 사용자를 만듭니다. 이후 해당 사용자에 `cluster-admin` 권한을 부여합니다. 
```bash
/root/.openshift/v4/10.default-setting/00_htpasswd_identity_provider.sh
```

<br>

정의되어 있는 node에 따라 MCP를 생성합니다.
```bash
/root/.openshift/v4/10.default-setting/01_machine_config_poll.sh
```

<br>

정의되어 있는 node의 용도에 따라 label를 추가/수정합니다. 
```bash
/root/.openshift/v4/10.default-setting/02_node_label_change.sh
```

<br>

router와 registry를 label에 맞게 이동시킵니다. 
```bash
/root/.openshift/v4/10.default-setting/03_router_registry_move.sh
```

<br>

INFRA영역을 구분하고 INFRA자원 외 다른 컨테이너들은 실행되지 못하도록 설정합니다. 
```bash
/root/.openshift/v4/10.default-setting/04_taint_tolerations.sh
```

<br>

NFS storage-class를 사용하기 위해 설정합니다. 
```bash
/root/.openshift/v4/10.default-setting/05_nfs-client-provisioner.sh
```

<br>

Internal Image Registry에 PV를 설정합니다. 

```bash
/root/.openshift/v4/10.default-setting/06_registry_pv_setting.sh
```

<br>

CatalogSource를 구성합니다.

```bash
/root/.openshift/v4/10.default-setting/07_operator_catalogsource.sh
```

<br>

Cluster Logging을 구성합니다. (pvc를 통한 영속적인 데이터 저장)

```bash
/root/.openshift/v4/10.default-setting/08_cluster_logging.sh
```
