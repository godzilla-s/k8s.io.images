## kubeadm 

环境： centOS 7.x   

执行前需要操作:
1. 关闭防火墙
```
systemctl stop firewalld
systemctl disable firewalld
```
2. 修改内核参数
```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
sysctl -p /etc/sysctl.d/k8s.conf
或 
sysctl --system

3. 关闭Selinux
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce=0
```

4. 关闭swap
```
swapoff -a
echo "vm.swappiness=0" >> /etc/sysctl.d/k8s.conf
```


### 安装kubelet, kubeadm, kubectl
#### 由于墙的原因，需要下载软件，安装软件源：
cat <<EOF > /etc/yum.repos.d/kubernetes.repo 
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
1. 可以通过`yum`安装 
```
yum install -y kubelet-1.14.1 kubeadm-1.14.1 kubectl-1.14.1
```
2. 或者手动下载，再配置/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/ --cni-bin-dir=/opt/cni/bin"
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```


sed -e 's/KUBELET_CGROUP_ARGS=--cgroup-driver=systemd/KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs/' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

### 需要注意事项
```
systemctl enable docker.service  # 这个要设置，不然会警告
systemctl enable kubelet 
```
```
vi /etc/hosts 
#设置域名映射
```

### 启动k8s master 
```
kubeadm init --kubernetes-version=v1.14.1 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.77.210
```
--pod-network-cidr: 指定Pod要使用的网段, 建议选用10.244.0.0/16<br>
--kubernetes-version: k8s版本<br>

kubeadm token list 

### 安装flannel网络插件 
```
kubectl apply -f kube-flannel.yaml
```

### 删除节点
```
kubectl delete node xxx (kubectl get node)
```
```
kubeadm reset -f #清理环境
```

### docker 安装
yum install -y yum-utils device-mapper-persistent-data lvm2
添加源：
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
查看仓库版本信息
yum list docker-ce.x86_64  --showduplicates |sort -r
yum makecache fast
yum install -y --setopt=obsoletes=0 docker-ce-18.06.3.ce-3.el7

systemctl enable docker 

### k8s-v1.14.1对应的版本:
k8s.gcr.io/kube-apiserver:v1.14.1
k8s.gcr.io/kube-controller-manager:v1.14.1
k8s.gcr.io/kube-scheduler:v1.14.1
k8s.gcr.io/kube-proxy:v1.14.1
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
// 网络插件
quay.io/coreos/flannel:v0.11.0-amd64


###  加入其它节点
kubeadm join 192.168.77.210:6443 --token lcd0kh.cek3k23mc0ke0yla --discovery-token-ca-cert-hash sha256:7686a331bba0ce89b50e860e605f8bac077df85d17ee0b320fe4aa157d13c51a



### helm 安装 
kubectl create serviceaccount --namespace=kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

helm init --upgrade -i gcr.io/kubernetes-helm/tiller:v2.14.2 --service-account=tiller --stable-repo-url https://apphub.aliyuncs.com/

helm init --upgrade -i registry.cn-shenzhen.aliyuncs.com/zuvakin/tiller:v2.14.2 --service-account=tiller --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

### 几个好一点的官方chart仓库 
1. 微软
helm repo add msapp http://mirror.azure.cn/kubernetes/charts/  
2. 阿里
helm repo add aliapp https://apphub.aliyuncs.com/
3. 

### 获取token 
kubeadm token create --ttl 0 # 生成一个永不过期的 token
### 获取discovery-token-ca-cert-hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

### 查看当前可用的api版本
```
kubectl api-version
```
