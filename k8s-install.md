# Kubernetes

## Setting ENV

### Turn off Selinux
```
$ sudo setenforce 0
$ sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
```

### Turn off swap
```
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
$ sudo swapoff -a
```

### Iptable bridged traffic

sysctl
```
$ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
$ sudo sysctl --system
```

### Firewall PORT

- MASTER 
```
6443            Kubernetes API 
2379 - 2380     etcd API
10250           Kubelet API
10251           kube-scheduler
10252           kube-controller-manager
8443            haproxy
```

- WORKER
```
10250           Kubelet API
30000-32767     NodePort
```

#### Install
```
$ sudo firewall-cmd --permanent --add-port={6443,30000-32767,2379-2380,10250,10251,10252,8443}/tcp
$ sudo firewall-cmd --reload
```

### Configure Kubernetes Repository

```
$ cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

---

## Install tools

### kubeadmin、kubelet、kubectl
> kubeadmin (required): bootstrap cluster
> 
> kubelet (required): start pods & containers
>
> kubectl (optional): cluster command line

```
# yum install kubeadm kubelet kubectl --disableexcludes=kubernetes

# systemctl enable kubelet --now
```

### CRI
> Container Runtime Interface

CRI-O
> CRI-O version need same with kubernetes version

```
# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
sudo yum install cri-o

sudo systemctl daemon-reload
sudo systemctl start crio
```

---

## Set Hostname on Nodes
e.g. 
* 10.20.60.56 dtsak8smaslg1
* 10.20.60.57 dtsak8smaslg2
* 10.20.60.58 dtsak8smaslg3
* 10.20.60.61 dtsak8swrklg1
* 10.20.60.62 dtsak8swrklg2
* 10.20.60.63 dtsak8swrklg3

> pod-network-cidr: 192.168.0.0/16

```
# vi /etc/hosts

10.20.60.56     dtsak8smaslg1
10.20.60.57     dtsak8smaslg2
10.20.60.58     dtsak8smaslg3
10.20.60.61     dtsak8swrklg1
10.20.60.62     dtsak8swrklg2
10.20.60.63     dtsak8swrklg3
```

## Haproxy & Keepalived

### Install
```
# yum install haproxy keepalived
```

### Setting

#### Haproxy
> All master node add `/etc/haproxy/haproxy.cfg`

```
# cd /etc/haproxy/
# cp haproxy.cfg haproxy.cfg.orig
# cat >> haproxy.cfg <<END
frontend kubernetes
    bind 10.20.60.56:8443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes ====> 這個是hostname

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server master01 10.20.60.56:6443 check fall 3 rise 2
    server master02 10.20.60.57:6443 check fall 3 rise 2
    server master03 10.20.60.58:6443 check fall 3 rise 2
END
```

verification haproxy:

```
# systemctl status haproxy
# yum install nc
# nc -v 10.20.60.56:6443
```

#### Keepalived
> IS'S NEED???

START haproxy
```
# systemctl enable haproxy --now
# systemctl enable keepalived --now
```

## Certificates
> Generate the self-signed Kubernetes CA to provision identities for other Kubernetes components

### openssl 

openssl can manually generate certificates for your cluster.

Generate a ca.key with 2048bit:

```
# openssl genrsa -out ca.key 2048
```

According to the ca.key generate a ca.crt (use -days to set the certificate effective time):

```
# openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt
```

Generate a server.key with 2048bit:

```
# openssl genrsa -out server.key 2048
```

Create a config file for generating a Certificate Signing Request (CSR). Be sure to substitute the values marked with angle brackets (e.g. <MASTER_IP>) with real values before saving this to a file (e.g. csr.conf). Note that the value for MASTER_CLUSTER_IP is the service cluster IP for the API server as described in previous subsection. The sample below also assumes that you are using cluster.local as the default DNS domain name.

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = <country>
ST = <state>
L = <city>
O = <organization>
OU = <organization unit>
CN = <MASTER_IP>

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = <MASTER_IP>
IP.2 = <MASTER_CLUSTER_IP>

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```

Generate the certificate signing request based on the config file:

```
# openssl req -new -key server.key -out server.csr -config csr.conf
```

Generate the server certificate using the ca.key, ca.crt and server.csr:

```
# openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
-CAcreateserial -out server.crt -days 10000 \
-extensions v3_ext -extfile csr.conf
```

View the certificate:

```
# openssl x509  -noout -text -in ./server.crt
```

Finally, add the same parameters into the API server start parameters.


## Initializing the Kubernetes Cluster

```
# export kubever=$(kubectl version | base64 | tr -d '\n')

# cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: {$kubever}
controlPlaneEndpoint: "10.20.60.57:8443" ===> IP or Hostname ("dtsak8smaslg1:6443")
networking:
  podSubnet: "192.168.0.0/16" ===> Calico default CIDR
EOF
```

Init 
```
# kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs
```

kubeadmin join command sample：

> MASTER
```
kubeadm join 10.20.60.56:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --experimental-control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
```

> WORKER
```
kubeadm join 10.20.60.56:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```

> Check listen port
```
netstat -ntlp
```

## Join worker nodes to Kubernetes cluster

This command will create a bootstrap token for you. You can specify the usages for this token, the "time to live" and an optional human friendly description.

```
# kubeadm join 10.20.60.56:6443 --token [TOKEN] --discovery-token-ca-cert-hash sha256:[SHA256]
```
TIPS: 10.20.60.56:6443 ----> 是不是也可以用 hostname:6443


> --token
```
# kubeadm token generate
  b0f7b8.8d1767876297d85c
```

> --discovery-token-ca-cert-hash
```
# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

## Calico CNI plugin
> install only on Master 1

```
# wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
# sed -i 's/192.168.0.0\/16/10.244.0.0\/16/g' calico.yaml => use default, no need to edit
# kubectl apply -f calico.yaml
```

## Try to get pods & nodes
```
# kubectl get pods --all-namespaces
# kubectl get nodes
```

---

## Extention

### kubec auto completion

習慣了 linux shell 的 auto completion 功能的朋友，一定愛死這個功能(按 Tab 鍵就能將命令行自動補全)。這個在 kubectl 上面也同樣可以啓用：

```
# yum install bash-completion
# echo 'source <(kubectl completion bash)' >> ~/.bashrc
# source ~/.bashrc
```
從此，所有 kubeclt 後面的命令就盡情的敲 Tab 鍵吧~~