# Instalação k8s 1.24.1 usando o CRI-O em Distribuição baseada em Debian

> **Lembrando:** _Todos os comandos deste tutorial esta sendo executando com o usuário **root**_


## Atualizando o servidor e instalando ferramentas basicas

````bash
apt-get update && apt-get upgrade -y && apt-get autoremove -y 
apt-get install -y apt-transport-https ca-certificates curl vim
````

## instalando o CRI-O

### Preparando a máquina para o CRI-O

Pré requisito, instale o libseccomp2:

````bash
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 04EE7237B7D453EC
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 648ACFD622F3D138

echo 'deb http://deb.debian.org/debian buster-backports main' > /etc/apt/sources.list.d/backports.list
apt update
apt install -y -t buster-backports libseccomp2 || apt update -y -t buster-backports libseccomp2
````
### Instalando o CRI-O

Precisa informar em qual Distribuição voce esta usando conform a tabela:
| Sistem Operacional |$OS             |
|--------------------|:--------------:|
| Debian Unstable    | Debian_Unstable|
| Debian Testing     | Debian_Testing |
| Debian 11          | Debian_11      |
| Ubuntu 22.04       | xUbuntu_22.04  |
| Ubuntu 20.04       | xUbuntu_20.04  |
| Ubuntu 19.10       | xUbuntu_19.10  |
| Ubuntu 19.04       | xUbuntu_19.04  |
| Ubuntu 18.04       | xUbuntu_18.04  |

Deve ser informado uma variavel de ambiente `$OS` para defivir qual o sua distribuição.
A versão do CRI-O também precisa ser informando na variavel de ambiente `VERSION_CRIO`.

````bash     
export OS=xUbuntu_20.04
export VERSION_CRIO=1.24

echo "deb [signed-by=/usr/share/keyrings/libcontainers-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb [signed-by=/usr/share/keyrings/libcontainers-crio-archive-keyring.gpg]  https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION_CRIO/$OS /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION_CRIO.list


mkdir -p /usr/share/keyrings
curl -L "https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key" | gpg --dearmor -o /usr/share/keyrings/libcontainers-archive-keyring.gpg
curl -L "https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION_CRIO/$OS/Release.key" | gpg --dearmor -o /usr/share/keyrings/libcontainers-crio-archive-keyring.gpg

apt-get update
apt-get install cri-o cri-o-runc

systemctl daemon-reload
systemctl enable crio
systemctl start crio
````

## instalando o Kubernetes 

Agora podemos instalar o kubernetes para todas as maquinas que serão do cluster:

````bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
br_netfilter
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
ip_vs
EOF

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1
EOF

sysctl --system

curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
````
### configuração do systemd no cgoup kubelet

Edite o arquivo /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf conforme como:
````bash
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
## The following line to be added for CRI-O 
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"  <----- AQUI
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS $KUBELET_CGROUP_ARGS <----- AQUI
````
Reinicie o kubelet e o CRI-O
````bash
systemctl restart crio 
systemctl restart kubelet 
````

## Configurando o nó master

````bash
kubeadm config images pull
kubeadm init  --pod-network-cidr=10.244.0.0/16
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml


````

## condfurando o nó workers

````bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables = 1
EOF
````