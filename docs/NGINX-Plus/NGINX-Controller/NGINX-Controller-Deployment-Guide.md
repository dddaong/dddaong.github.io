NGINX Controller 설치 가이드


## NGINX Controller


본 가이드에서는 이중화 등이 없는 싱글 노드 NGINX Controller의 설치를 다룹니다.


단, Online Install이 가능한 환경, Offline Install이 필요한 환경으로 구분해 각각 소개합니다.


- 가이드에서는 `CentOS 7.4+` 환경을 사용합니다.


- `controller-installer-<version>.tar.gz` 패키지와 `controller_license.txt` 라이선스 파일이 필요하며, `controller-installer-<version>.tar.gz` 파일이 `/var/tmp`에 업로드 되어있다고 가정합니다.
- Embedded Config DB를 사용할 수도 있으나 가이드를 위해 PostgreSQL을 사용합니다.
- NFS 사용법도 과정 중에 간략히 소개합니다.






> ###### User 주의사항
>
> 3.12.0 이상의 버전은 `root` 계정으로 `NGINX Controller`를 설치할 수 없습니다.


> ###### Clustering 주의 사항
>
> Controller Clustering을 위해서는 ConfigDB, TimeSeriesDB 모두 Local를 사용해선 안됩니다.
>
> Configdb, tsdb 둘 다 Controller가 설치되는 노드가 아닌 별도의 노드에 설치되어야 하며, configdb는 별도 호스트에 postgresql을 설치, tsdb는 별도 호스트에 nfs를 설치 사용하는 방식으로 DB를 분리해야 합니다. 사용한다.
>
> 또한 온전한 Quorum 및 Resilient Cluster 구성을 위해선  3대의 노드가 필요합니다.
>
> 따라서 <u>온전한 Resilient Cluster 구성을 위해선 최소 4 대의 노드가 필요</u>합니다.






#### NGINX Controller Online Install


##### Node and Repository Set-up


```bash
#hostname and DNS Set
hostnamectl set-hostname ngxctlr.itian.ml
echo 'nameserver    8.8.8.8' >> /etc/resolv.conf
```






```bash
#CentOS Repository (kakao mirror) Set
tee -a /etc/yum.repos.d/centos-kakao.repo <<EOF 1>/dev/null
[base]
name=CentOS-7 - Base
baseurl=http://mirror.kakao.com/centos/7/os/x86_64/
gpgcheck=0
[updates]
name=CentOS-7 - Updates
baseurl=http://mirror.kakao.com/centos/7/updates/x86_64/
gpgcheck=0
[extras]
name=CentOS-$releasever - Extras
baseurl=http://mirror.kakao.com/centos/7/extras/x86_64/
gpgcheck=0
EOF
```






```bash
#Docker, PostgreSQL, EPEL Repository Set
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum install -y epel-release
```






```bash
#Google kubernetes Repository Set
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
sudo yum makecache fast
```






> 만일 Repo file에서 `$basearch`, `$releasever` 변수 적용이 안 되어 yum 명령에 오류가 발생하는 경우, 아래 Command Set을 실행하면 변수를 Static하게 변경합니다.
>
> ```bash
> export repolist="$(ls -p /etc/yum.repos.d/ | grep -v '/$')"
> export rel="$(rpm --query --file --queryformat '%{VERSION}\n' /etc/system-release | cut -c1 )"
> export barch="$(rpm --query --queryformat '%{ARCH}\n' "kernel-$(uname --kernel-release)")";
> for repos in $repolist;do sed -i s/\$releasever/${rel}/g /etc/yum.repos.d/$repos && sed -i s/\$basearch/${barch}/g /etc/yum.repos.d/$repos; done
> ```
>
> 






##### PostgreSQL Install


Local External DB를 설치하는 과정입니다.
Embedded ConfigDB를 사용하려면 이 과정을 **넘어가시면 됩니다.**


3.9.0 이상의 버전에서는 PostgreSQL12를 사용합니다. 그 이전의 버전에서는 PostgreSQL9를 사용합니다.


```bash
#Postgresql12 Install
sudo yum -y install postgresql12 postgresql12-server


#DB Initialize
/usr/pgsql-12/bin/postgresql-12-setup initdb 
```






```bash
# DB Listener Configure
sudo vi /var/lib/pgsql/12/data/postgresql.conf
```
> 기본값으로 주석처리 되어 있는 아래 라인을
>
> ```bash
> #listen_addresses = 'localhost'
> #port = 5432
> ```
>
> 원하는 `IP Address`와 함께 주석 해제 합니다.
>
> ```bash
> listen_addresses = '<IP Address>'
> port = 5432
> ```






```bash
# DB Access Control Configure
sudo vi /var/lib/pgsql/12/data/pg_hba.conf
```


>  `pg_hba.conf` 문서 하단의 Access Control 기본값은 아래와 같습니다.
>
>  ```bash
>  # TYPE  DATABASE        USER            ADDRESS                 METHOD
>  #"local" is for Unix domain socket connections only
>  local   all             all                                     peer
>  #IPv4 local connections:
>  host    all             all             127.0.0.1/32            ident
>  ...
>  ```
>
>  아래와 같이 수정합니다.
>
>  ```bash
>  # "local" is for Unix domain socket connections only
>  local   all             all                                     peer
>  local   all             nginxc                                  trust
>  local   all             postgres                                peer
>  # IPv4 local connections:
>  host    all             all             127.0.0.1/32            ident
>  host    all             all             0.0.0.0/0               trust
>  ```
>  
>[참고] Public IP를 사용하게 되면 PostgresSQL 취약점 공격이 자주 들어오기 때문에, 보안 상의 이유로 아래와 같이 제한해주는 것도 좋습니다. 
>  
>```bash
>  # "local" is for Unix domain socket connections only
>  local   all             all                                     peer
>  local   all             nginxc                                  trust
>  local   all             postgres                                peer
>  # IPv4 local connections:
>  host    all             all             127.0.0.1/32            ident
>  host    all             all             172.0.0.0/8             trust
>  host    all             all             10.0.0.0/8              trust
>  ```
>  
>참고로 `10.0.0.0/8`과 `172.0.0.0/8`은 NGINX Controller의 Kubernetes Pod Network Range 입니다.






```bash
#PostgreSQL12 Enable and Start
systemctl enable --now postgresql-12


#Set up Database, User, User Role and Authority
su - postgres
createuser nginxc;
createdb nginxc;
psql;


alter user nginxc with superuser createrole createdb replication bypassrls;
alter user nginxc with encrypted password 'nginxc';
alter database nginxc owner to nginxc;
 
\q
exit
```






###### NFS 설정 (for External tsdb, Optional for Clustering)


Clustering을 위해 별도 호스트에 tsdb를 NFS로 사용하는 방법으로, 참고를 위해 작성한 내용입니다.
싱글 노드 NGINX Controller 설치를 진행 중이라면 다음으로 **넘어가시면 됩니다.**


```bash
@DB Host
yum install -y portmap nfs-utils* libgssapi
vi /etc/exports
##
/var/ngxtsdb   *(rw,sync)


systemctl enable --now rpcbind;
systemctl enable --now portmap;
systemctl enable --now rpcidmapd;
systemctl enable --now nfs;


@Controller Node
yum install nfs-utils
#or
./opt/controller-installer/helper.sh prereqs nfs


systemctl enable --now rpcbind;
systemctl enable --now rpcidmapd;


#필요시
mount -t nfs <xxx.xxx.xxx.xxx>:/var/ngxtsdb/ /
```






##### NGINX Controller Install


```bash
### Hostname, DNS Set
hostnamectl set-hostname ngxctlr.itian.ml
echo 'nameserver    8.8.8.8' >> /etc/resolv.conf


### Kubernetes Node Initial Tweak


#Set Selinux status to Permissive
sed -i -re 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/sysconfig/selinux;
sed -i -re 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config;
setenforce 0;
#sestatus


## Firewall Stop and Diable
systemctl disable firewalld;
systemctl stop firewalld;
#systemctl status firewalld


## Swap Disable
swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab;
#swapon


## Linux Network and Docker Cgroup Driver
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p;
#sysctl net.ipv4.ip_forward;


tee -a /etc/sysctl.d/k8s.conf <<EOF 1>/dev/null
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system;


mkdir -p /etc/docker
tee -a /etc/docker/daemon.json <<EOF 1>/dev/null
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
    "max-size": "10m",
    "max-file": "10"
    }
}
EOF
```






##### Install Docker 18.09.9 and NGINX Controller Requires


```bash
yum -y install  docker-ce-18.09.1 docker-ce-cli-18.09.1 device-mapper-persistent-data lvm2 jq oniguruma socat conntrack yum-versionlock
systemctl enable --now docker
```






##### NGINX Controller Install


컨트롤러 인스톨러 패키지를 압축 해제하고, 생성된 폴더의 `./install.sh ` 파일을 실행합니다.


```bash
cd /var/tmp && tar zxvf controller-installer-<VERSION>.tar.gz
cd controller-installer
./install.sh
```


Command Argument를 사용해 준비된 커맨드를 입력하면 추가적인 정보 입력 없이 설치할 수 있습니다.


커맨드 예시는 아래와 같습니다.


```bash
### 3.10.0~
./install.sh -y --self-signed-cert --fqdn <FQDN> /
--configdb-host <DB HOST IP> --configdb-port 5432 --configdb-user <db-user> --configdb-pass <db-pass> --configdb-enable-ssl "false" \
--tsdb-volume-type local \
--smtp-host 127.0.0.1 --smtp-port 25 --smtp-authentication "false" --smtp-use-tls "false" --noreply-address noreply@itian.co.kr --admin-email <admin email> --admin-password <admin password> --organization-name itian --admin-firstname admin --admin-lastname admin


### 3.10.0~ (TSDB NFS 사용 시)
./install.sh -y --self-signed-cert --fqdn <FQDN> /
--configdb-host <DB HOST IP> --configdb-port 5432 --configdb-user <db-user> --configdb-pass <db-pass> --configdb-enable-ssl "false" \
--tsdb-volume-type nfs --tsdb-nfs-host 175.196.235.25 --tsdb-nfs-path /var/ngxtsdb \
--smtp-host 127.0.0.1 --smtp-port 25 --smtp-authentication "false" --smtp-use-tls "false" --noreply-address noreply@itian.co.kr --admin-email <admin email> --admin-password <admin password> --organization-name itian --admin-firstname admin --admin-lastname admin


### ~3.9.0
./install.sh -y --self-signed-cert --fqdn nc.itian.ml --db-host <DB HOST IP> --db-port 5432 --db-user <db-user> --db-pass <db-pass> --db-enable-ssl "false" --tsdb-volume-type local --smtp-host 127.0.0.1 --smtp-port 25 --smtp-authentication "false" --smtp-use-tls "false" --noreply-address noreply@itian.co.kr --admin-email <admin email> --admin-password <admin password> --organization-name itian --admin-firstname admin --admin-lastname admin
```


<!--stackedit_data:
eyJoaXN0b3J5IjpbMTQ3MzczMDYyNF19
-->