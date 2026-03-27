# Lab: AWS Network Load Balancer (NLB) Setup

🎯 Objective
	•	Create 2 EC2 instances
	•	Install a simple web server
	•	Create a Network Load Balancer
	•	Route traffic to instances
	•	Verify load balancing

⸻

🧱 Architecture

User → NLB → Target Group → EC2 Instances (2)


⸻

🪜 Step 1: Create EC2 Instances
	1.	Go to Amazon Web Services Console
	2.	Navigate to EC2 → Launch Instance

Create 2 instances:
	•	Name: nlb-instance-1, nlb-instance-2
	•	AMI: Amazon Linux 2
	•	Instance type: t2.micro (free tier)
	•	Key pair: create/select
	•	Network: default VPC
	•	Auto-assign public IP: Enable

Security Group:

Allow:
	•	SSH (22) → your IP
	•	HTTP (80) → Anywhere

⸻

🧪 Step 2: Install Web Server

SSH into both instances and run:

sudo yum update -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd

echo "Hello from $(hostname)" | sudo tee /var/www/html/index.html

👉 Each instance will show different hostname → helps verify load balancing

⸻

🌐 Step 3: Test Instances Individually
	•	Open browser:
	•	http://<instance-1-public-ip>
	•	http://<instance-2-public-ip>

✅ You should see different outputs

⸻

🎯 Step 4: Create Target Group
	1.	Go to EC2 → Target Groups
	2.	Click Create target group

Configure:
	•	Target type: Instances
	•	Name: nlb-target-group
	•	Protocol: TCP
	•	Port: 80
	•	VPC: default

Health Check:
	•	Protocol: HTTP
	•	Path: /

Register targets:
	•	Select both EC2 instances
	•	Click Include as pending below
	•	Create target group

⸻

⚖️ Step 5: Create Network Load Balancer
	1.	Go to EC2 → Load Balancers
	2.	Click Create Load Balancer
	3.	Choose Network Load Balancer

⸻

Configure NLB:
	•	Name: my-nlb
	•	Scheme: Internet-facing
	•	IP type: IPv4

Listeners:
	•	Protocol: TCP
	•	Port: 80

Availability Zones:
	•	Select at least 2 AZs
	•	Choose subnets

⸻

Attach Target Group:
	•	Select nlb-target-group

Click Create

⸻

⏳ Step 6: Wait for Health Checks
	•	Go to Target Group → Targets tab
	•	Wait until:

Status: healthy



⸻

🌍 Step 7: Test Load Balancer
	1.	Copy DNS name of NLB
	2.	Open in browser:

http://<nlb-dns-name>

	3.	Refresh multiple times

✅ You should see:

Hello from ip-xxx-xxx-xxx-1
Hello from ip-xxx-xxx-xxx-2

👉 That means load balancing is working 🎉

