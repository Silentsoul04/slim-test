---
kubernetes.md - 测试v500对kubernetes的兼容性及其基本功能
Hardware platform: D05，D03
Software Platform: CentOS
Author: Liu Caili <liu_caili@hoperun.com>  
Date: 2017-12-13 9:45:00 
Categories: Estuary Documents  
Remark:
---

- **Dependency:**
    - 测试环境搭建
       	- 准备4台服务器：
       	- 192.168.1.254  # k8s-master节点
       	- 192.168.1.218  # k8s-node-1节点
        - 192.168.1.233  # k8s-node-2节点


- **Test:**
    - 检查机器之间是否网络互通
    - 配置各机器的hostname
    	- 192.168.1.254: hostnamectl --static set-hostname  k8s-master
    	- 192.168.1.218: hostnamectl --static set-hostname  k8s-node-1
    	- 192.168.1.233: hostnamectl --static set-hostname  k8s-node-2
    - 配置各机器的hosts
        - 在三台机器上均执行如下命令：
            echo '192.168.1.254    k8s-master
            192.168.1.254   etcd
            192.168.1.254   registry
            192.168.1.218   k8s-node-1
            192.168.1.233   k8s-node-2' >> /etc/hosts
    - 关闭三台机器上的防火墙
         - systemctl disable firewalld.service
         - systemctl stop firewalld.service
    - 给三台机器安装etcd
         - yum install etcd -y
    - 修改k8s-master的etcd配置如下：
        - cat /etc/etcd/etcd.conf
        - ETCD_NAME=etcd
        - ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
        - ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
        - ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
        - ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd:2380"
        - ETCD_INITIAL_CLUSTER="etcd=http://etcd:2380,k8s-node-1=http://k8s-node-1:2380,k8s-node-2=http://k8s-node-2:2380"
        - ETCD_INITIAL_CLUSTER_STATE="new"
        - ETCD_INITIAL_CLUSTER_TOKEN="mritd-etcd-cluster"
        - ETCD_ADVERTISE_CLIENT_URLS="http://etcd:2379,http://etcd:4001"
    - 修改k8s-node-1的etcd配置如下：
        - cat /etc/etcd/etcd.conf
        - ETCD_NAME=k8s-node-1
        - ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
        - ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
        - ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
        - ETCD_INITIAL_ADVERTISE_PEER_URLS="http://k8s-node-1:2380"
        - ETCD_INITIAL_CLUSTER="etcd=http://etcd:2380,k8s-node-1=http://k8s-node-1:2380,k8s-node-2=http://k8s-node-2:2380"
        - ETCD_INITIAL_CLUSTER_STATE="new"
        - ETCD_INITIAL_CLUSTER_TOKEN="mritd-etcd-cluster"
        - ETCD_ADVERTISE_CLIENT_URLS="http://k8s-node-1:2379,http://k8s-node-1:4001"
    - 修改k8s-node-2的etcd配置如下：
        - cat /etc/etcd/etcd.conf
        - ETCD_NAME=k8s-node-2
        - ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
        - ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
        - ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
        - ETCD_INITIAL_ADVERTISE_PEER_URLS="http://k8s-node-2:2380"
        - ETCD_INITIAL_CLUSTER="etcd=http://etcd:2380,k8s-node-1=http://k8s-node-1:2380,k8s-node-2=http://k8s-node-2:2380"
        - ETCD_INITIAL_CLUSTER_STATE="new"
        - ETCD_INITIAL_CLUSTER_TOKEN="mritd-etcd-cluster"
        - ETCD_ADVERTISE_CLIENT_URLS="http://k8s-node-2:2379,http://k8s-node-2:4001"
    - 启动etcd服务
        - systemctl start etcd    
    - 验证etcd环境是否成功
        - etcdctl set testdir/testkey0 0
        - etcdctl get testdir/testkey0 #查看是否返回0#
    - 查看etcd集群状态
        - etcdctl -C http://etcd:4001 cluster-health #查看是否返回cluster is healthy#
        - etcdctl -C http://etcd:2379 cluster-health #查看是否返回cluster is healthy#
    - 给三台机器安装docker
        - yum install docker -y
    - 配置master的Docker配置文件，使其允许从registry中拉取镜像
        - vim /etc/sysconfig/docker
        - OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false'
        - if [ -z "${DOCKER_CERT_PATH}" ]; then
        -     DOCKER_CERT_PATH=/etc/docker
        - fi
        - OPTIONS='--insecure-registry registry:5000' #添加这行#
    - 开启docker服务
        - chkconfig docker on
        - service docker start
    - 各节点安装ntp
        - yum install ntp -y
    - 各节点启动ntpd服务
        - systemctl start ntp
    - 各机器安装kubernetes
    - 查看kubernetes是否安装的源来自estuary
    - 查看kubernetes版本是否为 1.6.4
    - 配置k8s-master的kubernetes-apiserver
        - vim /etc/kubernetes/apiserver
        - KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
        - KUBE_API_PORT="--port=8080"
        - KUBE_ETCD_SERVERS="--etcd-servers=http://etcd:2379"
        - KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
        - KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
    - 配置k8s-master的kubernetes配置文件
        - vim /etc/kubernetes/config
        - KUBE_LOG_LEVEL="--v=0"
        - KUBE_LOGTOSTDERR="--logtostderr=true"
        - KUBE_ALLOW_PRIV="--allow-privileged=false"
        - KUBE_MASTER="--master=http://k8s-master:8080"
    - 启动k8s-master的kube-apiserver服务
        - systemctl enable kube-apiserver.service
        - systemctl start kube-apiserver.service
    - 启动k8s-master的kube-controller-manager服务
        - systemctl enable kube-controller-manager.service
        - systemctl start kube-controller-manager.service
    - 启动k8s-master的kube-apiserver服务
        - systemctl enable kube-controller-manager.service
        - systemctl start kube-controller-manager.service
    - 修改k8s-node-1/k8s-node-2的kubernetes配置文件
        - vim /etc/kubernetes/config
        - KUBE_LOGTOSTDERR="--logtostderr=true"
        - KUBE_LOG_LEVEL="--v=0"
        - KUBE_ALLOW_PRIV="--allow-privileged=false"
        - KUBE_MASTER="--master=http://k8s-master:8080"
    - 修改k8s-node-1的kubelet配置
        - vim /etc/kubernetes/kubelet
        - KUBELET_ADDRESS="--address=0.0.0.0"
        - KUBELET_HOSTNAME="--hostname-override=k8s-node-1"
        - KUBELET_API_SERVER="--api-servers=http://k8s-master:8080"
        - KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
        - KUBELET_ARGS=""
    - 修改k8s-node-2的kubelet配置
        - vim /etc/kubernetes/kubelet
        - KUBELET_ADDRESS="--address=0.0.0.0"
        - KUBELET_HOSTNAME="--hostname-override=k8s-node-2"
        - KUBELET_API_SERVER="--api-servers=http://k8s-master:8080"
        - KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
        - KUBELET_ARGS=""
    - 修改docker的cgroup驱动
        - vim /lib/systemd/system/docker.service
        - 将 --exec-opt native.cgroupdriver=systemd  修改为:
        - --exec-opt native.cgroupdriver=cgroupfs
    - 重启docker服务
         - systemctl daemon-reload
         - systemctl restart docker.service
    - 启动k8s-node-1/k8s-node-2的kubelet服务
         - systemctl enable kubelet.service
         - systemctl start kubelet.service
    - 启动k8s-node-1/k8s-node-2的kube-proxy服务
         - systemctl enable kube-proxy.service
         - systemctl start kube-proxy.service
    - 在master查看集群中节点及节点状态
         - kubectl -s http://k8s-master:8080 get node
         - kubectl get nodes
    - 所有机器安装flannel
         - yum install flannel -y
    - 配置所有节点的Fannel
         - vi /etc/sysconfig/flanneld
         - FLANNEL_ETCD_ENDPOINTS="http://etcd:2379"
         - FLANNEL_ETCD_PREFIX="/atomic.io/network"
    - 配置etcd中关于flannel的key
         - etcdctl mk /atomic.io/network/config '{ "Network": "10.0.0.0/16" }'
    - 启动各节点的flannel服务
         - systemctl enable flanneld.service
         - systemctl start flanneld.service 
    - 重新启动master上的各服务
         - service docker restart
         - systemctl restart kube-apiserver.service
         - systemctl restart kube-controller-manager.service
         - systemctl restart kube-scheduler.service
    - 重启启动k8s-node-1/k8s-node-2各服务
         - service docker restart
         - systemctl restart kubelet.service
         - systemctl restart kube-proxy.service
    - 启动一个简单的容器
         - kubectl run my-nginx --image=nginx --replicas=2 --port=80
    - 通过端口将应用连接到Internet上
         - kubectl expose rc my-nginx --port=80 --type=LoadBalancer
    - 查看rc
        - kubectl get rc
    - 查看service
        - kubectl get svc my-nginx
    - 创建启动容器的yaml配置文件
         - vim ./hello-world.yaml
         - apiVersion: v1
         - kind: Pod
         - metadata:
         -      name: hello-world
         - spec:  # 当前pod内容的声明
         -      restartPolicy: Never
         - containers:
         -      - name: hello
         -        image: "ubuntu:14.04"
         -        command: ["/bin/echo","hello”,”world"]
    - 通过这个YAML文件创建pod
         - kubectl create -f ./hello-world.yaml
    - 检验配置文件的正确性
         - kubectl create -f ./hello-world.yaml --validate
    - 查看pod状态
         - kubectl get pods
    - 查看pod输出
         - kubectl logs hello-world
    - 删除容器&pod
         - kubectl delete rc my-nginx
         - kubectl delete svc my-nginx
         - kubectl delete pod hello-world
    - 各节点卸载 kubernetes 
    - 各节点卸载 flannel & ntp & etcd & docker
  
  
- **Result:**
    - 测试上述步骤是否全部通过，若是，则pass；若不是，则fail