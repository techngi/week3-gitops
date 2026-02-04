### Hybrid Setup using local VM and AWS EKS

- Create local VMs

VM1: jenkins-ci
Jenkins (Docker preferred)
Docker engine
Trivy scanner
Git + tools

Hardware Specs - 4 vCPU, 8GB RAM, 60GB disk

```bash
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.define "jenkins-ci" do |jc|
    jc.vm.box = "ubuntu/jammy64"
    jc.vm.hostname = "jenkins-ci"

    # --- Networking ---
    # Adapter 1: NAT (default - internet access)
    # Adapter 2: Host-only (stable access from host)
    jc.vm.network "private_network", ip: "192.168.56.10"

    # --- Provider settings ---
    jc.vm.provider "virtualbox" do |vb|
      vb.name = "jenkins-ci"
      vb.memory = 8192       # 8 GB RAM
      vb.cpus  = 4           # 4 CPU cores

      # --- Disk: 60 GB (VDI, dynamically allocated) ---
      vb.customize [
        "createhd",
        "--filename", "#{Dir.pwd}/jenkins-ci-disk.vdi",
        "--size", 61440        # 60 GB in MB
      ]

      vb.customize [
        "storageattach", :id,
        "--storagectl", "SATA Controller",
        "--port", 1,
        "--device", 0,
        "--type", "hdd",
        "--medium", "#{Dir.pwd}/jenkins-ci-disk.vdi"
      ]
    end
  end

end
```


- Create gitops repo on github

```bash
mkdir -p ~/week3-gitops
cd ~/week3-gitops/
git init
sudo nano README.md
git add README.md
git config --global user.email "sanaqvi573@gmail.com"
git config --global user.name "Syed Sohail Abbas"
git commit -m "Initial Commit"
git remote add origin https://github.com/techngi/week3-gitops.git
git remote set-url origin https://github.com/techngi/week3-gitops.git
git push -u origin master

mkdir -p helm/week3-app envs/dev envs/prod
```

- Install utilities

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install ca-certificates curl gnupg lsb-release git unzip jq
```

- Install Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker version
```

- Run Jenkins in Docker + persistent storage

```bash
sudo mkdir -p /srv/jenkins_home # Create folders
sudo chown -R 1000:1000 /srv/jenkins_home
```

- Run Jenkins:

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v /srv/jenkins_home:/var/jenkins_home \
  --restart unless-stopped \
  jenkins/jenkins:lts
```

- Get initial admin password:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

- Open Jenkins from host:
http://<vm-hostonly-ip>:8080


- Install plugins (minimum):
Pipeline
Git
Credentials Binding
Docker Pipeline
Blue Ocean (optional)
GitHub integration (optional)

- Install CI tools on VM1 (AWS + K8s CLIs + scanners)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

- Install KubeCtl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

- Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

- Install Trivy (image scan)

```bash
sudo apt -y install wget
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /etc/apt/keyrings/trivy.gpg
echo "deb [signed-by=/etc/apt/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt -y install trivy
trivy -v
```

# Create EKS in the cheapest way (NO NAT Gateway)

- Download and install eksctl

```bash
curl -s --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

# Create and configure AWS credentials
- Create an IAM user for your lab

```bash
aws configure
# set region: ap-southeast-2 (Sydney)
aws sts get-caller-identity
```

# Create AWS Cluster

- Create YAML file to setup cluster in AWS

```bash
cat > cluster-public.yaml <<'YAML'
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: devops-hybrid
  region: ap-southeast-2
  version: "1.32"

vpc:
  nat:
    gateway: Disable

managedNodeGroups:
  - name: ng-dev
    instanceType: t3.small
    desiredCapacity: 2
    minSize: 1
    maxSize: 2
    privateNetworking: false
    volumeSize: 30
    ssh:
      allow: false

iam:
  withOIDC: true
YAML
```

```bash

eksctl create cluster -f cluster-public.yaml

aws eks describe-cluster --name devops-hybrid --region ap-southeast-2

aws eks update-kubeconfig --name devops-hybrid --region ap-southeast-2

kubectl get pods -A
```

# Create ArgoCD docker in AWS Cluster and access it

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd get pods

kubectl -n argocd port-forward svc/argocd-server 8085:80

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

On Host from vagrant folder
ssh -L 8085:localhost:8085 vagrant@192.168.56.10

http://localhost:8085
```


# Create Helm Chart

```bash
mkdir -p helm
helm create helm/week3-app
ls -la helm/week3-app
```

It should include Chart.yaml, values.yaml, templates/, etc.

week3-gitops/
├── helm/
│   └── week3-app/
│       ├── Chart.yaml
│       ├── templates/
│       └── values.yaml        ← default values
│
└── envs/
    ├── dev/
    │   └── values.yaml        ← dev overrides
    └── prod/
        └── values.yaml        ← prod overrides

# Access Argo CD

```bash
sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
argocd version --client

argocd login localhost:8085 --insecure

kubectl create namespace devops-week5
```bash

# Create ArgoCD App

```bash
argocd app create week3-app-dev --repo https://github.com/techngi/week3-gitops.git --path helm/week3-app --dest-namespace devops-week5 \
--dest-server https://kubernetes.default.svc --values ../../envs/dev/values.yaml --sync-policy automated
```

# Create ECR Repository and upload image

```bash
aws ecr create-repository --repository-name week3-app --region ap-southeast-2
aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin 421869852482.dkr.ecr.ap-southeast-2.amazonaws.com


docker pull sanaqvi573/week3-app:latest
docker tag sanaqvi573/week3-app:latest 421869852482.dkr.ecr.ap-southeast-2.amazonaws.com/week3-app:latest
docker push 421869852482.dkr.ecr.ap-southeast-2.amazonaws.com/week3-app:latest
aws ecr list-images --repository-name week3-app --region ap-southeast-2
```
