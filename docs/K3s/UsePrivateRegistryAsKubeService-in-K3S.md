# K3s Installation with Private Registry

#### This is For Multi Node Cluster
```bash
SVRNAME="itian.ddaong.ga"
sudo hostnamectl set-hostname ${SVRNAME}

# Install Docker for k3s
curl https://releases.rancher.com/install-docker/19.03.sh | sh
sudo usermod -aG docker root

# Preparation for Private Registry So K3S referring to this conf file
## Run before K3S Installation
cat << EOF>/etc/rancher/k3s/registries.yaml
mirrors:  
  "${SVRNAME}":
    endpoint:
    - "http://${SVRNAME}"
EOF
```

K3S 설치 수행

```bash
### K3s Install
# rootless / no CNI / Change Cluster, Service cidr / no Traefik / Use Docker
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="--flannel-backend=none --disable-network-policy --cluster-cidr=10.10.0.0/16 --service-cidr=10.20.0.0/16 --disable=traefik --docker" sh -

# kubectl 자동완성
yum -y install bash-completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Calico CNI Install
curl -O https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
kubectl apply -f calico.yaml

```


```bash
PRIVREGDOMAIN="reg.itian.ddaong.ga"
## Private Registry on Kubernetes
mkdir -p /k3s/registry/ && cd /k3s/registry/

cat <<EOF>kube-registry.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kube-registry-v0
  namespace: kube-system
  labels:
    k8s-app: kube-registry
    version: v0
spec:
  replicas: 1
  selector:
    k8s-app: kube-registry
    version: v0
  template:
    metadata:
      labels:
        k8s-app: kube-registry
        version: v0
    spec:
      containers:
      - name: registry
        image: registry:2
        resources:
          limits:
            cpu: 100m
            memory: 200Mi
        env:
        - name: REGISTRY_HTTP_ADDR
          value: :5000
        - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
          value: /var/lib/registry
        volumeMounts:
        - name: image-store
          mountPath: /var/lib/registry
        ports:
        - containerPort: 5000
          name: registry
          protocol: TCP
      volumes:
      - name: image-store
        hostPath:
          path: /var/lib/registry-storage
        #emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: kube-registry
  namespace: kube-system
  labels:
    k8s-app: kube-registry
    kubernetes.io/name: "KubeRegistry"
spec:
  selector:
    k8s-app: kube-registry
  ports:
  - name: registry
    port: 5000
    targetPort: 5000
    protocol: TCP
  type: LoadBalancer
---
apiVersion: networking.k8s.io/v1
kind: Ingress  
metadata:
  name: docker-registry-ingress  
  annotations:  
    kubernetes.io/ingress.class: "traefik"  
spec:  
  rules:  
  - host: ${PRIVREGDOMAIN}
    http:  
      paths:  
      - path: /
        backend:
          serviceName: docker-registry-service
          servicePort: 5000
---
EOF

kubectl apply -f kube-registry.yaml
```

