# [Istio Study Week 2] 

## 1.1 Basic Concept of Istio 

## 1.2 Istalling Minikube

### 1.2.1 Installing Minikube with Bash file [2024.09.18 기준]
<pre>
<code>
#!/bin/bash

if [ -f /etc/debian_version ]; then
    echo "ubuntu found"
    
    if [ "$(id -u)" -ne 0 ]; then
        echo "Please run this bash in root."
        exit 1
    fi

    sudo apt-get update
    sudo apt-get upgrade -y

    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

    # Add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update

    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

    elif [ -f /etc/redhat-release ]; then
        echo "rehl found"

    if [ "$(id -u)" -ne 0 ]; then
        echo "Please run this bash in root."
        exit 1
    fi

    sudo yum update -y

    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

    sudo yum install -y yum-utils
    sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

    sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin


else
    echo "Only support Debian/Rehl"
fi

sudo systemctl start docker
</code>
</pre>

### 1.2.2 Start Minikue cluster

<pre>
<code>
minikube config set driver docker
minikube start

#root 계정이 아닌 계정으로 사용
#만약 docker 권한이 없으면 설정 필요
#sudo usermod -aG docker $USER && newgrp docker
</code>
</pre>

## 1.3 Command Set of Hands on Demo

### 1.3.1 Istio Init 
<pre>
<code>
#Installing Istio and Istio plugins
kubectl apply -f ./1-istio-init.yaml
kubectl apply -f ./2-istio-minikube.yaml

#Lastest version of Istio does not make pods anymore.
#It registered as CustomResourceDefinitions.
kubectl get po -n istio-system

#Set kiali User
kubectl apply -f ./3-kiali-secret.yaml
</code>
</pre>


#### [Optional] How to Decode contents in 3-kiali-secret.yaml
```
echo {encoded_string} | base64 -d
```



### 1.3.2 Enabling sidecar injection
SideCar를 injection하는 법
1. yaml로 istio binary 및 app을 실행
   * 추천하지 않음
   * kubernetes definition에 영향이 있을 수 있음
2. Flag를 설정
   * SideCar Injector를 사용해 Flag가 설정된 곳 SideCar 주입
   * Pod를 Scheduling 시 Istio가 탐지 후 Inject
   * Label을 설정해야 동작
   * 기본적으로 NS 단위로 설정


<pre>
<code>
#Make sure you type correctly
kubectl label namespace default istio-injection=enabled

#To check label is set
kubectl describe ns default
</code>
</pre>

### 1.3.3 Deploying an Application
```
kubectl apply -f ./4-application-full-stack.yaml

#Get minikube IP to access Kiali, Jaeger
minikube ip

#Kiali
#{ip}:30080

#Jaeger
#{ip}:31001
```

### 1.3.4 Setting Timeout
#### 일정 시간 응답이 없을 시 timeout 시킨다.
```
route 블럭 위에 timeout 설정
```