## Lab 1: Use Terraform to Setup the `Docker Server` and `Jenkins Server` for CICD Lab.

**Objective:**
The objective of this lab is to set up two AWS EC2 instances, `one for Jenkins` and `one for Docker,` using Terraform. This lab aims to provide a foundation for building a Continuous Integration/Continuous Deployment (CICD) environment.

### Task 1: Manually launch the Jump Server EC2 Instance

* Region: North Virginia (us-east-1).
* Use tag Name: `Mehar-Jump-Server`
* AMI Type and OS Version: `Ubuntu 22.04 LTS`
* Instance type: `t2.micro`
* Create key pair with name: `Mehar-DevOps-Keypair`
* Create security group with name: `Mehar-DevOps-SG`
   (Include Ports: `22 [SSH],` `80 [HTTP],` `8080 [Jenkins],` `9999 [Tomcat],` and `4243 [Docker]`)
* Configure Storage: 10 GiB
* Click on `Launch Instance.`


### Task 2: Installing Terraform onto `Jump Server` to automate the creation of 2 more EC2 instances.

Once the Jump Server is up and running, SSH into the machine using `MobaXterm` or `Putty` with the username `ubuntu` and do the following:

```
sudo hostnamectl set-hostname AnchorServer
bash
```
```
sudo apt update
```
```
sudo apt install wget unzip -y
```
```
wget https://releases.hashicorp.com/terraform/1.9.2/terraform_1.9.2_linux_amd64.zip
```
[Click here](https://developer.hashicorp.com/terraform/downloads) for Terraform's Latest Versions
```
unzip terraform_1.9.2_linux_amd64.zip
ls
```
```
sudo mv terraform /usr/local/bin
```
```
rm terraform_1.9.2_linux_amd64.zip
```
```
ls
terraform -v
```
```
terraform
```
---------------------------------------------------------------------
### Task-2: Install Python 3, pip, AWS CLI, and Ansible on to Anchor Server
Install Python 3 and the required packages:
```
sudo apt-get update
```
```
sudo apt-get install python3-pip -y
```
Installing `AWS CLI`
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
Installing Ansible
```
sudo apt-get install ansible -y
```
For Authentication with AWS we need to provide `IAM User's CLI Credentials`
```
aws configure
```
#### Credentials Example:

| **Access Key ID** | **Secret Access Key** |
| ----------------- | --------------------- |
| AKIAXMWJXSSHRD27T6SC | H4Vh0U5oenKfmJ/+FEUcbaGbDjcnGAmZvQLX7zTT |

<details>
  <summary>To know how to create New Credentials, Click here:</summary>

##### Here is a step-by-step summary of the instructions:

1. Go to the AWS console. On the top right corner, click on your name or AWS profile ID.
2. Click on Security Credentials.
3. Under AWS IAM Credentials, click on **Create Access Key**.
4. If you already have two active keys, deactivate and delete the older one. Create a new one, download, and save it.
5. Complete the `aws configure` step using the newly created access key, secret key, region, and output format.
</details>

---------------------------------------------------------------------
#### Once configured, do a smoke test to check if your credentials are valid and got the access to AWS account.

You can check using any one command or both.
```
aws s3 ls
```
(Or)
```
aws iam list-users
```

#### Create a host inventory file with the necessary permissions (Ansible need this Inventory file for identifying the targets)
 ```
 sudo mkdir /etc/ansible && sudo touch /etc/ansible/hosts
```
```
 sudo chmod 766 /etc/ansible/hosts
```
**Note:** The above command gives `Read, Write and Execute` permissions to `Owner,` `Read and Write` permissions to the `Group,` and `Read and Write` permissions to `Others.`

---------------------------------------------------------------------
### Task-3: Utilizing Terraform, initiate the deployment of two new servers: `Docker-server` and `Jenkins-server` 

* For **Git Operations Lab** we will use the same **Anchor EC2** from where we are operating now 

**Step-01:** As a first step, create a keyPair using `ssh-keygen` Command.

(Same public will be attached to newly created EC2 Instances)

**Note:**
1. This will create `id_rsa` and `id_rsa.pub` in Anchor Machine in `/home/ubuntu/.ssh/` path.
2. While creating choose defaults like:
   * path as **/home/ubuntu/.ssh/id_rsa**,
   * don't set up any passphrase, and just hit the '**Enter**' key for 3 questions it asks.

```
ssh-keygen -t rsa -b 2048 
```

<details>
  <summary>Click here for explanation</summary>
  
* `-t rsa:` Specifies the type of key to create, in this case, RSA.
* `-b 2048:` Specifies the number of bits in the key, 2048 bits in this case. The larger the number of bits, the stronger the key.
</details>

 **Step-02:** Create the terraform directory and set up the config files in it

Create the Terraform configuration and variables files as described.

```
mkdir devops-labs && cd devops-labs
```

**Step-03:** Now create the Terraform config files.
```
vi DevOpsServers.tf
```
Copy and paste the below code into `DevOpsServers.tf`
```
provider "aws" {
  region = var.region
}

resource "aws_key_pair" "mykeypair" {
  key_name   = var.key_name
  public_key = file(var.public_key)
}

# to create 2 EC2 instances
resource "aws_instance" "my-machine" {
  # Launch 2 servers
  for_each = toset(var.my-servers)

  ami                    = var.ami_id
  key_name               = var.key_name
  vpc_security_group_ids = [var.sg_id]
  instance_type          = var.ins_type

  # Read from the list my-servers to name each server
  tags = {
    Name = each.key
  }

  provisioner "local-exec" {
    command = <<-EOT
      echo [${each.key}] >> /etc/ansible/hosts
      echo ${self.public_ip} >> /etc/ansible/hosts
    EOT
  }
}
```
Now, create the variables file with all variables to be used in the `DevOpsServers.tf` config file.
```
vi variables.tf
```
**Note:** Change the following Inputs in `variables.tf.`

1. Edit the **Allocated Region** (**Ex:** ap-south-1) & **AMI ID** of same region,
2. Replace the same **Security Group ID** Created for the Anchor Server
3. Add your Name for **KeyPair** ("**YourName**-CICDlab-KeyPair")

```
variable "region" {
    default = "us-east-1"
}

# Change the SG ID. You can use the same SG ID used for your CICD anchor server
# Basically the SG should open ports 22, 80, 8080, 9999, and 4243
variable  "sg_id" {
    default = "sg-06dc8863d3ed3d280" # us-east-1
}

# Choose a free tier Ubuntu AMI. You can use below. 
variable "ami_id" {
    default = "ami-04505e74c0741db8d" # us-east-1; Ubuntu
}

# We are only using t2.micro for this lab
variable "ins_type" {
    default = "t2.micro"
}

# Replace 'yourname' with your first name
variable key_name {
    default = "YourName-Jenkins-Docker-KeyPair"
}

variable public_key {
    default = "/home/ubuntu/.ssh/id_rsa.pub"   #Ubuntu OS
}

variable "my-servers" {
  type    = list(string)
  default = ["jenkins-server", "docker-server"]
}
```
**Step-04:** Now, execute the terraform commands to launch the new servers
```
terraform init
```
```
terraform fmt
```
```
terraform validate
```
```
terraform plan
```
```
terraform apply -auto-approve
```
Once the Changes are Applies, Go to `EC2 Dashboard` and check that `2 New Instances` are launched.

**Step-05:** Once the Terraform commands are executed, check the `inventory file` and ensure the below output.
```
cat /etc/ansible/hosts
```
The above command displays the IP addresses of the `Jenkins server` and `docker server` as an example shown below.

* [jenkins-server]

  44.202.164.153
  
* [docker-server]

  34.203.249.54

**(Optional Step):** When you `Stop` and `Start` the EC2 Instances, the Public IP Changes. In that case, execute the below command to update Jenkin's & Docker's new Public IPs in `Inventory file.` 
```
sudo vi /etc/ansible/hosts 
```
Once Updated, Save the File.

**Step-06:**  Check the access from `Anchor to Jenkins` and `Anchor to Docker`

##### From `Anchor Server` SSH into `Jenkins-Server` and check they are accessible.

```
ssh ubuntu@<Jenkins ip address>
```
#### Set the hostname as
```
sudo hostnamectl set-hostname Jenkins
bash
```
```
sudo apt update
```
**Exit** only from the Jenkins Server, not the Anchor Server.

##### Now from `Anchor Server` SSH into `Docker-Server` and check they are accessible.

```
ssh ubuntu@<Docker ip address>  
```
#### Set the hostname as
```
sudo hostnamectl set-hostname Docker
bash
```
```
sudo apt update
```
**Exit** only from the Docker Server, not the Anchor Server.

---------------------------------------------------------------------
### Task-4: Use `Ansible` to deploy respective packages onto each of the servers 

#### Step-01: In Anchor Server Create a directory and change to it
```
cd ~
mkdir ansible && cd ansible
```
#### Step-02: Now, Create a playbook, which will deploy packages onto the `Docker-server` and `Jenkins-Server.`

* Create a new File with the name `DevOpsSetup.yml.`
```
vi DevOpsSetup.yml
```
* Copy and paste the below code and save it.
```
---
- name: Start installing Jenkins pre-requisites before installing Jenkins
  hosts: jenkins-server
  become: yes
  become_method: sudo
  gather_facts: no

  tasks:

  - name: Update apt repository with latest packages
    apt:
      update_cache: yes
      upgrade: yes

  - name: Installing jdk17 in Jenkins server
    apt:
      name: openjdk-17-jdk
      update_cache: yes
    become: yes

  - name: Installing jenkins apt repository key
    apt_key:
      url: https://pkg.jenkins.io/debian/jenkins.io-2023.key
      state: present
    become: yes

  - name: Configuring the apt repository
    apt_repository:
      repo: deb https://pkg.jenkins.io/debian binary/
      filename: /etc/apt/sources.list.d/jenkins.list
      state: present
    become: yes

  - name: Update apt-get repository with "apt-get update"
    apt:
      update_cache: yes

  - name: Finally, its time to install Jenkins
    apt: name=jenkins update_cache=yes
    become: yes

  - name: Jenkins is installed. Lets start 'Jenkins' now!
    service: name=jenkins state=started

  - name: Wait until the file /var/lib/jenkins/secrets/initialAdminPassword is present before continuing
    wait_for:
      path: /var/lib/jenkins/secrets/initialAdminPassword

  - name: You can find Jenkins admin password under 'debug'
    command: cat /var/lib/jenkins/secrets/initialAdminPassword
    register: out

  - debug: var=out.stdout_lines


- name: Start the Docker installation steps
  hosts: docker-server
  become: yes
  become_method: sudo
  gather_facts: no

  tasks:

  - name: Update 'apt' repository with latest versions of packages
    apt:
      update_cache: yes

  - name: install docker prerequisite packages
    apt:
      name: ['ca-certificates', 'curl', 'gnupg', 'lsb-release']
      update_cache: yes
      state: latest

  - name: Install the docker apt repository key
    apt_key: url=https://download.docker.com/linux/ubuntu/gpg state=present
    become: yes

  - name: Configure the apt repository
    apt_repository:
      repo: deb https://download.docker.com/linux/ubuntu bionic stable
      state: present
    become: yes

  - name: Update 'apt' repository
    apt:
      update_cache: yes

  - name: Install Docker packages
    apt:
      name: ['docker-ce', 'docker-ce-cli', 'containerd.io']
      update_cache: yes
    become: yes

  - name: Install jdk17 in Docker server. Maven needs this.
    apt:
      name: openjdk-17-jre-headless
      update_cache: yes
    become: yes

  - name: Start Docker service
    service:
      name: docker
      state: started
      enabled: yes

  - lineinfile:
       dest: /lib/systemd/system/docker.service
       regexp: '^ExecStart='
       line: 'ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:4243 -H unix:///var/run/docker.sock'

  - name: Reload systemd
    command: systemctl daemon-reload

  - name: docker restart
    service:
      name: docker
      state: restarted
...
```
#### Step-03: Run the above playbook to deploy the packages onto target servers.
```
ansible-playbook DevOpsSetup.yml
```
At the end of this step, the `Docker-Server` and `Jenkins-Server` will be ready for performing further Labs

#### Step-04:

**1. Verify the Jenkins Landing Page:**

   * Open a web browser and navigate to your Jenkins landing page using your Jenkins server's public IP address. Replace `<Your_Jenkins_IP>` with the actual public IP.
   
   ```
   http://<Your_Jenkins_IP>:8080/
   ```

**2. Verify the Docker Landing Page:**

   * Open a web browser and access the Docker landing page using your Docker server's public IP address. Replace `<Your_Docker_IP>` with the actual public IP.
   
   ```
   http://<Your_Docker_IP>:4243/version
   ```

#### =============================END of LAB-01=============================
---