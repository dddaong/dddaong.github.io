---
title: "K3S Installation for SingleNode Cluster Guide"
author: DDAONG
---

# K3S Installation Guide

K3S는 [K3S](https://k3s.io/) 홈페이지에서 제공하는 Script를 실행하는 방법으로 간단하게 구축이 완료됩니다.


다만, 이 Script를 통해 설치를 수행하면, 자동으로 Flannel CNI를 사용하거나, 
Ingress Controller로 Traefik이 설치 되는 등, 익숙하지 않은 Default Option이 설정되어 있습니다.


그래서 Default Option을 사용하지 않고, 아래와 같이 변경하여 설치하려고 합니다.

- Use Docker as CRI

- Disable Flannel CNI / Configure Calico CNI

- Disable Traefik Ingress Controller


편의를 위해 아래 서비스를 설치 추가로 설치합니다.

- Cert-manager (Let's Encrypt - 무료 SSL Cert 발급)

- NGINX Ingress Controller 



## K3s Installation with docker

```bash
SVRNAME="itian.ddaong.ga"
sudo hostnamectl set-hostname ${SVRNAME}

mkdir -p /etc/rancher/k3s/
cat << EOF > /etc/rancher/k3s/config.yaml
write-kubeconfig-mode: 644
token: "secret"
cluster-cidr: 10.10.0.0/16
service-cidr: 10.40.0.0/16
EOF
```

```bash
# Install Docker 19.03 for k3s
curl https://releases.rancher.com/install-docker/19.03.sh | sh
sudo usermod -aG docker root

### K3s Install
# rootless / no CNI / cidr 변경 / no Traefik / Use Docker
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="--flannel-backend=none --disable-network-policy --cluster-cidr='10.10.0.0/16' --service-cidr='10.20.0.0/16' --disable=traefik --docker" sh -

# Calico CNI Install
curl -O https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml


sed -i s/\#\ \-\ name\:\ CALICO_IPV4POOL_CIDR/\-\ name\:\ CALICO_IPV4POOL_CIDR/g calico.yaml
sed -i s/\#\ \ \ value\:\ \"192.168.0.0\/16\"/\ \ value\:\ \"10.10.0.0\/16\"/g

kubectl apply -f calico.yaml

## script Enable IP Forwarding to avoid -
## Klipper SVCLB CrashLoopback Error 
if ! grep -q "allow_ip_forwarding" /etc/cni/net.d/10-calico.conflist; then
addlinehere=$(grep  -n 'policy' /etc/cni/net.d/10-calico.conflist | cut -d ':' -f 1)
sed -i "${addlinehere}i \ \ \ \ \ \ \"container_settings\"\: \{\"allow_ip_forwarding\"\:\ true\}\," /etc/cni/net.d/10-calico.conflist
fi

# kubectl Bash Auto-Completion
yum -y install bash-completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

```


>설치 스크립트의 `--cluster-cidr='10.10.0.0/16'` 옵션은 Flannel CNI에 적용되는 부분으로, 이것만으로는 효과가 없습니다. 
>Calico를 사용하는 환경에서는 Calico YAML 파일에서 옵션을 수정해 줘야 합니다.

> LoadBalancer Type Service를 설정하게 되면 Klipper 서비스에서도 아래의 로그와 함께 CrashLoopback Error가 발생할 수 있습니다.  
```bash
+ cat /proc/sys/net/ipv4/ip_forward
+ '[' 1 '!=' 1 ]
```
> 아래 라인을 추가해주는 방법으로 에러를 해결할 수 있습니다.
> `"container_settings": {"allow_ip_forwarding": true},`
> 관련 Cmdline은 설치 방법에 포함되어 있습니다.


#### NGINX Ingress Controller
[NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/deploy/)
```bash
curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml

mv deploy.yaml nginx-ingress-deploy.yaml

kubectl apply -f nginx-ingress-deploy.yaml
```


#### Cert Manager Installation
[Cert-Manager](https://cert-manager.io/)는 Ingress Resource의 Annotation을 확인해 인증과정을 자동으로 수행합니다.
샘플 Yaml 파일을 설치 과정 아래 첨부합니다.

```bash
curl -LO https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml

kubectl apply -f cert-manager.yaml
```

> - Annotation 중 cert-manager.io/cluster-issuer 값이
> -  letsencrypt-staging이면 인증 과정없이 Fake 인증서를 생성해주고,
> - letsencrypt-staging이면 인증 과정을 거쳐 정식인증서를 발급해줍니다.
  
  
   
아래는 다른 서비스 추가시 활용할 수 있는 Ingress Sample YAML 입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: file-ddaong-ga
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/backend-protocol: HTTP
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
      - file.ddaong.ga
      secretName: file-ddaong-ga-tls-prod
  rules:
  - host: file.ddaong.ga
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: filebrowser
            port:
              number: 8080
```


