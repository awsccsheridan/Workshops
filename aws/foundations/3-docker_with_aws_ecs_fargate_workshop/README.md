# 🚀 Docker → AWS ECS Fargate Workshop

## 📌 Workshop Overview

In this hands-on workshop, you will:

- Containerize a Python Flask app using Docker 🐳  
- Push it to AWS ECR 📦  
- Deploy it using ECS Fargate ☁️  
- Access your app publicly 🌍  

👉 By the end, you’ll have a **live app running on the internet**

---

# 🎯 Learning Objectives

- Understand Docker basics (images vs containers)
- Build and run containers locally
- Push images to AWS ECR
- Deploy serverless containers using ECS Fargate
- Expose your app using a Load Balancer

---

# ⚙️ Prerequisites

## 🐳 Docker Desktop
Download: https://www.docker.com/products/docker-desktop  
Verify:
```bash
docker --version
```

## 🧰 Git
Download: https://git-scm.com/downloads  
Verify:
```bash
git --version
```

## ☁️ AWS CLI
Install: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html  
Verify:
```bash
aws --version
```

## 🔑 AWS Account
Create: https://aws.amazon.com/free  
👉 Use region: `us-east-1`

## 🐍 Python (3.10+ recommended)
Download: https://www.python.org/downloads  
Verify:
```bash
python --version
```

---

# 🔐 AWS CLI LOGIN (REQUIRED)

We will use **browser-based login** via `aws login`.

## ▶️ Step 0.1 — Login

```bash
aws login
```

- A browser window will open  
- Sign into your AWS account  
- Allow CLI access  

## ▶️ Step 0.2 — Verify

```bash
aws sts get-caller-identity
```

✅ You should see your account ID and user/role.

> If `aws login` is not available in your CLI, ask the instructor for help.

---

# 📦 Starter Code

Create a new folder and add the following files.

## 🐍 app.py
```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello AWS Cloud Club! 🚀"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

## 📄 requirements.txt
```
flask==3.0.3
```

## 🐳 Dockerfile
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

---

# 🛠️ Hands-On Steps

## ▶️ Step 1 — Run Locally (without Docker)

```bash
pip install flask
python app.py
```

Open: http://localhost:5000  
You should see:
```
Hello AWS Cloud Club! 🚀
```

---

## ▶️ Step 2 — Build Docker Image

```bash
docker build -t my-flask-app .
```

---

## ▶️ Step 3 — Run Docker Container

```bash
docker run -p 5000:5000 my-flask-app
```

Open again: http://localhost:5000

---

## ▶️ Step 4 — Create Amazon ECR Repository

AWS Console → **ECR → Create repository**

- Name: `my-flask-app`
- Visibility: Private

Copy the **Repository URI** (you will use it next).

---

## ▶️ Step 5 — Authenticate Docker to AWS

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com
```

---

## ▶️ Step 6 — Tag & Push Image

```bash
docker tag my-flask-app <repository-uri>
docker push <repository-uri>
```

---

## ▶️ Step 7 — Create ECS Cluster

ECS → **Create Cluster**

- Type: Fargate
- Name: `my-cluster`

---

## ▶️ Step 8 — Create Task Definition

ECS → **Task Definitions → Create**

- Launch type: Fargate
- Container image: your ECR URI
- Container port: `5000`
- CPU: `0.25 vCPU`
- Memory: `0.5 GB`

---

## ▶️ Step 9 — Create Service

Cluster → **Create Service**

- Launch type: Fargate
- Number of tasks: `1`
- Assign public IP: **ENABLED**

### 🔥 Load Balancer

- Type: Application Load Balancer
- Listener: HTTP (port 80)
- Target group port: `5000`

### 🔐 Security Group (IMPORTANT)

Allow inbound:
```
HTTP (Port 80) → 0.0.0.0/0
```

---

## ▶️ Step 10 — Access Your App 🌍

Copy the **Load Balancer DNS** and open:

```
http://<load-balancer-dns>
```

🎉 You should see:
```
Hello AWS Cloud Club! 🚀
```

---

# 🛠️ Troubleshooting

- **App not loading?** Check port `5000` and target group mapping  
- **Container stops?** Check ECS → Tasks → Logs  
- **Can’t access URL?** Ensure security group allows HTTP (80)  
- **Push failed?** Re-run ECR login command  

---

# 🧹 Cleanup (Avoid Charges)

Delete:
- ECS Service  
- ECS Cluster  
- Load Balancer  
- Target Group  
- ECR Repository  

---

# 🧠 Architecture Flow

```
Local → Docker → ECR → ECS (Fargate) → Load Balancer → Internet
```

---

# 🔥 Pro Tips

- Use `0.0.0.0` in Flask (not localhost)
- Keep ports consistent (5000 everywhere)
- Fargate = serverless containers (no EC2 required)
- Load Balancer makes your app publicly accessible
