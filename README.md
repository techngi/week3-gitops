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
