# 1. kubernates 集群搭建手册

- [1. kubernates 集群搭建手册](#1-kubernates-集群搭建手册)
  - [1.1. 创建虚拟机](#11-创建虚拟机)
  - [1.2. 开放虚拟机端口](#12-开放虚拟机端口)
  - [1.3. 操作系统环境准备](#13-操作系统环境准备)
  - [1.4. 安装docker](#14-安装docker)
  - [1.5. 安装kubectl](#15-安装kubectl)
  - [1.6. 开始安装 kubectl kubelet kubeadm](#16-开始安装-kubectl-kubelet-kubeadm)
  - [1.7. 部署Master](#17-部署master)
  - [1.8. 安装Dashboard](#18-安装dashboard)

## 1.1. 创建虚拟机

| Role   | Name        | Comment |
| ------ | ----------- | ------- |
| msater | k8smaster01 |         |
| node 1 | k8snode01   |         |
| node 2 | k8snode02   |         |
| node 3 | k8snode03   |         |

## 1.2. 开放虚拟机端口

- Master Port

| Port      | Name                       | Comment |
| --------- | -------------------------- | ------- |
| 6443      | KubernetesAPIServer        | 1       |
| 2379-2380 | EtcdServerClientAPI        | 2       |
| 10250     | KubeletAPI                 | 3       |
| 10251     | KubeScheduler              | 4       |
| 10252     | KubeControllerManager      | 5       |
| 10255     | ReadOnlyKubeletAPIHeapster | 6       |

- Node Port

| Port        | Name                       | Comment |
| ----------- | -------------------------- | ------- |
| 30000-32767 | NodePortServices           |         |
| 10250       | KubeletAPI                 |         |
| 10255       | ReadOnlyKubeletAPIHeapster |         |

## 1.3. 操作系统环境准备

1. CentOS系统升级命令

    ```sh
    sudo yum update
    ```

2. 如果开启了 swap 分区，kubelet 会启动失败(可以通过将参数 --fail-swap-on 设置为false 来忽略 swap on)，故需要在每台机器上关闭 swap 分区：

    ```sh
        sudo swapoff -a
    ```

3. 关闭 SELinux，否则后续 K8S 挂载目录时可能报错 Permission denied
   - 手动命令：

    ```sh
        sudo setenforce 0
    ```

   - 修改配置文件，永久生效

   ```sh
        sudo vim /etc/selinux/config
        SELINUX=disabled
   ```

4. 修改network conf

   ```sh
        sudo vim /etc/sysctl.d/k8s.conf
   ```

    1. 内容

   ```sh
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
   ```

5. 修改/etc/hosts

    ```sh
        sudo vim /etc/hosts
    ```

| IP         | host name    |
| ---------- | ------------ |
| 172.16.0.4 | k8s-master01 |
| 172.16.0.6 | k8s-node01   |
| 172.16.0.5 | k8s-node02   |
| 172.16.0.7 | k8s-node03   |

172.16.0.4  k8s-master
172.16.0.6  k8s-node-1
172.16.0.5  k8s-node-2
172.16.0.7  k8s-node-3

## 1.4. 安装docker

- [CentOS Docker 安装](https://www.runoob.com/docker/centos-docker-install.html)

    ```markdown
        Docker 支持以下的 64 位 CentOS 版本：
        - CentOS 7
        - CentOS 8

    ```

    &emsp; 使用国内 daocloud 一键安装命令：

    ```sh
        curl -sSL https://get.daocloud.io/docker | sh
    ```

- 安装 Docker Engine-Community
  
    &emsp;选择国内的一些源地址(阿里云):

    ```sh
        sudo yum-config-manager \
            --add-repo \
            http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    ```


    ```sh
    sudo yum install docker-ce-20.10.6 docker-ce-cli-20.10.6 containerd.io
    ```

- 启动 Docker:

    ```sh
    sudo systemctl start docker

    ```

- 通过运行 hello-world 映像来验证是否正确安装了 Docker Engine-Community 

    ```sh
        sudo docker run hello-world
    ```

- 配置docker 国内镜像

    ```sh
        sudo mkdir -p /etc/docker

        sudo tee /etc/docker/daemon.json << -'EOF'
        {
        "registry-mirrors": ["https://t3e5gxr9.mirror.aliyuncs.com"]
        }
        EOF

        sudo systemctl daemon-reload
        sudo systemctl restart docker

    ```

## 1.5. 安装kubectl

- kubectl是什么

&emsp;&emsp; kubectl是一款用于运行Kubernetes集群命令的管理工具。

1. 添加Kubernetes的yum源

    ```sh
        cat >> /etc/yum.repos.d/kubernetes.repo <<EOF
        [kubernetes]
        name=Kubernetes
        baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
        enabled=1
        gpgcheck=1
        repo_gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
        EOF
    ```

2. 查看可安装的版本

   ```sh
        yum list kubectl –showduplicates -y
        已加载插件：fastestmirror
        base                                                     | 3.6 kB     00:00
        docker-main                                              | 2.9 kB     00:00
        elrepo                                                   | 2.9 kB     00:00
        epel/x86_64/metalink                                     | 5.0 kB     00:00
        epel                                                     | 4.7 kB     00:00
        extras                                                   | 3.4 kB     00:00
        kubernetes                                               | 1.3 kB     00:00
        updates                                                  | 3.4 kB     00:00
        (1/5): epel/x86_64/group_gz                                | 266 kB   00:01
        (2/5): epel/x86_64/updateinfo                              | 851 kB   00:00
        (3/5): kubernetes/primary                                  | 6.0 kB   00:01
        (4/5): updates/7/x86_64/primary_db                         | 3.6 MB   00:03
        (5/5): epel/x86_64/primary_db                              | 6.1 MB   00:06
        Loading mirror speeds from cached hostfile
        * base: mirrors.neusoft.edu.cn
        * elrepo: mirrors.tuna.tsinghua.edu.cn
        * epel: mirrors.tongji.edu.cn
        * extras: mirrors.neusoft.edu.cn
        * updates: mirrors.aliyun.com
        kubernetes                                                                49/49
        可安装的软件包
        kubectl.x86_64                        
   ```

3. yum方式安装kubectl

   ```sh
       sudo yum install -y kubectl.x86_64
   ```

4. 检测kubectl配置(TODO)

    ```sh
        kubectl version
        kubectl cluster-info
        kubectl cluster-info dump
        
    ```

5. 创建配置目录

    ```sh
        mkdir -p ~/.kube
    ```

6. 创建配置文件

    ```sh
        touch ~/.kube/config
    ```

7. 修改配置文件(TODO)

    ```sh
        vim ~/.kube/config
    ```

8. (TODO)登录 master 节点，将 /etc/kubernetes/admin.conf 文件拷贝到本地计算机 $HOME/.kube/config ，此路径是 kubectl 客户端 --kubeconfig 配置参数的默认路径。如果用户把证书放在了其他位置，那么每次执行 kubectl 命令都需要设置 --kubeconfig=/path/to/kubeconfig 。 示例：

    ```sh
        scp root@192.168.100.11:/etc/kubernetes/admin.conf ~/.kube/config
    ```
scp testadmin@172.16.0.4:/etc/kubernetes/admin.conf ~/.kube/config

&emsp;&emsp;k8s 1.12 中的目录是  /etc/kubernetes/bootstrap.kubeconfig


## 1.6. 开始安装 kubectl kubelet kubeadm 

1. 开始安装

    ```sh
        sudo yum install kubectl kubelet kubeadm && sudo systemctl enable kubelet && sudo systemctl start kubelet
    ```


## 1.7. 部署Master

1. 初始化k8s集群

    > - 这里的192.168.10.108改为自己的IP地址
    > - 这里kubernetes-version改为当前安装的版本

    ```sh
        sudo kubeadm init --kubernetes-version=1.21.0 --apiserver-advertise-address=172.16.0.4 --image-repository registry.aliyuncs.com/google_containers --service-cidr=10.10.0.0/16 --pod-network-cidr=10.122.0.0/16

    ```

2. 这里可能会遇到某个模块加载不上的可能

```sh
    sudo docker pull coredns/coredns:1.8.0
    sudo docker tag coredns/coredns:1.8.0 registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0
```

3. 部署CNI插件

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl get pods -n kube-system
```

4. 验证节点状态

```sh

    kubectl get nodes -o wide

```



5. Docker 镜像加速设置：

https://cr.console.aliyun.com/cn-wulanchabu/instances/mirrors

## 1.8. 安装Dashboard

1. github官方文档[链接](https://github.com/kubernetes/dashboard)

2. 
```sh 

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml

```

