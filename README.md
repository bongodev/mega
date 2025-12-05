# Full Stack MERN APP

## Tech stack used in this project:
- GitHub (Code)
- Docker (Container)
- Jenkins (CI)
- ArgoCD (CD)
- OWASP (Dependency check)
- SonarQube (Quality)
- Trivy (Filesystem Scan)
- AWS EKS (Kubernetes)
- Helm (Monitoring using Grafana and Prometheus)


### Pre-requisites to implement this project:

> This project will be deployed on United States (Oregon) - us-west-2 but deploy your preffered region.

- <b>Create 1 Master machine on AWS with 2CPU, 8GB of RAM (t2.large) and 30 GB of storage manually or using Terraform.</b>
#
- <b>Open all the PORTs in security group of master machine</b> <br />
  | Port Range    | Source    | Description           |
  | ------------- | --------- | --------------------- |
  | 22            | 0.0.0.0/0 | SSH                   |
  | 443           | 0.0.0.0/0 | HTTPS                 |
  | 30000 - 32767 | 0.0.0.0/0 | NodePort services     |
  | 25            | 0.0.0.0/0 | SMTP                  |
  | 3000 - 10000  | 0.0.0.0/0 | Registered Ports      |
  | 6443          | 0.0.0.0/0 | Kubernetes API server |
  | 80            | 0.0.0.0/0 | HTTP                  |
  | 465           | 0.0.0.0/0 | SMTPS                 |


> We are creating this master machine because we will configure Jenkins master, eksctl, EKS cluster creation from here.

Install & Configure Docker by using below command, "NewGrp docker" will refresh the group config hence no need to restart the EC2 machine.

```bash
sudo apt update
```
```bash
sudo apt-get install docker.io -y

sudo usermod -aG docker ubuntu && newgrp docker

sudo chown $USER /var/run/docker.sock 

OR

sudo reboot

OR

sudo chmod 777 /var/run/docker.sock
```
# Install and configure Jenkins (Master machine)
## Sometimes Jenkins older versions are cached, clear them:
```bash
  sudo rm -f /etc/apt/sources.list.d/jenkins.list
  sudo rm -f /usr/share/keyrings/jenkins-keyring.asc
```

```bash
sudo apt update -y
sudo apt install fontconfig openjdk-17-jre -y

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
  
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
  
sudo apt update -y
```
## Confirm the latest Jenkins version is available
```bash
  apt-cache madison jenkins
```
## Install the latest Jenkins version
```bash
  sudo apt install jenkins -y
```
## Verify Jenkins version after installation
```bash
  sudo systemctl status jenkins
  jenkins --version
```
- <b>Now, access Jenkins Master on the browser on port 8080 and configure it:</b>
```bash
  sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
- <b>install jenkins suggested plugins </b>.
## Configure AWSCli (Master machine)
  - IAM user with **access keys and secret access keys**
  - AWSCLI should be configured
  - Download AWSCLI:
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  sudo apt install unzip
  unzip awscliv2.zip
  sudo ./aws/install
  ```
- Configure AWSCLI:
```bash
aws --version
aws configure
```

## Create an AWS Role Called mega-ec2-role and attach it to Master machine
  - Create Role:
    AWS IAM > roles > Create role > AWS Service > Use case (ec2) > Next > AdministratorAccess> Role name (mega-ec2-role) > Create Role
  - Add Role to EC2 Master machine:
    Master machine > Actions > Security > Modify IAM Role > Select mega-ec2-role > Update IAM Role
## Create EKS Cluster on AWS (Master machine)
### Install **kubectl** and **eksctl** (Master machine)
  - Install **kubectl** 
  ```bash
  curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
  chmod +x ./kubectl
  sudo mv ./kubectl /usr/local/bin
  kubectl version --short --client
  ```

  - Install **eksctl** 
  ```bash
  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
  sudo mv /tmp/eksctl /usr/local/bin
  eksctl version
  ```
  
  - <b>Create EKS Cluster (Master machine - it might take 15 to 20 minutes)</b>
  ```bash
  eksctl create cluster --name=mega \
                      --region=us-west-2 \
                      --version=1.30 \
                      --without-nodegroup
  ```
  - <b> Check clusters
    ```bash
      eksctl get clusters -o json
      Go to AWS CloudFormation, you should see ***eksctl-mega-cluster***
    ```
  - <b>Associate IAM Open ID Connect provider (OIDC Provider) on Master machine</b>
  ```bash
  eksctl utils associate-iam-oidc-provider \
    --region us-west-2 \
    --cluster mega \
    --approve
  ```
  - <b>Create Nodegroup on Master machine, it might take 15 to 20 minutes</b>
  - <i>It will create 2 nodes ec2 machines</i>
  ```bash
  eksctl create nodegroup --cluster=mega \
                       --region=us-west-2 \
                       --name=mega \
                       --node-type=t2.large \
                       --nodes=2 \
                       --nodes-min=2 \
                       --nodes-max=2 \
                       --node-volume-size=29 \
                       --ssh-access \
                       --ssh-public-key=eks-nodegroup-key 
  ```
- <i>OR, It will create 1 node ec2 machine</i>
  ```bash
  eksctl create nodegroup --cluster=mega \
                       --region=us-west-2 \
                       --name=mega \
                       --node-type=t2.large \
                       --nodes=1 \
                       --nodes-min=1 \
                       --nodes-max=1 \
                       --node-volume-size=29 \
                       --ssh-access \
                       --ssh-public-key=eks-nodegroup-key 
  ```

>  Make sure the ssh-public-key "eks-nodegroup-key" is available in your aws account

- <b>Check if Nodegroup is created</b>
```bash
  kubectl get nodes -n mega
  Also, go to AWS EC2, you should see your desired node machines got created
```
#
- Install and configure SonarQube (Master machine)
- <i>Pull the latest SonarQube Community Edition</i>
```bash
    docker pull sonarqube:community
```
- <i>Run the latest SonarQube docker container</i>
```bash
        docker run -d \
          --name sonarqube \
          -p 9000:9000 \
          -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
          sonarqube:community
```
```bash
    docker ps
    You should see SonarQube container is running
```
```bash
    Got to the link: <ec2-machine-ip>:9000/ and setup SonarQube account
    Initial user & passwoed: admin, admin
```
#
- Install Trivy (On Master Machine)
```bash
> Update dependencies
sudo apt update -y && sudo apt install -y wget curl apt-transport-https gnupg lsb-release

> Add the Trivy signing key (secure method)
curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  sudo gpg --dearmor -o /usr/share/keyrings/trivy-archive-keyring.gpg

> Add the Trivy repository (clean, non-duplicating)
echo "deb [signed-by=/usr/share/keyrings/trivy-archive-keyring.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list > /dev/null

> Install the latest Trivy
sudo apt update -y
sudo apt install -y trivy

> Verify version
trivy --version
```

#
## Add email for notification
<p>
  Follow this 
  <a href="https://docs.google.com/document/d/1dFRT_RP4yhHcCMiZug1mMVc3XWnDap8g4iMR8XLIAHw/view" target="_blank">document</a> for email app and set it up to Jenkins
</p>

## Steps to implement the project:
- <b>Go to Jenkins Master and click on <mark> Manage Jenkins --> Plugins --> Available plugins</mark> install the below plugins:</b>
  - OWASP Dependency-Check
  - SonarQube Scanner
  - Docker
  - Pipeline: Stage View
  - Blue Ocean

## Add OWASP Dependency Check (it  might take 20 minutes to get installed):
```bash
Jenlkins > Manage > Tools > Dependency-Check installations
Hit: Add Dependency-Check
Input: Name = OWASP > Check Mark: Install automatically > + Add installer > Select: Install from github.com > Save
```
## Store SonarQube token in Jenkins
```bash
Go to <machine_ip>:9000
Create SonarQube Token:
  - Administration > Security > users > Tokens
  - Generate the token and copy it, store it to Jenkins > Credentials > Global Credentials > Kind: Secret text
  - Connect the created Sonar TOKEN with Jenkins > Tools > Search: SonarQube Scanner installations
  - Hit: Add SonarQube Scanner > Name: Sonar > CheckMark: Install Automatically > Install from Maven Central > Save
```
## Integrate SonarQube with Jenkins
```bash
  Jenkins > manage > system > SonarQube servers > SonarQube installations:
  Name: Sonar > Server URL: <Machine_IP>:9000 > Server authentication token: Select sonar-key from dropdown
```

## Create SonarQube Webhook for Jenkins
```bash
  SonarQube > Administration > Configuration > Webhooks
  URL to use (Jenkins url): <ec2-machine-ip>:8080/sonarqube-webhook
```

## Add GitHub PAT Key to Jenkins
```bash
  - Create a GitHub PAT Key
  - Add the key to Jenkins > Credentials > Global Credentials > Kind: Username with password
```


## Install ArgoCD on Master Machine
- <b>Install and Configure ArgoCD (Master Machine)</b>
  - <b>Create argocd namespace</b>
  ```bash
  kubectl create namespace argocd
  ```
  - <b>Apply argocd manifest</b>
  ```bash
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```
  - <b>Make sure all pods are running in argocd namespace</b>
  ```bash
  watch kubectl get pods -n argocd
  ```
  - <b>Install argocd CLI</b>
  ```bash
  sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
  ```
  - <b>Provide executable permission</b>
  ```bash
  sudo chmod +x /usr/local/bin/argocd
  ```
  - <b>Check argocd services</b>
  ```bash
  kubectl get svc -n argocd
  ```
  - <b>Change argocd server's service from ClusterIP to NodePort</b>
  ```bash
  kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
  ```
  - <b>Confirm service is patched or not</b>
  ```bash
  kubectl get svc -n argocd
  ```
  ## Check ArgoCd in Node Machine (Not in Master)
  - <b> Check the port where ArgoCD server is running (Type should be NodePort now) and expose (i.e. port 80:30169 argocd-server) it on security groups of a worker node (mega-mega-Node not mega node)</b>

  - <b>Access it on browser, click on advance and proceed with</b>
  ```bash
  <public-ip-worker>:<port> i.e.: <worker_ip>:argocd-server-port 30169
  proceed to unsafe
  ```

  - <b>Fetch the initial password of argocd server</b>
  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
  ```
  - <b>Username: admin</b>
  - <b> Now, go to <mark>User Info</mark> and update your argocd password
  - <b> set new password </b>
#

## Connect GitHub Repo in Argocd
```bash
  Settings > Repositories > Connect Repo > VIA HTTP/HTTPS (for public repo)
  For Private repo: user - github-user-name, password - create a github PAT key (i.e. argocd-key)
```
## Add Cluster in Argocd
```bash
  Login to ArgoCD in terminal (master machine terminal):
    $ argocd login <argocd-url> --username admin
      example: argocd login 44.255.55.146:32672 --username admin

  Check ArgoDC Cluster List:
    $ argocd cluster list (you should see default cluster)

  Get context:
    kubectl config get-contexts

  Add cluster information to inform ArgoCD where to deploy:
    $ argocd cluster add cluster-name-from-context --name any-name
      Example: argocd cluster add mega-project-user@mega.us-west-2.eksctl.io --name mega-ekscluster
            mega-project-user@mega.us-west-2.eksctl.io is the concept of RBAC

```
## Create an app in Argocd (browser)
```bash
  Applications > New app:
  - Set these: Application Name: mega | Project Name: default | Sync Policy: Automatic
  - Check mark: Enable Auto-Sync, Prune Resources, Self Heal, Auto-Create Namespace
  - SOURCE: Resource URL: mega project GitHub URL, Revision: main (gitHub branch)
  - Path: kubernetes (In GitHub, kubenetes manifest directory is "kubernetes")
  - DESTINATION: select the cluster (not default one)
  - Namespace: mega
```

## Add FrontEnd and BackEnd services NodePort to Node Instance (worker) Security group:
```bash
  backend nodePort: 31100
  frontend nodePort: 31000
```

## Add Shared Library Repo for Jenkins to pick up the Shared Library:
```bash
Repo URL: https://github.com/bongodev/jenkins-shared-library
Jenkins > manage > System > Global Trusted Pipeline Libraries: Name: Shared | Project Repository: Repo URL | Credentials: Select GitHub-PAT-Key
```

## Add DockerHub PAT key to Jenkins:
```bash
  Jenkins > Credentials > Global Credentials
```

## SetUP CI Pipeline in Jenkins:
```bash
  Jenkins > + New Item > Create CI Pipeline with Jenkinsfile code (from repo)
  Build the job with image tag (first build might take 20 minutes)
```

## SetUP CD Pipeline in Jenkins:
```bash
  Jenkins > + New Item > Create CD Pipeline with GitOps>Jenkinsfile code (from repo)
  Build the job
```

## Do this before Building/hitting CI and CD Jenkins jobs:
```bash
  Go to Automations directory:
  File: updatefrontendnew.sh > change INSTANCE_ID=<ec2-node-machine-ID where your app will be running>
  File: updatebackendnew.sh > change INSTANCE_ID=<ec2-node-machine-ID where your app will be running>
```

#
## Setup Application Monitor on EKS cluster using prometheus and grafana via HELM (On Master machine)
- Install Helm Chart
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
```
```bash
chmod 700 get_helm.sh
```
```bash
./get_helm.sh
```

#
-  Add Helm Stable Charts for Your Local Client
```bash
helm repo add stable https://charts.helm.sh/stable
```

#
- Add Prometheus Helm Repository
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

#
- Create Prometheus Namespace
```bash
kubectl create namespace prometheus
kubectl get ns
```

#
- Install Prometheus using Helm
```bash
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

#
- Verify prometheus installation
```bash
kubectl get pods -n prometheus
```

#
- Check the services file (svc) of the Prometheus
```bash
kubectl get svc -n prometheus
```

#
- Expose Prometheus and Grafana to the external world through Cluster IP to NodePort
> [!Important]
> Edit Type: ClusterIp to NodePort, make sure you save the file.

```bash
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
```

#
- Verify service
```bash
$ kubectl get svc -n prometheus
- It should be stable-kube-prometheus-sta-operator
```
- Add Prometheus stable-kube-prometheus-sta-operator IP to your EC2 Node Machine's SecurityGroup
#
- Now,let’s change the SVC file of the Grafana and expose it to the outer world (Edit Type: ClusterIp to NodePort)
```bash
kubectl edit svc stable-grafana -n prometheus

```

#
- Check grafana service
```bash
kubectl get svc -n prometheus
It should be stable-graphana 
```
- Add Graphana stable-graphana IP to your EC2 Node Machine's SecurityGroup
#
- Get a password for grafana
```bash
kubectl get secret --namespace prometheus stable-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
> [!Note]
> Username: admin

#
- Now, view the Dashboard in Grafana

#
## ⚠️ Clean Up EKS Cluster to save money
- <b id="Clean">Delete eks cluster</b>
```bash
eksctl delete cluster --name=mega --region=us-west-2
Delete the master machine
```
