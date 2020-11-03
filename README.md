<br />
<p align="center">
  <a href="https://github.com/othneildrew/Best-README-Template">
    <img src="assets/logo_kube.png" alt="Logo" width="300">
  </a>

  <h3 align="center">Kubernetes Stack Config (Single Node Master)</h3>
</p>

## Stack 
* Docker
* K8S
* Dashboard
* Grafana
* Prometheus

### S.O
- CentOS 7.0 x64

### Installation Dependences

```sh
sudo yum clean all
sudo yum update -y
sudo yum groupinstall 'Development Tools'
sudo yum install wget
sudo yum install bash-completion bash-completion-extras
```

### Desabilitando SELINUX & Firewall
```sh
sudo setenforce 0
sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

### Desabilitando SWAP
```sh
sudo swapoff -a
vim /etc/fstab
```
- Devemos comentar(#) a seguinte linha para não habilitar mais o swap
- #/dev/mapper/centos-swap swap swap defaults 0 0
### Configuração de módulos de kernel
```sh
sudo vim /etc/modules-load.d/k8s.conf
```
-Acrescentar as seguintes linhas de modulos:

  - br_netfilter
  - ip_vs
  - ip_vs_rr
  - ip_vs_sh
  - ip_vs_wrr
  - nf_conntrack_ipv4

### Installation Docker & Kubernetes

```sh
curl -fsSL https://get.docker.com | bash
sudo systemctl enable docker
sudo systemctl start docker
# cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

sudo mkdir -p /etc/systemd/system/docker.service.d
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo groupadd docker
sudo usermod -aG docker $USER

```


- Criar arquivo para o repositorio do CentOS para adicionar o repo do K8S
```txt

[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

- Salvar o Conteúdo acima em :
```sh
vim /etc/yum.repos.d/kubernetes.repo
```

- Instalando K8S
```sh
sudo yum install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet && sudo systemctl start kubelet
```

-Configurar alguns parâmetros de kernel no sysctl

Em vim /etc/sysctl.d/k8s.conf
```txt
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

- Iniciando K8S
```sh
sudo sysctl --system
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

- Iniciando o Master Node
```sh
kubeadm init --apiserver-advertise-address $(hostname -i)
```

- Terminando Configuração dando permissão ao usuario
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Instalando Pod Network
```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
kubectl get pods -n kube-system
```


### Dashboard K8S

### Criando SSL assinado com Let's Encrypt 
```sh
sudo yum install epel-release
sudo yum install certbot
certbot certonly --standalone -d meudominio.com --staple-ocsp -m email@seudominio.com --agree-tos
```

### Certificando DashBoard Kubernetes
- Necessário criar o diretorio /certs e dentro da pasta copiar os certificados com extensão .crt e .key
```sh
kubectl create namespace kubernetes-dashboard
kubectl create secret generic kubernetes-dashboard-certs --from-file=/certs -n kubernetes-dashboard
```
- Implantação do DASHBOARD K8S
- Necessário fazer a seguinte alteração no YAML recomended do DashBoard
- Substituir --auto-generate-certificate e colocar as seguintes linhas em seu lugar:
```sh
  - --tls-cert-file=dashboard.crt
  - --tls-key-file=dashboard.key
```
-Atenção não executar diretamente esse YAML, pois a certificação não irá funcionar sem a substituição acima, causando problemas no deploy do POD
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

- Definindo do IP Externo para o serviço dashboard
```sh
kubectl patch svc -n kubernetes-dashboard kubernetes-dashboard -p '{"spec":{"externalIPs":["x.x.x.x"]}}'
```

- Criar usuário e dar permissão para o token gerenciar o DASHBOARD

```sh
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

```sh
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

- Adquirir o token
```sh
kubectl describe secret admin-user-token-lsjgt -n kubernetes-dashboard
```

### Configuração Extra
- Permitir com o Master Node instale YAMLS nele.
```sh
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### Instando Helm 3.0
- Download em https://helm.sh/
```sh
tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```
### Instalando GRAFANA
```sh 
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana --namespace=monitoring
kubectl patch svc grafana-svc -n monitoring  -p '{"spec":{"externalIPs":["x.x.x.x"]}}'
```

### Instalando PROMETHEUS
```sh 
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-prom-release bitnami/prometheus-operator
kubectl patch svc prometheus-svc -n monitoring  -p '{"spec":{"externalIPs":["x.x.x.x"]}}'
```
## Enable kubectl autocompletion (run shell with sudo su)

```sh
echo 'source <(kubectl completion bash)' >>~/.bashrc
kubectl completion bash >/etc/bash_completion.d/kubectl
```

