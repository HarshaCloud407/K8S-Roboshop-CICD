# ---------- 1. Prerequisites ----------
1. Ubuntu Servers --- 2 ( 2cpu min & Volumes are min 25GB)  

2. IAM Role Create With AdminFullAcess.

3. SecurityGroup with ALLTCP

# Update the system and install the basic tools needed below
```
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl unzip wget tar gnupg apt-transport-https ca-certificates
```

# ---------- 2. AWS CLI v2 ----------
# Download, unzip, and install the AWS command-line tool
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```
# ---------- 3. AWS credentials ----------
# Asks for your Access Key, Secret Key, region, and output format
aws configure set region "$REGION"
```
aws configure
```
# Confirm the credentials work (shows your account and user)
```
aws sts get-caller-identity
```

# ---------- 4. kubectl ----------
# Download the latest stable kubectl and put it in /usr/local/bin
```
ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')
echo $ARCH   # should print amd64 (or arm64)

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/${ARCH}/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

# ---------- 5. eksctl ----------
# Download eksctl (the tool that creates EKS clusters) and install it
```
ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')
curl -fsSL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_${ARCH}.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```
# ---------- 6. Create the cluster ----------
# Builds a managed EKS cluster with 2-4 worker nodes (takes ~15-20 minutes)
```
eksctl create cluster \
  --name my-eks-cluster \
  --region ap-southeast-2 \
  --version 1.33 \
  --nodegroup-name my-nodes \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed
```
# ---------- 7. Verify ----------
```
kubectl get nodes -o wide     # worker nodes and their status
kubectl get pods -A           # all pods in all namespaces
kubectl cluster-info          # cluster endpoint info
```

# ---------- 8. Delete-EKS Cluster --------
```
eksctl delete cluster --name my-eks-cluster --region ap-southeast-2
```

----------------------------------------------------------------------------------------------------------

# ---------- 1. Prerequisites ----------
# Update package list and install tools needed to add Docker's repository
```
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

# ---------- 2. Add Docker's official repository ----------
# Create the keyring folder and download Docker's signing key
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
# Add Docker's repository for your Ubuntu version
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
# ---------- 3. Install Docker ----------
```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
# Let your user run docker without sudo (takes effect after logout/login)
```
sudo usermod -aG docker ubuntu

docker --version
```
# ---------- 4. Run SonarQube ----------
# Code-quality tool, available at http://<server-ip>:9000 (default login: admin / admin)
```
sudo docker run -d --name sonar \
  --restart unless-stopped \
  -p 9000:9000 \
  sonarqube:lts-community
```
# ---------- 5. Run Nexus ----------
# Artifact repository, available at http://<server-ip>:8081
# Data is kept in the "nexus_data" volume so it survives container removal
```
sudo docker run -d --name nexus \
  --restart unless-stopped \
  -p 8081:8081 \
  -e INSTALL4J_ADD_VM_PARAMS="-Xms512m -Xmx1g -XX:MaxDirectMemorySize=1g" \
  -v nexus_data:/nexus-data \
  sonatype/nexus3:latest
```

# ---------- 6. Get the Nexus admin password ----------
```
sudo docker exec nexus cat /nexus-data/admin.password
```
------------------------------------------------------------------------------------------
# ------------ Jenkins-install- Process --------------------------------------------------

Jenkins Installation on Ubuntu 24.04 (End to End)
1. Update the system
bash
```
sudo apt update && sudo apt upgrade -y
```
3. Install Java 21
bash
```
sudo apt install fontconfig openjdk-21-jre -y
java -version
```
5. Add the Jenkins repository (2026 signing key)
bash
```
sudo mkdir -p /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```
4. Install Jenkins
bash
```
sudo apt update
sudo apt install jenkins -y
```
6. Start and enable the service
bash
```
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
8. Get the initial admin password
bash
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
1. Install the AWS CLI v2

bash
```
sudo apt-get update && sudo apt-get install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```
2. Install kubectl

bash
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```
3. Check that the jenkins user can find them

bash
```
sudo -u jenkins aws --version
sudo -u jenkins kubectl version --client

```
===========trivy-installation-------------------------

Option 1: apt repository (recommended)

bash
```
sudo apt-get install -y wget gnupg lsb-release

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt-get update
sudo apt-get install -y trivy
```

Option 2: install script (if the apt repo gives you trouble)

bash
```
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin
```

Verify the install

bash
```
trivy --version
```
