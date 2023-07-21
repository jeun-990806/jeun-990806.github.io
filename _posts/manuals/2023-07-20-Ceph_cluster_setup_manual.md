---
title: Ceph cluster setup manual
author: Jieun
date: 2023-07-20
layout: post
category: manuals
---

한 학기동안 Ceph 때문에 꽤나 고생을 했다. 공식 문서 매뉴얼은 잘 작성되어 있는 것 같은데, 셋업 과정에서 오류가 생기면 참고할만한 문서가 많이 없다. 어차피 매뉴얼로 남겨야 해서, 겸사겸사 블로그에도 옮긴다.

## 노드 구성
본 매뉴얼에서는 다음과 같이 클러스터를 구성했다.
* admin_node: `ceph-deploy`를 실행하는 노드
* monitor_node: 모니터 노드
* manager_node: 매니저 노드
* osd_node_1, osd_node_2, osd_node_3: Object Storage Daemons

현 시점 기준, 구성 시 살펴봐야 할 것은 다음과 같다.
* **모든 노드의 운영체제는 Ubuntu 20.04 이하여야 한다.** Ubuntu 22.04를 지원하는 Ceph release가 무슨 이유인지 동결되는 바람에 설치가 제대로 안 됨.
* 모든 노드는 ssh 접속이 가능해야 한다.
* Monitor 노드는 public IP가 할당되어 있으며, 3300, 6789 포트가 열려 있어야 한다.
    * 모든 노드가 같은 네트워크 상에 존재하면 private IP로도 모니터링이 가능하다. 
    * 공유기에 연결된 머신을 포트포워딩하여 monitor로 사용하려고 하면 자동 구성이 꼬여서 문제가 발생하는 듯.
* OSD 노드는 6800-7300 포트가 열려 있어야 한다. 그리고 사용하지 않는 저장 장치가 있어야 한다.

### 포트 설정하기
`iptables` or `ufw`를 사용해 포트를 열어둔다.
* `iptables`를 사용하는 경우 (Monitor 노드에서 6789 포트 열기)
```shell
sudo iptables -A INPUT -p tcp --dport 6789 -j ACCEPT
```
* `ufw`를 사용하는 경우 (OSD 노드에서 6800-7300 포트 열기)
```shell
sudo ufw allow 6300:7300/tcp
```

### 저장 장치 확인하기
`fdisk` 커맨드로 인식된 저장 장치를 모두 확인할 수 있다.
```shell
sudo fdisk --list
```
마운트되어 있거나, 파티션이 잡혀있는 저장 장치는 사용할 수 없다. 따라서 파티션 삭제 후 `wipefs` 커맨드로 파일 시스템까지 지워 주자. 가끔 마운트되지 않은 장치에 `Device or resource busy` 에러가 발생하는 경우가 있는데, 가상 볼륨에 매핑되어 있지는 않은지 체크하자. 매핑 정보는 `dmsetup` 커맨드로 확인할 수 있다.
```shell
sudo dmsetup info
```

## Ceph-deploy 설치
`ceph-deploy`를 실행할 노드에는 `ceph-deploy`가 설치되어 있어야 한다. 여기서는 `octopus` 버전을 기준으로 설명한다.
```shell
wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
echo deb https://download.ceph.com/debian-octopus/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
sudo apt update
sudo apt install ceph-deploy
```

## SSH 설정
다음으로는 password-less ssh 설정이 필요하다. 우선, 모든 Ceph 노드에 `ntp`를 설치한다.
```shell
sudo apt install ntp
```
그리고, admin 노드가 접속하는 데 사용할 계정을 새로 만든다. 
```shell
sudo useradd -d /home/ceph-node -m ceph-node
sudo passwd ceph-node
```
`sudo` 권한이 있으면서, 패스워드 입력 없이 `sudo`가 가능해야 한다. `ceph-deploy` 중 패스워드 입력이 불가능해서라는 듯.
```shell
echo "ceph-node ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ceph-node
sudo chmod 0440 /etc/sudoers.d/ceph-node
```
이렇게 모든 노드에 새 계정을 만들어둔다.

admin 노드로 돌아가서, ssh의 구성을 수정한다. 
```shell
vi ~/.ssh/config
```
`config` 파일은 다음과 같이 쓸 수 있다.
```shell
Name monitor_node
    HostName 123.456.78.90
    Port 50001
    User ceph-node

Name manager_node
    HostName 123.456.78.91
    Port 50001
    User ceph-node

Name osd_node_1
    HostName 123.456.78.92
    Port 50001
    User ceph-node

Name osd_node_2
    HostName 123.456.78.93
    Port 50001
    User ceph-node

Name osd_node_3
    HostName 123.456.78.94
    Port 50001
    User ceph-node
```
그리고 SSH keys를 생성한다.
```shell
ssh-keygen

Generating public/private key pair.
Enter file in which to save the key (/ceph-admin/.ssh/id_rsa): [Enter]
Enter passphrase (empty for no passphrase): [Enter]
Enter same passphrase again: [Enter]
Your identification has been saved in /ceph-admin/.ssh/id_rsa.
Your public key has been saved in /ceph-admin/.ssh/id_rsa.pub.
```
마지막으로, 모든 노드에 SSH keys를 배포한다.
```shell
ssh-copy-id monitor_node
ssh-copy-id manager_node
ssh-copy-id osd_node_1
ssh-copy-id osd_node_2
ssh-copy-id osd_node_3
```
## Create Cluster
admin 노드에서 Ceph keyring을 저장할 디렉토리를 만든다. `ceph-deploy`로 클러스터를 생성하면, 현재 디렉토리에 keyring이 생성되기 때문이다.
```shell
mkdir ceph-cluster
cd ceph-cluster
```
클러스터 생성은 initial monitor를 추가함으로써 시작한다.
```shell
ceph-deploy new monitor_node
```
그리고 클러스터 노드들에 Ceph를 설치한다.
```shell
ceph-deploy install monitor_node manager_node osd_node_1 osd_node_2 osd_node_3
```
초기 모니터 노드를 배포하고 key를 모은다. 네트워크 구성 상 문제가 있는 경우, 주로 이 단계에서 문제가 발생한다.
```shell
ceph-deploy mon create-initial
```
그리고, keyring을 클러스터 내의 노드에 배포한다.
```shell
ceph-deploy admin monitor_node manager_node osd_node_1 osd_node_2 osd_node_3
```
Manager 노드를 배포한다.
```shell
ceph-deploy mgr create manager_node
```
마지막으로 OSD 노드를 추가한다. `--data` 옵션으로는, 데이터를 저장할 저장 장치의 이름을 설정하면 된다.
```shell
ceph-deploy osd create --data /dev/sda osd_node_1
ceph-deploy osd create --data /dev/sda osd_node_2
ceph-deploy osd create --data /dev/sda osd_node_3
```

## Check the Cluster Status
`ceph-deploy install`로 Ceph가 설치된 노드에서는,
```shell
sudo ceph -s
```
커맨드로 클러스터의 상태와 구성 정보를 확인할 수 있다. `/etc/ceph`에 저장된 키링을 사용해야 하므로, 수퍼 유저 권한으로만 실행된다.

## Troubleshooting
### Can't communicate with remote host... possibly because python3 is not installed there: cannot send (already closed?)
`~/.ssh/config`에 명시된 사용자가 `sudoer`에 추가되어 있는지 확인하자.  
사담으로, 정말 도움 안 되는 오류 메시지같다. 대부분의 머신에서 파이썬은 설치되어 있을 것이다. `sudo`가 안 되어서 발생하는 경우가 태반…