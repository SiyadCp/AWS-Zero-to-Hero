# AWS EC2 Tenancy Types

Tenancy in Amazon EC2 defines **how EC2 instances are placed on physical hardware**. It determines whether instances share hardware with other AWS customers or run on dedicated infrastructure.

AWS provides different tenancy options to meet **security, compliance, and licensing requirements**.

---

# 1. Shared Tenancy (Default)

Shared tenancy is the **default option** when launching EC2 instances.

In this model, multiple AWS customers share the same physical hardware, but instances remain **logically isolated** using virtualization.

## Characteristics

* Multiple customers share the same host machine
* Fully managed by AWS
* Most cost-effective option
* Default tenancy for EC2

## Advantages

* Lowest cost
* Easy to launch and manage
* Suitable for most workloads

## Use Cases

* Web applications
* Development and testing environments
* Microservices
* Most cloud-native applications

---

# 2. Dedicated Instances

Dedicated Instances run on **hardware dedicated to a single AWS account**.

Even though multiple instances from the same account may share the same host, **no other AWS customers share that physical server**.

## Characteristics

* Hardware dedicated to a single AWS account
* Instances may share host with other instances in the same account
* Higher cost than shared tenancy

## Advantages

* Increased isolation
* Helps meet compliance requirements

## Use Cases

* Regulatory compliance workloads
* Security-sensitive applications
* Organizations requiring dedicated infrastructure

---

# 3. Dedicated Hosts

Dedicated Hosts provide **an entire physical server dedicated to your account**.

This option gives you visibility and control over the **physical sockets, cores, and host placement**.

## Characteristics

* Full physical server allocated to one customer
* Provides control over instance placement
* Supports license-based software

## Advantages

* Ideal for BYOL (Bring Your Own License)
* Compliance requirements
* Visibility of hardware resources

## Use Cases

* Enterprise applications
* Software with licensing restrictions
* Compliance-heavy environments

---

# Summary

| Tenancy Type        | Hardware Sharing            | Control Level | Cost    | Typical Use Case              |
| ------------------- | --------------------------- | ------------- | ------- | ----------------------------- |
| Shared (Default)    | Shared with other customers | Low           | Lowest  | General workloads             |
| Dedicated Instances | Dedicated to one account    | Medium        | Higher  | Compliance workloads          |
| Dedicated Hosts     | Fully dedicated server      | High          | Highest | License-based enterprise apps |

---

# Key Points

* **Shared tenancy** is the most commonly used and cost-effective option.
* **Dedicated Instances** provide additional hardware isolation.
* **Dedicated Hosts** offer the highest level of control and are useful for licensing requirements.

---

# Conclusion

EC2 tenancy options allow organizations to choose the right level of **isolation, compliance, and cost efficiency** based on their workload requirements.
