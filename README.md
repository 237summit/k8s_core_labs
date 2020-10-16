# Kubernetes 실습환경구성

**Single Master, Multi node Kubernets** 환경을 구성합니다.

|HOST|IP address  | arch | CPU | Memory | OS |
|--|--|--|--|--|--|
|master.example.com|10.100.0.104|X86_64|2core|4GiB |CentOS 7.6|
|node1.example.com|10.100.0.101|X86_64|2core|2GiB |CentOS 7.6|
|node2.example.com|10.100.0.102|X86_64|2core|2GiB |CentOS 7.6|
|node3.example.com|10.100.0.103|X86_64|2core|2GiB |CentOS 7.6|


## 1. Docker Install
master, node1,node2, node3 시스템에 도커 설치

[docs.docker.com](https://docs.docker.com/engine/install/centos/)

    # yum install -y yum-utils
    # yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    # yum install docker-ce docker-ce-cli containerd.io
    # systemctl start docker && systemctl enable docker
    # docker version
	    Client: Docker Engine - Community
		 Version:           19.03.8
		 API version:       1.40
		 Go version:        go1.12.17
		 Git commit:        afacb8b
		 Built:             Wed Mar 11 01:27:04 2020
		 OS/Arch:           linux/amd64
		 Experimental:      false

		Server: Docker Engine - Community
		 Engine:
		  Version:          19.03.8
		  API version:      1.40 (minimum version 1.12)
		  Go version:       go1.12.17
		  Git commit:       afacb8b
		  Built:            Wed Mar 11 01:25:42 2020
		  OS/Arch:          linux/amd64
		  Experimental:     false
		 containerd:
		  Version:          1.2.13
		  GitCommit:        7ad184331fa3e55e52b890ea95e65ba581ae3429
		 runc:
		  Version:          1.0.0-rc10
		  GitCommit:        dc9208a3303feef5b3839f4323d9beb36df0a9dd
		 docker-init:
		  Version:          0.18.0
		  GitCommit:        fec3683

    
## 2. kubeadm, kubectl, kubelet 설치 동작
master, node1,node2, node3 시스템에 kubeadm, kubectl, kubelet 설치 및 동작

[kubernetes.io](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

	1) Swap disabled
	# swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
	
	2) Letting iptables see bridged traffic
	# cat <<EOF > /etc/sysctl.d/k8s.conf
	net.bridge.bridge-nf-call-ip6tables = 1
	net.bridge.bridge-nf-call-iptables = 1
	EOF
	# sysctl --system

	3) Disable firewall
	# systemctl stop firewalld 
	# systemctl disable firewalld
	
	4) Set SELinux in permissive mode (effectively disabling it)
	# setenforce 0
	# sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
	
	5) kubeadm, kubelet, kubectl 설치
	# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
	[kubernetes]
	name=Kubernetes
	baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
	enabled=1
	gpgcheck=1
	repo_gpgcheck=1
	gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
	exclude=kubelet kubeadm kubectl
	EOF
	
	# yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
	# systemctl start kubelet && systemctl enable kubelet


## 3. Master 컴포넌트 구성및 네트워크 환경구성
master 시스템에서만 구성 
### Creating a single control-plane cluster with kubeadm

	# kubeadm init
	...
	To start using your cluster, you need to run the following as a regular user:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config
	...
	kubeadm join 10.100.0.104:6443 --token 1ou05o.kkist9u6fbc2uhp3 --discovery-token-ca-cert-hash sha256:8d9a7308ea6ff73.........576c112f326690


**kubeadm init** 명령실행시 master에 kubernetes 컴포넌트들이 생성된다. 
출력 결과에서 위의 내용중 mkdir, cp, chown 명령을 kubernetes 관리자 계정에서 실행한다.

	$ mkdir -p $HOME/.kube
	$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

**kubeadm init** 명령 실행 결과 나오는 kubeadm join <토크> 은  worker node가 master에 연결될때 필요하다. 별도로 저장해두는것이 좋다.

	# cat > token.tx
	kubeadm join 10.100.0.104:6443 --token 1ou05o.kk...3 --discovery-token-ca-cert-hash sha256:8d9a7308ea6ff73.........576c112f326690
	<Ctrl>+<d>

### Installing a Pod network add-on
Weave Net works

	# kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

	# kubectl get nodes

## 4. Worker Node 구성
node1, node2, node3에서 실행
master의 **kubeadm init** 명령 실행시 출력된 토큰을 가지고 마스터와 연결

	# kubeadm join 10.100.0.104:6443 --token 1ou05o.kk...3 --discovery-token-ca-cert-hash sha256:8d9a7308ea6ff73.........576c112f326690

마스터 노드에서 kubectl 명령으로 쿠버네티스 상태 확인

	# kubectl get nodes
	NAME                 STATUS   ROLES    AGE   VERSION
	master.example.com   Ready    master   10m   v1.18.0
	node1.example.com    Ready    <none>   17m   v1.18.0
	node2.example.com    Ready    <none>   17m   v1.18.0
	node3.example.com    Ready    <none>   17m   v1.18.0
	
	
## 5. bash shell에서 [TAB]키를 이용해 kubernetes command 자동완성 구성하기
	
	# source <(kubectl completion bash)
	# echo "source <(kubectl completion bash)" >> ~/.bashrc


## 6. 실습 : 간단한 yaml 파일을 생성해서 nginx를 배포해보자.

	1. 아래와 같은 example.yaml을 생성한다.
	$ cat > example.yaml
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: deploy-exam
	spec:
	  replicas: 1
	  selector:
	    matchLabels:
	      app: nginx
	  template:
	    metadata:
	      labels:
	        app: nginx
	    spec:
	      containers:
	      - name: nginx
	        image: nginx

	2. 앞에서 생성한 yaml을 이용해 웹서버 nginx를 실행한다.
	# kubectl create -f example.yaml

	3. 동작중인 deploy를 확인
	# kubectl get deploy

	4. depoy의 child로 동작되는 replicaset과 pod를 확인한다.
	# kubectl get rs
	# kubectl get pod

	Pod 확장하기
	5. 다음 명령을 사용하여 Pod 수를 5개로 확장하자:
	$ kubectl scale deploy deploy-exam --replicas=5
	$ kubectl get pod
	$ kubectl get pod -o wide

	4. 생성된 deploy를 제거한다.
	$ kubectl delete deploy deploy-exam
