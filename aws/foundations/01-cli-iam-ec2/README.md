# 🚀 AWS Foundations: CLI Setup, IAM & Secure EC2 Access

Hosted by **Nayan M.**
📅 Friday, Jan 23 | 6:00 PM – 8:00 PM EST
📍 Microsoft Teams

---

## 🎥 Workshop Resources

| Resource             | Link                                           |
| -------------------- | ---------------------------------------------- |
| 📄 Workshop Guide    | This README                                    |
| 🧩 Scripts Used      | Included in this folder                        |

---

## 📌 Overview

This hands-on workshop introduces **core AWS concepts** using **industry-standard, CLI-first workflows**.

Instead of relying heavily on the AWS Console, this session focuses on real-world cloud engineering practices using the AWS CLI.

By the end of this workshop, you will:

* Configure AWS locally using AWS CLI
* Create IAM users securely
* Launch an EC2 instance using the CLI
* Deploy a static website using user-data
* Connect securely via SSH

---

## 🧠 Prerequisites

* Active AWS account
* Laptop with terminal access
* Stable internet connection
* Basic command-line familiarity (helpful, not required)

---

## 🏗 What We Will Build

* AWS CLI configured locally
* IAM user with appropriate permissions
* EC2 instance provisioned via CLI
* Security Group configured for HTTP + SSH
* Static website deployed automatically
* Secure SSH access to EC2

---

# 🛠 Step 1 – Install AWS CLI

### macOS

```bash
brew install awscli
aws --version
```

### Windows

Download and install the AWS CLI MSI installer from the official AWS website.

Verify installation:

```bash
aws --version
```

### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install awscli -y
aws --version
```

### Amazon Linux

```bash
sudo yum install awscli -y
aws --version
```

---

# 🔐 Step 2 – Configure AWS CLI

### Recommended

```bash
aws login
```

If needed:

```bash
aws configure
```

Use:

* Region: `us-east-1`
* Output: `json`

Verify configuration:

```bash
aws sts get-caller-identity
```

---

# 🔑 Step 3 – Create EC2 Key Pair

```bash
aws ec2 create-key-pair \
  --key-name workshop-key \
  --query 'KeyMaterial' \
  --output text > workshop-key.pem
```

### Set Proper Permissions

Linux/macOS:

```bash
chmod 400 workshop-key.pem
```

Windows:

```bash
icacls workshop-key.pem /inheritance:r
icacls workshop-key.pem /grant:r "%USERNAME%:R"
```

---

# 🔒 Step 4 – Create Security Group

```bash
aws ec2 create-security-group \
  --group-name workshop-sg \
  --description "Workshop security group"
```

Allow SSH:

```bash
aws ec2 authorize-security-group-ingress \
  --group-name workshop-sg \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

Allow HTTP:

```bash
aws ec2 authorize-security-group-ingress \
  --group-name workshop-sg \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

---

# 🌐 Step 5 – Create User Data Script

Create a file named:

```
userdata.sh
```

Open it:

Linux/macOS:

```bash
nano userdata.sh
```

Windows:

```bash
notepad userdata.sh
```

Paste the following:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from AWS EC2 🚀</h1>" > /var/www/html/index.html
```

Save and exit.

---

# 🖥 Step 6 – Launch EC2 Instance

```bash
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t2.micro \
  --key-name workshop-key \
  --security-groups workshop-sg \
  --user-data file://userdata.sh
```

To retrieve instance details:

```bash
aws ec2 describe-instances
```

Copy the **Public IP address**.

---

# 🔑 Step 7 – SSH Into EC2

```bash
ssh -i workshop-key.pem ec2-user@<PUBLIC-IP>
```

---

# 🌍 Step 8 – Access the Website

Open your browser:

```
http://<PUBLIC-IP>
```

You should see:

```
Hello from AWS EC2 🚀
```

---

# ⚠️ Troubleshooting

| Issue                   | Solution                     |
| ----------------------- | ---------------------------- |
| Permission denied (SSH) | `chmod 400 workshop-key.pem` |
| SSH timeout             | Ensure port 22 is open       |
| Website not loading     | Ensure port 80 is open       |
| CLI errors              | Check region & credentials   |

---

# 💰 Important – Avoid Charges

Terminate your instance after the workshop:

```bash
aws ec2 terminate-instances --instance-ids <INSTANCE-ID>
```

---

# 🔮 What’s Next?

Future AWS workshops will cover:

* IAM Roles
* EC2 to S3 integration
* Terraform basics
* CI/CD pipelines
* Production security hardening

---

# 👨‍💻 Author

**Nayan M.** <br>
Cloud & DevOps Workshop Series

---

# ⭐ Support

If this workshop helped you:

* Star the repository
* Share with peers
* Join upcoming sessions

---
