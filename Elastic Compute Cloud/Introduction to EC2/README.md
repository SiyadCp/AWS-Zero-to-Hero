# Introduction to Amazon EC2

---

## 1. What is Virtualisation?
![Virtualisation concept](https://upload.wikimedia.org/wikipedia/commons/thumb/0/08/Hardware_Virtualization_%28copy%29.svg/800px-Hardware_Virtualization_%28copy%29.svg.png)

Virtualisation is the process of creating a virtual version of a physical resource — such as a server, storage, or network — using software. Instead of running one operating system on one physical machine, virtualisation allows multiple operating systems to run simultaneously on the same physical hardware.

This means a single powerful physical server can be divided into multiple isolated virtual environments, each acting as its own independent computer. This maximises hardware utilisation, reduces costs, and improves flexibility.

**Why it matters:**
- Reduces the need for physical hardware
- Enables faster provisioning of computing resources
- Forms the foundation of cloud computing

---

## 2. What is a Virtual Machine (VM)?

A **Virtual Machine (VM)** is a software-based computer that runs on top of physical hardware. It behaves exactly like a real computer — it has its own CPU, memory, storage, and operating system — but it exists entirely in software.

Each VM is fully isolated from other VMs running on the same physical machine. If one VM crashes or is compromised, the others are not affected.

**Key characteristics of a VM:**
- Runs its own operating system (Windows, Linux, etc.)
- Isolated from other VMs on the same host
- Can be started, stopped, cloned, or deleted like a file
- Portable — can be moved between physical machines

---

## 3. Types of Hypervisors

A **Hypervisor** (also called a Virtual Machine Monitor) is the software layer that sits between the physical hardware and the virtual machines. It manages and allocates hardware resources to each VM.

There are two types of hypervisors:

### Type 1 – Bare Metal Hypervisor
Runs **directly on the physical hardware** — there is no host operating system underneath it.

- Faster and more efficient
- Used in enterprise and cloud environments
- **Examples:** VMware ESXi, Microsoft Hyper-V, AWS Nitro System, KVM

### Type 2 – Hosted Hypervisor
Runs **on top of a host operating system** (like Windows or macOS), and the VMs run on top of that.

- Easier to set up, suitable for personal or development use
- Slightly slower due to the extra OS layer
- **Examples:** Oracle VirtualBox, VMware Workstation, Parallels

> **AWS EC2 uses a Type 1 hypervisor** — the AWS Nitro System — for high performance and security.

---

## 4. What is Amazon EC2?

**Amazon Elastic Compute Cloud (EC2)** is a web service provided by AWS that allows you to rent virtual computers (virtual machines) in the cloud. Instead of buying and maintaining physical servers, you can launch a virtual server on AWS within minutes and pay only for what you use.

EC2 is the backbone of AWS compute services and is used to host applications, run workloads, process data, and much more.

**Key features of EC2:**
- **Elastic** — scale up or down based on demand
- **Flexible** — choose CPU, memory, storage, and OS
- **Pay-as-you-go** — billed per second or per hour
- **Global** — deploy in any AWS region worldwide
- **Secure** — integrated with IAM, VPC, and Security Groups

---

## 5. What is an EC2 Instance?

An **EC2 Instance** is a single virtual server running in the AWS cloud. When you launch an EC2 instance, AWS allocates virtual CPU, memory, storage, and networking resources to it based on the **instance type** you choose.

Think of an instance as a single VM that you can SSH into, install software on, and run your applications — just like a physical server, but in the cloud.

### Instance Types
EC2 offers different instance types optimised for different use cases:

| Family | Optimised For | Example |
|--------|--------------|---------|
| General Purpose | Balanced CPU & memory | t3.micro, m5.large |
| Compute Optimised | High CPU workloads | c5.large |
| Memory Optimised | Large in-memory datasets | r5.large |
| Storage Optimised | High disk I/O | i3.large |
| GPU Instances | ML / graphics workloads | p3.2xlarge |

### Instance States
- **Running** — actively running and billed
- **Stopped** — shut down, not billed for compute (storage still billed)
- **Terminated** — permanently deleted

---

## 6. What is an AMI?

**AMI (Amazon Machine Image)** is a pre-configured template used to launch an EC2 instance. It contains everything needed to boot a virtual machine — the operating system, application software, configurations, and data volumes.

When you launch an EC2 instance, you must choose an AMI. The AMI defines **what your instance looks like when it starts**.

### What an AMI includes:
- **Root volume snapshot** — OS and pre-installed software
- **Launch permissions** — who can use the AMI
- **Block device mapping** — storage volumes to attach on launch

### Types of AMIs:
- **AWS-provided AMIs** — Amazon Linux, Ubuntu, Windows Server (maintained by AWS)
- **AWS Marketplace AMIs** — pre-built images from third-party vendors (e.g., NGINX, WordPress)
- **Custom AMIs** — AMIs you create from your own configured EC2 instances

### Why AMIs are useful:
- Launch identical instances quickly (great for Auto Scaling)
- Save a configured environment as a reusable template
- Share images across AWS accounts or regions

---

## Summary

| Concept | Description |
|--------|-------------|
| Virtualisation | Running multiple virtual computers on one physical machine |
| Virtual Machine | A software-based computer with its own OS and resources |
| Type 1 Hypervisor | Runs directly on hardware (e.g., AWS Nitro, KVM) |
| Type 2 Hypervisor | Runs on top of a host OS (e.g., VirtualBox) |
| EC2 | AWS service to rent virtual servers in the cloud |
| EC2 Instance | A single virtual server launched on EC2 |
| AMI | A template used to launch an EC2 instance |

---

*Happy Learning! ☁️*
