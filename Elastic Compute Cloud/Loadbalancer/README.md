
#  AWS Network Load Balancer (NLB) Lab

## 🎯 Objective
- Create 2 EC2 instances
- Install a web server
- Create a Network Load Balancer (NLB)
- Route traffic to instances
- Verify load balancing

---

## 🧱 Architecture

User → NLB → Target Group → EC2 Instances (2)

---

## 🪜 Step 1: Create EC2 Instances

1. Go to AWS Console → EC2 → Launch Instance

### Create 2 instances:
- Name: `nlb-instance-1`, `nlb-instance-2`
- AMI: Amazon Linux 2
- Instance type: t2.micro
- Key pair: create/select
- Network: default VPC
- Auto-assign public IP: Enable

### Security Group:
Allow:
- SSH (22) → your IP  
- HTTP (80) → Anywhere  

---

## 🧪 Step 2: Install Web Server

SSH into both instances and run:

```bash
sudo yum update -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd

echo "Hello from $(hostname)" | sudo tee /var/www/html/index.html


⸻

🌐 Step 3: Test Instances Individually

Open in browser:

http://<instance-1-public-ip>
http://<instance-2-public-ip>

✅ Each instance should show a different hostname

⸻

🎯 Step 4: Create Target Group
	1.	Go to EC2 → Target Groups → Create

Configuration:
	•	Target type: Instances
	•	Name: nlb-target-group
	•	Protocol: TCP
	•	Port: 80
	•	VPC: default

Health Check:
	•	Protocol: HTTP
	•	Path: /

Register Targets:
	•	Select both EC2 instances
	•	Click Include as pending below
	•	Create target group

⸻

⚖️ Step 5: Create Network Load Balancer
	1.	Go to EC2 → Load Balancers → Create
	2.	Choose Network Load Balancer

Configuration:
	•	Name: my-nlb
	•	Scheme: Internet-facing
	•	IP type: IPv4

Listener:
	•	Protocol: TCP
	•	Port: 80

Availability Zones:
	•	Select at least 2 AZs
	•	Choose subnets

Attach Target Group:
	•	Select nlb-target-group

Click Create

⸻

⏳ Step 6: Wait for Health Checks
	•	Go to Target Group → Targets tab
	•	Wait until status shows:

healthy


⸻

🌍 Step 7: Test Load Balancer
	1.	Copy the DNS name of the NLB
	2.	Open in browser:

http://<nlb-dns-name>

	3.	Refresh multiple times

✅ Expected Output:

Hello from ip-xxx-xxx-xxx-1
Hello from ip-xxx-xxx-xxx-2


⸻
