### Hybrid Setup using local VM and AWS EKS

- Create local VM

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
```
