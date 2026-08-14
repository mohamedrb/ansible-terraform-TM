# ansible-terraform-TM
ansible-terraform-TM
﻿# travelmemory-deployment-ansible-terraform
Project Statement
MERN Application to use: https://github.com/UnpredictablePrashant/TravelMemory
Tasks:
Part 1: Infrastructure Setup with Terraform
1. AWS Setup and Terraform Initialization:
   - Configure AWS CLI and authenticate with your AWS account.
   - Initialize a new Terraform project targeting AWS.
2. VPC and Network Configuration:
   - Create an AWS VPC with two subnets: one public and one private.
   - Set up an Internet Gateway and a NAT Gateway.
   - Configure route tables for both subnets.
3. EC2 Instance Provisioning:
   - Launch two EC2 instances: one in the public subnet (for the web server) and another in the private subnet (for the database).
   - Ensure both instances are accessible via SSH (public instance only accessible from your IP).
4. Security Groups and IAM Roles:
   - Create necessary security groups for web and database servers.
   - Set up IAM roles for EC2 instances with required permissions.
5. Resource Output:
   - Output the public IP of the web server EC2 instance.
  

Project solution:
Architecture overview

Internet
   ↓
[Internet Gateway]
   ↓
[Public Subnet]
   → EC2 (Node.js MERN App)
   ↓ (via NAT)
[Private Subnet]
   → EC2 (MongoDB)

Step: 1:
AWS Setup + Terraform Init

Aws configure
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/fb30cd79-6f48-4d1e-9966-18474b480717" />
Project Structure
terraform-mern/
│── main.tf
│── variables.tf
│── outputs.tf
│── provider.tf

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/68c06f4e-061e-40dd-ba71-f4e29160fa07" />

Provider.tf
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/0449ddd5-af42-42ac-9e01-9be9aa6987ff" />


Step: main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.39.0"
    }
  }
}

# vpc
resource "aws_vpc" "mern_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "mern-vpc"
    name = "mern-vpc"
  }
}

# Public Subnet
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.mern_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone = "ap-south-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "mern-public-subnet"
  }
  
}

# private subnet
resource "aws_subnet" "private_subnet" {
  vpc_id     = aws_vpc.mern_vpc.id
  cidr_block = "10.0.2.0/24"
  availability_zone = "ap-south-1a"
   tags = {
    Name = "mern-private-subnet"
  }
}

# internet gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.mern_vpc.id
   tags = {
    Name = "mern-igw"
  }
}

# Route Table (Public)
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.mern_vpc.id
   tags = {
    name = "mern-public-rt"
  }
}

resource "aws_route" "public_route" {
  route_table_id         = aws_route_table.public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

# NAT Gateway (for private subnet)
resource "aws_eip" "nat_eip" {
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public_subnet.id
   tags = {
    Name = "mern-natgw"
  }
}

# Private Route Table
resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.mern_vpc.id
   tags = {
    name = "mern-private-rt"
  }
}

resource "aws_route" "private_route" {
  route_table_id         = aws_route_table.private_rt.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat.id
}

resource "aws_route_table_association" "private_assoc" {
  subnet_id      = aws_subnet.private_subnet.id
  route_table_id = aws_route_table.private_rt.id
}


# security groups

# frontend-sg creation
resource "aws_security_group" "web_sg" {
  vpc_id = aws_vpc.mern_vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 3000
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

   tags = {
    name = "mern-frontend-sg"
  }
}

# database sg creation:
resource "aws_security_group" "db_sg" {
  vpc_id = aws_vpc.mern_vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 3001
    to_port     = 3001
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    name = "mern-backend-sg"
  }
}

# ec2 instance creation:
# frontend ec2 instance
resource "aws_instance" "frontend" {
  ami           = "ami-05d2d839d4f73aafb" # Amazon Linux
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_subnet.id
  key_name      = "mern-key"

  associate_public_ip_address = true

  vpc_security_group_ids = [aws_security_group.web_sg.id]
  tags = {
    name = "mern-frontend"
  }
}

# backend ec2 instance creation
resource "aws_instance" "backend" {
  ami           = "ami-05d2d839d4f73aafb"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.private_subnet.id
  key_name      = "mern-key"

  vpc_security_group_ids = [aws_security_group.db_sg.id]
  tags = {
    name = "mern-backend"
  }
}

# iam role
resource "aws_iam_role" "ec2_role" {
  name = "ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Effect = "Allow",
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}






Outputs.tf
output "frontend_public_ip" {
  value = aws_instance.frontend.public_ip
}
output "backend_public_ip" {
  value = aws_instance.backend.public_ip
}

Folder structure
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/1eb0773d-1913-462b-85a9-0c3d8cc5a39f" />

Part-2 Project Implementation using Ansible
Windows:
1.	wsl --install -d Ubuntu
2.	sudo apt update
3.	sudo apt install ansible -y
4.	ansible –version
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/dceaa9e5-3581-4669-bbd2-2400a6b48c4b" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/6d84c796-7022-4810-94ba-4a1166523640" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/df609d41-50a9-40f7-910d-48eac7444052" />


Creating hosts.ini
[web]
bastion ansible_host=13.233.92.171 ansible_user=ubuntu ansible_ssh_private_key_file=~/mern-key.pem

[db]
private1 ansible_host=10.0.2.158 ansible_user=ubuntu ansible_ssh_private_key_file=~/mern-key.pem ansible_ssh_common_args='-o ProxyCommand="ssh -i ~/mern-key.pem ubuntu@13.233.92.171 -W %h:%p"'
Creating db.yml for database mongodb configuration on aws instance in private subnet
- name: Setup MongoDB Server
  hosts: db
  become: yes

  vars:
    mongo_admin_user: admin
    mongo_admin_password: password123   # ⚠️ change in real projects

  tasks:

    - name: Update apt cache
      apt:
        update_cache: yes

    #Install dependencies
    - name: Install required packages
      apt:
        name:
          - gnupg
          - curl
        state: present

    # Add MongoDB GPG key and official repo only on supported distro (jammy)
    - name: Import MongoDB GPG key
      shell: |
        curl -fsSL https://pgp.mongodb.com/server-6.0.asc | \
        gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
      args:
        creates: /usr/share/keyrings/mongodb-server-6.0.gpg

    - name: Add MongoDB repository
      copy:
        dest: /etc/apt/sources.list.d/mongodb-org-6.0.list
        content: |
          deb [ arch=amd64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse


    - name: Update apt cache again
      apt:
        update_cache: yes

    # Install MongoDB: use official mongodb-org for jammy, otherwise install distro package
    - name: Install MongoDB
      apt:
        name: mongodb-org
        state: present


       

    - name: Enable and start MongoDB service
      systemd:
        name: mongod
        state: started
        enabled: yes

    # Create admin user using mongosh if available, otherwise mongo
    - name: Create MongoDB Admin User
      shell: |
        if command -v mongosh >/dev/null 2>&1; then
          mongosh --eval "db = db.getSiblingDB('admin'); if (!db.getUser('{{ mongo_admin_user }}')) { db.createUser({ user: '{{ mongo_admin_user }}', pwd: '{{ mongo_admin_password }}', roles: [{ role: 'root', db: 'admin' }] }); }"
        elif command -v mongo >/dev/null 2>&1; then
          mongo --eval "db = db.getSiblingDB('admin'); if (!db.getUser('{{ mongo_admin_user }}')) { db.createUser({ user: '{{ mongo_admin_user }}', pwd: '{{ mongo_admin_password }}', roles: [{ role: 'root', db: 'admin' }] }); }"
        else
          echo "No mongo shell available" >&2
          exit 1
        fi
      args:
        executable: /bin/bash

    - name: Allow remote access in MongoDB config
      replace:
        path: /etc/mongod.conf
        regexp: '^  bindIp: 127.0.0.1'
        replace: '  bindIp: 0.0.0.0'

 

    #Firewall setup
    - name: Install UFW
      apt:
        name: ufw
        state: present

    - name: Enable UFW
      ufw:
        state: enabled
        policy: allow

    - name: Allow required ports
      ufw:
        rule: allow
        port: "{{ item }}"
      loop:
        - 22
        - 80
        - 443
        - 3000
        - 3001
        - 27017

    # SSH security
    - name: Disable root SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
        backup: yes

    - name: Restart SSH service
      service:
        name: ssh
        state: restarted


Creating web.yml for ec2 instance in public subnet

- name: Setup Web Server
  hosts: web
  become: yes

  vars:
    app_dir: /home/ubuntu/app
    frontend_dir: "{{ app_dir }}/frontend"
    backend_dir: "{{ app_dir }}/backend"
    backend_url: "http://13.206.107.143:3001"

  tasks:

    # ================= BASIC SETUP =================

    # - name: Update apt cache
    #   apt:
    #     update_cache: yes

    - name: Install required packages
      apt:
        name:
          - curl
          - git
        state: present

    - name: Add NodeSource repo
      shell: curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
      args:
        executable: /bin/bash

    - name: Install Node.js
      apt:
        name: nodejs
        state: present

    # ================= CLONE CODE =================

    - name: Clone Repository
      git:
        repo: "https://github.com/saiyedin786/TravelMemory_prashant.git"
        dest: "{{ app_dir }}"
        force: yes

    - name: Set ownership to ubuntu
      file:
        path: "{{ app_dir }}"
        owner: ubuntu
        group: ubuntu
        recurse: yes

    # ================= PM2 SETUP =================

    - name: Install PM2 globally
      npm:
        name: pm2
        global: yes
      become: yes

    - name: Set npm global bin path manually
      set_fact:
        npm_global_bin: "/usr/bin"

    # ================= BACKEND =================

    - name: Install Backend Dependencies
      command: npm install
      args:
        chdir: "{{ backend_dir }}"
      become_user: ubuntu

    # - name: Stop existing backend
    #   shell: |
    #     pm2 PATH={{ npm_global_bin.stdout }}:$PATH pm2 delete backend || true
    #   become_user: ubuntu

    - name: Start Backend
      command: pm2 start index.js --name backend
      args:
        chdir: "{{ backend_dir }}"
      become_user: ubuntu

    # ================= FRONTEND =================

    - name: Install Frontend Dependencies
      command: npm install
      args:
        chdir: "{{ frontend_dir }}"
      become_user: ubuntu

    # - name: Create frontend .env file
    #   copy:
    #     dest: "{{ frontend_dir }}/.env"
    #     content: |
    #       REACT_APP_API_URL={{ backend_url }}
    #   become_user: ubuntu

    - name: Build Frontend
      command: npm run build
      args:
        chdir: "{{ frontend_dir }}"
      become_user: ubuntu

    - name: Install serve globally
      npm:
        name: serve
        global: yes
      become: yes

    # - name: Stop existing frontend
    #   shell: |
    #     pm2 PATH={{ npm_global_bin.stdout }}:$PATH pm2 delete frontend || true
    #   become_user: ubuntu

    - name: Start Frontend
      command: pm2 start npx --name frontend -- serve -s /home/ubuntu/app/frontend/build -l 3000
      args:
        chdir: "{{ frontend_dir }}"
      become_user: ubuntu

    # ================= PM2 SAVE =================

    - name: Save PM2 process list
      command: pm2 save
      become_user: ubuntu

    - name: Setup PM2 startup
      command: pm2 startup systemd -u ubuntu --hp /home/ubuntu





Terraform commands:
1.	terraform init
2.	terraform plan
3.	terraform apply

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/d5880196-43a8-40e5-b9ef-fb6857a10b49" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/be5f056e-3e12-4b1f-8feb-2247901ea3bc" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/1f8983d7-b038-4532-9834-237df7b37246" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/e0bdb7b2-e402-40ab-97a5-533abe41645e" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/a3de39f7-1f66-4a90-82ba-c63b3a4095f5" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/a607fb1e-a991-4d47-89bb-eed422188fa9" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/92e843cf-df84-4000-bd3f-303db777f597" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/15c5858d-3a44-4a1c-a498-c0d70413e550" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/815b8511-f678-468d-ab2f-1feaf1113803" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/e85bafd0-8682-4b3e-8b72-7755ffbd52f7" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/7c6e8f48-a16b-4f7b-808b-ebe8a8e2c721" />

Connecting to ec2 instance in public subnet and configuration as a bastion host
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/5ce555e9-e1f9-452e-abfc-bbe9a87d5cb7" />

Copying the content of mern-key.pem private key into the bastion host
Touch mern-key.pm
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/ba71f385-43ea-441a-82c6-35c5edefba91" />

Connecting from bastion host to ec2 instance in private subnet

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/7feb76f7-5c2b-4f04-a130-1c605aeca158" />

Now my ansible files are
Db.yml – this will create install mongodb in ec2 instance in private subnet
Web.yml :- this will configure frontend and backend in ec2 instance in public subnet

Ansible commands
1.	ansible-playbook -i hosts.ini db.yml
2.	ansible-playbook -i hosts.ini web.yml

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/8d6e87da-96d6-48f6-babc-45aa8a5cf7f2" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/d5f5d3b6-2bea-4594-badb-6562f0d68228" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/a1f7753e-504c-4acd-a5a5-f6cb96bb8a0d" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/122651be-bb71-4865-895a-dd9d68e9bb3b" />

Testing from the browser:
http://3.110.83.171:3000/
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/85b75046-d5f6-4d35-b3ca-8da58a8a89b1" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/1126c76e-4099-4a15-a2b4-d1c02e17a9dd" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/3d9d2146-4f40-4175-a3b3-f5beebad143d" />














