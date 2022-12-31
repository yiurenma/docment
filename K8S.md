
> 此文档记录基于阿里云ubuntu操作系统搭建k8s集群的全部过程，基于瑞通学习平台相关视频进行归纳总结。
>
> 第一次试验大概花费不到2块钱，5个小时左右时间。

## 1.2 基础环境配置

### 1.2.1 硬件环境

机器配置：2核CPU，8G内存，40G系统盘

系统：Ubuntu 16.04.6 LTS

机器数量：3台 （master01, node01, node02） 

### 1.2.2 软件环境

#### 1.2.2.1 操作系统配置

```bash
### hostname 修改
hostnamectl set-hostname master01
hostnamectl set-hostname node01
hostnamectl set-hostname node02

### hosts 表
vi /etc/hosts
#kubernetes
172.28.74.37	master01
172.28.74.39	node01
172.28.74.38	node02
```

#### 1.2.2.2 Dokcer 安装与配置

```bash
### apt 包索引更新
apt-get update

### 安装软件包以允许 apt 通过 https 使用存储库
apt-get -y install apt-transport-https ca-certificates curl gnupg-agent software-properties-common 

### 添加 Docker 的官方 GPG 秘钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

### 检索指纹的后8个字符，验证现在是否拥有带有指纹的秘钥
sudo apt-key fingerprint 0EBFCD88

### 安装 add-get-repository 工具
apt-get -y install software-properties-common

### 添加稳定的存储库
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

### apt 包索引更新
apt-get update

### 查看 Docker 版本
apt-cache madison docker-ce

### 安装 docker-ce-5.20.10.8~3-0-ubuntu-focal
apt-get -y install docker-ce=5:20.10.8~3-0~ubuntu-focal docker-ce-cli=5:20.10.8~3-0~ubuntu-focal containerd.io
docker info

### 解决问题：WARNING：No swap limit support操作系统下 docker 不支持内存限制的警告
在基于 RPM(centos) 的系统上不会出现此警告，该系统默认下启用这些功能
vim /etc/default/grub 添加或编辑 GRUB_CMDLINE_LINUX 行以添加这两个键值对 "cgroup_enable=memory swapaccount=1",
最终效果：
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1 net.ifnames=0 console=tty0 console=ttyS0,115200n8 noibrs"

### 执行命令更新 grub 并重启机器
update-grub && reboot

### Docker 在 1.13 版本之后，将系统 iptables 中 FORWARD 链的默认策略设置为 DROP，并为连接到 docker0 网桥的容量添加了 ACCETP 规则。但是 kubernetes 因为频繁使用 iptables ，如果策略是 DROP，则会导致很多网络问题。
临时解决办法（重启会导致此更改失效）： iptables -P FORWARD ACCEPT
永久解决办法： vim /lib/systemd/system/docker.service 
在 [Service] 下添加：ExecStartPost=/sbin/iptables -P FORWARD ACCEPT
systemctl daemon-reload && systemctl restart docker.service 

### 设置 daemon.json
cat > /etc/docker/daemon.json <<EOF
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts":
    {
        "max-size": "100m"
    },
    "storage-driver": "overlay2"
}
EOF
systemctl daemon-reload && systemctl restart docker.service 
```

#### 1.2.2.3 K8s 安装与配置

```bash
### 配置 apt 库，安装 kubeadm, kubelet, kubectl 
sudo apt-get update && sudo apt-get install -y apt-transport-https curl 
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - 

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list 
deb https://apt.kubernetes.io/ kubernetes-xenial main 
EOF

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl 
sudo apt-mark hold kubelet kubeadm kubectl
```

## 1.3 k8s 初始化

### 1.3.1 master 初始化

```bash
### 命令详解
kubeadm config upload from-file: 由配置文件上传到集群中生成 ConfigMap
kubeadm config upload from-flags: 由配置文件生成 ConfigMap 
kubeadm config view: 查看当前集群中的配置值
kubeadm config print init-defaults: 输出 kubeadm init 默认参数文件的内容
kubeadm config print join-defaults: 输出 kubeadm join 默认参数文件的内容
kubeadm config migrate: 在新旧版之间进行配置转换
kubeadm config images list: 列出所需的镜像列表
kubeadm congig images pull：拉取镜像到本地

### 初始化 control-plane 节点方式一
kubeadm init --apiserver-advertise-address=172.28.74.37 

### 初始化 control-plane 节点方式二
##生成配置文件
kubeadm config print init-defaults > init-defaults.yml

##修改 clusterName 和 apiservice 地址
vim init-defaults.yml
localAPIEndpoint:
	advertiseAddress: 172.28.74.37
clusterName: demo-cluster01

##执行初始化操作
kubeadm init --config=init-defaults.yml

执行完初始化之后将最后的结果保存在文件中：管理用户配置，部署网络，添加节点相关信息
注意：如果在初始化集群的时候出现报错，请执行 kubeadm reset 命令执行重置，解决提示的报错后再执行初始化操作

### 配置用户使用 kubernetes 集群
## root 用户
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /root/.bashrc
source /root/.bashrc
## 非 root 用户
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
sudo chown $(id -u):$(id -g) $HOME/.kube/config

### 检查组件安装是否正常
#在这个步骤中会出现 scheduler NOT Healthy 的情况，在 /etc/kubernetes/manifests/kube-controller-manager.yaml 和 /etc/kubernetes/manifests/kube-scheduler.yaml 去掉--port=0 并且执行 systemctl restart kubelet 重启 kubelet
kubectl get componentstatuses
vi /etc/kubernetes/manifests/kube-controller-manager.yaml
vi /etc/kubernetes/manifests/kube-scheduler.yaml
systemctl restart kubelet
```

#### 1.3.1.1 k8s 网络安装

```bash
### 这里面要非常注意版本的选择。此次试验中 kubernetes 的版本是 1.22.1，所以calico选择的是最新版本的 3.20。因为瑞通视频中 kubernetes 使用的是 1.17.3，所以当时的 calico 是 3.8 版本，如果这里选择的是 3.8 版本，则会出现 coredns 一直处于 containerCreating 状态中，并且 calico 相关的 pods 都会创建失败。
kubectl apply -f https://docs.projectcalico.org/v3.20/manifests/calico.yaml
或者
kubectl apply -f https://docs.projectcalico.org/v3.20/manifests/canal.yaml
```

#### 1.3.1.2 kubectl 命令补全

```bash
### 查看 completion 帮助
kubectl completion -h 

###若要将 kubectl 自动补全添加到当前 shell
source <(kubectl completion bash)

### 将 kubectl 自动补全添加到配置文件中，可以在以后的 shell 中自动加载它
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

#### 1.3.1.3 检测 kubernetes 集群是否正常 （ master01 节点执行 ）

```bash
###检查组件安装是否正常
kubectl get componentstatuses

###检查集群系统信息
kubectl get pods -n kube-system 

###查看核心组件是否运行正常 （ Running ）
kubectl get pods -n kube-system 
```

### 1.3.2 work 节点添加与配置

```bash
### token 创建 ( master01 节点执行)
kubeadm token create

### token 永久创建 ( master01 节点执行)
kubeadm token create -ttl 0

### token 查看 ( master01 节点执行)
kubeadm token list 

### 获取 discovery-token-ca-cert-bash 值 ( master01 节点执行)
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \openssl dgst -sha256 -hex | sed 's/*.* //'

### 添加 work 节点到 kubernetes 集群 ( work node 节点执行)
kubeadm join 172.28.74，23:6443 --token  2eajaq.baegmys0jmr3ia30 \ 
--discovery-token-ca-cert-hash sha256:6850dffa5f8920ade2f2abe31225ab4ef4bb9a39a502bcb7a78fe6e7ab0d8942
```
