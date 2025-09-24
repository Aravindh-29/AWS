# AWS

<img width="918" height="710" alt="image" src="https://github.com/user-attachments/assets/d010d211-7c0b-442e-88de-7e2383f2914b" />

**full end-to-end setup** **.NET app on port 5000**,
**VPC → Subnets → NAT → Route Tables → SGs → Launch Template → ASG → ALB → RDS**
---

# 1️⃣ VPC

* **Name:** `dotnet-vpc`
* **CIDR:** `10.0.0.0/16`
* Region: Mumbai (`ap-south-1`)

---

# 2️⃣ Subnets (6 total)

| Type           | AZ          | CIDR         | Notes              |
| -------------- | ----------- | ------------ | ------------------ |
| Public-1a      | ap-south-1a | 10.0.1.0/24  | Auto-assign IPv4 ✅ |
| Public-1b      | ap-south-1b | 10.0.2.0/24  | Auto-assign IPv4 ✅ |
| Private-App-1a | ap-south-1a | 10.0.11.0/24 | App EC2 instances  |
| Private-App-1b | ap-south-1b | 10.0.12.0/24 | App EC2 instances  |
| Private-DB-1a  | ap-south-1a | 10.0.21.0/24 | RDS DB             |
| Private-DB-1b  | ap-south-1b | 10.0.22.0/24 | RDS DB             |

---

# 3️⃣ Internet Gateway (IGW)

* Create `dotnet-igw`
* Attach to `dotnet-vpc`

---

# 4️⃣ NAT Gateways

* Allocate **2 Elastic IPs**
* NAT-1a → Public-1a
* NAT-1b → Public-1b

---

# 5️⃣ Route Tables

* **public-rt:** `0.0.0.0/0 → IGW` → associate **Public-1a & 1b**
* **private-app-rt-1a:** `0.0.0.0/0 → NAT-1a` → associate Private-App-1a
* **private-app-rt-1b:** `0.0.0.0/0 → NAT-1b` → associate Private-App-1b
* **private-db-rt-1a:** local only → associate Private-DB-1a
* **private-db-rt-1b:** local only → associate Private-DB-1b

> DB subnets ki internet direct access ledhu.

---

# 6️⃣ Security Groups (SG)

### ALB-SG

* Inbound: HTTP 80 from `0.0.0.0/0`
* Outbound: All (or App-SG 5000)

### App-SG

* Inbound:

  * Port 5000 from **ALB-SG**
  * SSH 22 from your IP (optional, SSM better)
* Outbound:

  * MySQL 3306 → **DB-SG**
  * HTTP/HTTPS → Internet via NAT

### DB-SG

* Inbound: MySQL 3306 from **App-SG only**
* Outbound: None / restricted

---

# 7️⃣ Launch Template (EC2 .NET app)

* **Name:** `dotnet-app-template`
* AMI: Amazon Linux 2 / Ubuntu
* Instance type: t3.micro (testing)
* SG: App-SG
* IAM Role: EC2SSM
* User Data (cloud-init):

```bash
#!/bin/bash

# --------------------------
# 1. Update and Upgrade OS
# --------------------------
sudo apt update -y && sudo apt upgrade -y

# --------------------------
# 2. Install Git and .NET SDK 8
# --------------------------
sudo apt install -y git wget
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt update -y
sudo apt install -y dotnet-sdk-8.0

# --------------------------
# 3. Clone Your Git Repo
# --------------------------
cd /home/ubuntu
if [ ! -d "DbTestApp" ]; then
  git clone https://github.com/Aravindh-29/DbTestApp.git
fi
cd DbTestApp

# --------------------------
# 4. Publish the App
# --------------------------
sudo dotnet publish -c Release -o /home/ubuntu/published

# --------------------------
# 5. Create systemd Service
# --------------------------
sudo tee /etc/systemd/system/dbconnection.service > /dev/null <<EOL
[Unit]
Description=DbConnectionTester .NET App
After=network.target

[Service]
WorkingDirectory=/home/ubuntu/published
ExecStart=/usr/bin/dotnet /home/ubuntu/published/DbConnectionTester.dll --urls "http://0.0.0.0:5000"
Restart=always
RestartSec=10
User=ubuntu
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
EOL

# Reload systemd, enable and start service
sudo systemctl daemon-reload
sudo systemctl enable dbconnection.service
sudo systemctl start dbconnection.service

echo "✅ Setup complete! App is running and accessible via http://<EC2-Public-IP>:5000"

```

> Use **Secrets Manager** for DB credentials, replace in appsettings.json.

---

# 8️⃣ Target Group

* Name: `dotnet-app-tg`
* VPC: dotnet-vpc
* Protocol: HTTP
* Port: 5000
* Target type: Instance
* Health check: `/`

---

# 9️⃣ Application Load Balancer (ALB)

* Name: `dotnet-alb`
* Scheme: Internet-facing
* Subnets: Public-1a, Public-1b
* SG: ALB-SG

**Listener:** Port 80 → Forward to `dotnet-app-tg`

---

# 🔟 Auto Scaling Group (ASG)

* Name: `dotnet-app-asg`
* Launch Template: `dotnet-app-template`
* Subnets: Private-App-1a, Private-App-1b
* Desired: 2, Min:2, Max:4
* Attach Target Group: `dotnet-app-tg`
* Health check type: ELB

> ASG automatically registers instances with Target Group. Terminate instance → ASG launches new → ALB automatically updates.

---

# 1️⃣1️⃣ RDS MySQL (Multi-AZ)

* Subnet group: Private-DB-1a & 1b
* Engine: MySQL (choose version)
* Instance class: db.t3.micro (testing)
* Multi-AZ: Enable
* Storage: gp3
* Public access: No
* SG: DB-SG
* Credentials: store securely, optionally in Secrets Manager

> Copy endpoint → put in appsettings.json or fetch via IAM + Secrets Manager.

---

# 1️⃣2️⃣ Testing

1. Wait for RDS available
2. ASG boots 2 instances → EC2 logs check via SSM
3. ALB health check pass → targets healthy
4. Browser → `http://<ALB-DNS>` → should see .NET app running on port 5000

---

# 1️⃣3️⃣ Extra Best Practices

* Use SSM instead of SSH
* Use AWS Secrets Manager for DB credentials
* Enable RDS automated backups
* Use CloudWatch alarms for CPU, unhealthy hosts, storage
* Production: use HTTPS on ALB (ACM certificate)
* Pre-baked AMI for production apps instead of heavy user-data

---

# ✅ Quick Validation Checklist

* [ ] Public subnets → public-rt
* [ ] NAT gateways → private-app RTs
* [ ] ALB SG → port 80 inbound 0.0.0.0/0
* [ ] App-SG → ALB-SG port 5000 inbound
* [ ] RDS → private-db subnets, DB-SG 3306 inbound
* [ ] ASG → attached to Target Group, min 2 max 4
* [ ] User-data → installs .NET app on 5000

---
**ALB ni Route53 hosted zone lo map cheyyadam step-by-step**.
---

## 1️⃣ Pre-requisites

* Domain: `aravindh.xyz` (already bought in Dotpapa).
* Hosted zone: create chesav or AWS create chesindi after NS update.
* ALB: `dotnet-alb` created already, **DNS name** untundi like:

  ```
  dotnet-alb-123456789.ap-south-1.elb.amazonaws.com
  ```

---

## 2️⃣ Route53 lo A Record create cheyyadam

1. **Open AWS Route53** → Hosted Zones → `aravindh.xyz`.
2. Click **Create Record**.
3. **Record name:** (blank) for root domain `aravindh.xyz`
   or `www` for `www.aravindh.xyz`.
4. **Record type:** `A – IPv4 address`.
5. **Value/Route traffic to:**

   * Select **Alias** → **Application and Classic Load Balancer**.
   * Choose your region (same region as ALB).
   * Select your ALB `dotnet-alb`.
6. TTL: default (60 sec).
7. Save record.

---

## 3️⃣ Propagation check

* DNS changes may take **a few minutes**.
* Verify using:

  ```bash
  nslookup aravindh.xyz
  nslookup www.aravindh.xyz
  ```
* Browser lo open cheyyi: `http://aravindh.xyz` → it should reach ALB → app.

---

## 4️⃣ Optional: Redirect root → www

* Create another **A record** for `www.aravindh.xyz` → ALB.
* Or create **CNAME** for `www` → root.

---

## 5️⃣ Full flow

🌍 User → DNS lookup (`aravindh.xyz` in Route53) → ALB DNS → Target Group (5000) → App instance.

---


