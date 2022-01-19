# Kurulum
**Kubernetes Kurulum** konusuyla ilgili dosyalara buradan erişebilirsiniz.

## kubeadm kurulum

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

**0:** sanal makine oluşturma
```
multipass launch --name master -c 2 -m 2G -d 10G
multipass launch --name node1 -c 2 -m 2G -d 10G
```

**1:** Iptables bridged traffic ayarı

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

```
sudo sysctl --system
```

**2:** containerd kurulumu

```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

```
sudo modprobe overlay
sudo modprobe br_netfilter
```

```
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

```
sudo sysctl --system
```

```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install containerd -y
sudo mkdir -p /etc/containerd
sudo su -
containerd config default | tee /etc/containerd/config.toml
exit
sudo systemctl restart containerd
```

**3:** kubeadm kurulumu


```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```


Swap'ı tam kapatmak için aşağıdaki gibi işlem yapılması gerekiyor.
```
$ sudo swapoff -a
// swap alanı geçici olarak devre dışı kalacaktır.
$ sudo vi /etc/fstab 
// Bu dosyayı açtığımızda /swapfile şeklinde bir satır var ise bu alanı yorum alanı yapmamız gerekmektedir.
```

**4:** MasterNode üzerinde aşağıki işlemler yapılacaktır.

Aşağıdaki komutlar master node üzerinden çalıştırılması gerekiyor.

```
sudo kubeadm config images pull
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<ip> --control-plane-endpoint=<ip> --upload-certs
```

--upload-certs -> master node u high avaiblity için gerekiyor. Eğer yazılmazsa high avaiblity aktif hale gelmiyor.

sudo kubeadm config images pull -> Bu komut ile birlikte kubernetes asıl bileşenleri kuruluyor. 

sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<ip> --control-plane-endpoint=<ip> --upload-certs
Yukarıdaki komuttaki IP bölümüne master node'un ip bilgileri girilmesi gerekiyor. 

**kubeadm init komutunda aşağıdaki hata çıkarsa yapılacaklar**
Hata 1:  
  How to fix timeout at Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
  
  1) Reset cluster
```
sudo kubeadm reset
rm -rf .kube/
sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/lib/kubelet/
sudo rm -rf /var/lib/etcd
```
  2) put SELinux to permissive mode
```
setenforce 0 
```
  
3) enable net.bridge.bridge-nf-call-ip6tables and net.bridge.bridge-nf-call-iptables
```
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
modprobe br_netfilter 

cat <<EOF >  /etc/sysctl.d/kube.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

4) Add Kube repo fo kubeadm, kubelet, kubectl components:
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
```
  5) Install ans start Kube components ans services:
```
yum update && yum upgrade && yum install -y docker kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl start docker kubelet && systemctl enable docker kubelet
6)kubeadm init

kubeadm init --pod-network-cidr=10.244.0.0/16 -v=9
```
  
  
  
init komutu çalıştırıldıktan sonra aşağıdaki işlemler yapılabilecektir. 

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

  
**5:** Worker node larda çalıştırılacak komut
  
  4. bölümdeki çalıştırılan kubeadm init komutu ile oluşacak olan kubeadm join komutu node'larda çalıştırılacaktır. kubeadm join komutu control-plane için farklı worker node için farklı olarak verilmektedir. 
  
  

```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

**6:** Eğer node'lar ayaklanmaz ise aşağıdaki kodu çalıştırmak gerekecektir.

  
  Node lar arası haberleşmeyi kurmak için flannel kullanabilir. Fakat internete çıkışta hata oluşturmaktadır. 
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
Son versiyonu için 

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```
  
  Node lar arası haberleşmede calico daha sağlıklı çalışmaktadır. 
  
  curl https://docs.projectcalico.org/manifests/calico.yaml -O
  kubectl apply -f calico.yaml
  
  curl https://docs.projectcalico.org/manifests/calico-etcd.yaml -o calico_etcd.yaml
  kubectl apply -f calico_etcd.yaml

**7:** LoadBalancer için MetalLB'yi kullanacağız

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250

```
192.168.1.240-192.168.1.250 -> Varolan networkteki ip aralığının girilmesi gerekmektedir.
