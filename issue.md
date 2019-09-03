1. 修改网络参数:
```
[root@k8s-01 ~]# sysctl -p /etc/sysctl.d/k8s.conf
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No such file or directory
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory
```
解决方法:
安装bridge-util软件，加载bridge模块，加载br_netfilter模块
yum install -y bridge-utils.x86_64
modprobe bridge
modprobe br_netfilter

2. 下载kubectl，kubeadm,kubelet时报错:
```
[Errno 14] curl#6 - "Could not resolve host: mirrors.aliyun.com; Unknown error"
```
可能原因：DNS解析失败: vi /etc/resolv.conf
```
nameserver 8.8.8.8
nameserver 8.8.8.4
```

3. 添加docker下载源错误:
```
Could not fetch/save url https://download.docker.com/linux/centos/docker-ce.repo to file /etc/yum.repos.d/docker-ce.repo: [Errno 14] curl#6 - "Could not resolve host: download.docker.com; Unknown error"
```
j解决:
```
vi /etc/resolv.conf

nameserver 8.8.8.8
nameserver 8.8.8.4
```

4. kubeadm join 
Running pre-flight checks
```
 Failed to request cluster info, will try again: [Get https://192.168.77.210:6443/api/v1/namespaces/kube-public/configmaps/cluster-info: x509: certificate has expired or is not yet valid]
```
时间同步的原因
先下载 yum install -y ntp
??? ntpdate cn.pool.ntp.org  

5. kubeadm join
```
Unable to update cni config: No networks found in /etc/cni/net.d
Jul 20 23:08:36 k8s-02 kubelet[17444]: E0720 23:08:36.869907   17444 kubelet.go:2170] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized

 Error syncing pod aa1f58b4-ab62-11e9-aef0-000c292aba9d ("kube-proxy-s542n_kube-system(aa1f58b4-ab62-11e9-aef0-000c292aba9d)"), skipping: failed to "StartContainer" for "kube-proxy" with ImagePullBackOff: "Back-off pulling image \"k8s.gcr.io/kube-proxy:v1.14.1\""
```
需要在nod主机上安装下面镜像：注意版本
```
k8s.gcr.io/pause
quay.io/coreos/flannel
k8s.gcr.io/kube-proxy
```

6. 安装helm 
报错信息:
```
forwarding ports: error upgrading connection: error dialing backend: dial tcp 192.168.77.211:10250: connect: no route to host
```
目前不清楚为啥 
最好把helm安装在master机器上， 也就是在cluster node加入之前，把helm安装好。


7. 在master上安装，报错：
```
nodes are available: 1 node(s) had taints that the pod didn't tolerate.
```
解决：添加允许所有节点启动容器服务
kubectl taint nodes --all node-role.kubernetes.io/master-

8. 安装
```
 Failed create pod sandbox: rpc error: code = Unknown desc = failed to set up sandbox container "35002d4da88b47fde385a5cf5605d1ae565e264df724c3e2c876d75d309e2f69" network for pod "mongo-mongodb-845c844567-tkg4z": NetworkPlugin cni failed to set up pod "mongo-mongodb-845c844567-tkg4z_mongo" network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.2.1/24
```

ip link delete cni0
ip link delete flannel.1

kubeadm reset 
systemctl stop kubelet 
systemctl stop docker 
rm -rf /var/lib/cni/ 
rm -rf /var/lib/kubelet/* 
rm -rf /etc/cni/ 
ifconfig cni0 down 
ifconfig flannel.1 down 
ifconfig docker0 down

ip link delete cni0
ip link delete flannel.1
再重启 kubeadm reset, 
参考 
https://github.com/kubernetes/kubernetes/issues/39557
https://blog.csdn.net/a814902893/article/details/97273565

mount -o remount,rw '/sys/fs/cgroup'
ln -s /sys/fs/cgroup/cpu,cpuacct /sys/fs/cgroup/cpuacct,cpu

