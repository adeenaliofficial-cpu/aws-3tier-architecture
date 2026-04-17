# AWS 3‑Tier Architecture Lab – Project Documentation

This repository contains my first AWS infrastructure project: a secure 3‑tier environment built from scratch.  It records every step I took, why I made certain design decisions, and includes placeholders for the screenshots I captured along the way.  By documenting the rationale behind the architecture and the lessons learned, I hope to show that I’m not just following instructions—I understand the *why* behind each component and have a security‑first mindset.

## Project Summary

The goal of this lab was to design and deploy a **3‑tier architecture** on AWS.  The environment consists of:

- A **custom VPC** with clearly defined public and private subnets.
- Separate **security groups** for the bastion host, web server, application server, and database.
- A **bastion host** in the public subnet for controlled administrative access.
- A **web server** exposed to the internet via the public subnet.
- An **application server** hosted privately with no public IP.
- A private **RDS MariaDB** instance in its own subnet group.

Throughout the build I focused on **network segmentation**, **least privilege**, and **defence in depth** so that each tier is isolated and only receives the minimum necessary traffic.  The remainder of this document explains how to reproduce this environment and why each decision was made.

## Architecture Diagram

The high‑level design is illustrated below.  It shows the VPC, subnets, gateways, route tables, EC2 instances, and RDS instance.  A bastion host provides SSH access to the private tier, while a NAT Gateway enables outbound updates from private instances.

![Architecture Diagram](images/architecture-diagram.png)

## Build Steps & Rationale

### 1. Networking Setup

1. **Create a VPC** – I used the CIDR block `192.168.0.0/16` to allow enough address space for multiple subnets.  A custom VPC gives full control over IP addressing and routing.
2. **Create subnets** – Four subnets were created:
   - **Public subnet:** `192.168.1.0/24` for the bastion and web server.
   - **Private subnet 1:** `192.168.2.0/24` for the application server.
   - **Private subnet 2:** `192.168.3.0/24` reserved for future workloads or redundancy.
   - **Private subnet 3:** `192.168.4.0/24` dedicated to the database tier.

   Keeping the app and database tiers in private subnets prevents direct internet exposure and enforces network segmentation.
3. **Allocate an Elastic IP** – Required for the NAT Gateway so private instances can reach the internet for updates without being reachable from outside.
4. **Create an Internet Gateway** – Enables public subnet resources to send/receive traffic from the internet.
5. **Create a NAT Gateway** – Placed in the public subnet and associated with the Elastic IP.  This allows private subnet instances to initiate outbound traffic while blocking inbound connections.
6. **Route tables** – Two route tables were configured:
   - **Public route table** with a default route (`0.0.0.0/0`) pointing to the Internet Gateway and associated with the public subnet.
   - **Private route table** with a default route pointing to the NAT Gateway and associated with all private subnets.  This enforces separation even if security group rules are misconfigured.
7. **Security groups** – Four groups were created:
   - **Bastion SG:** Allows SSH from my IP and optional HTTP/HTTPS from anywhere.
   - **Web SG:** Allows HTTP/HTTPS from anywhere; restricted inbound access otherwise.
   - **App SG:** Allows SSH from the bastion SG, ICMP from the web SG, and later HTTP/HTTPS and MySQL from trusted sources.
   - **DB SG:** Allows MySQL/Aurora from the app SG and bastion SG only.  This prevents the web tier from hitting the database directly.

   Separating security groups enforces least privilege—if one tier is compromised it cannot directly reach the others.

### 2. Launch Compute Instances

1. **Bastion Host** – An Amazon Linux 2 instance in the public subnet with a public IP and the Bastion SG attached.  The bastion host is the single entry point for administrators and uses SSH keys for secure access.
2. **Web Server** – Another Amazon Linux 2 instance in the public subnet.  The user data script installs Apache, PHP, and the MariaDB client.  It runs without a graphical interface and starts the web service automatically on boot.
3. **Application Server** – Launched in **Private Subnet 1** with no public IP.  A user data script installs and starts the MariaDB server (to practise local DB setup).  Access is only via the bastion host or the web server through the security groups.

### 3. Provision the Database

1. **DB Subnet Group** – Created in Amazon RDS to span the two private subnets reserved for the database.  This allows for multi‑AZ failover in the future.
2. **RDS Instance** – A MariaDB instance deployed in private subnets with **public access disabled** and the **DB SG attached**.  I chose the free tier for cost reasons during the lab.  In production, you would enable encryption and automated backups.

### 4. Validate Connectivity

1. SSH into the bastion host using the provided key pair.
2. Transfer the private key to the bastion and set its permissions (`chmod 400`).
3. SSH from the bastion into the application server using its private IP.  This confirms that routing and the security groups allow internal access.
4. Ping the web server from the app server to test east–west connectivity.
5. Connect to the RDS endpoint from the application server using the MySQL client and run `SHOW DATABASES;` to verify access.

These tests verify that each tier can communicate only with its intended peers and that the database is not exposed directly to the internet.

## Screenshots

As you perform the lab, capture evidence of each major step and save the images in the `images/` folder with the following filenames.  Once the screenshots are in place, they will render automatically in the README.

| Step | Filename | Description |
|-----|---------|-------------|
| **VPC / Subnets** | `images/01-vpc-subnets.png` | Shows the VPC and subnets you created in the AWS console. |
| **Route Tables / IGW / NAT Gateway** | `images/02-routing.png` | Captures your route tables and gateways after configuration. |
| **Security Groups** | `images/03-security-groups.png` | Lists inbound rules for each security group. |
| **EC2 Instances** | `images/04-ec2-instances.png` | Displays the bastion, web, and app instances running. |
| **DB Subnet Group / RDS** | `images/05-rds.png` | Shows the DB subnet group and the RDS instance settings. |
| **Bastion SSH / App Server Access** | `images/06-ssh-test.png` | Proof that you can SSH from the bastion to the app server. |
| **Database Connectivity Test** | `images/07-db-test.png` | Proof that you can connect to the database from the app tier. |

Add any additional screenshots that you find useful (e.g., NAT Gateway creation, subnet association details).  Place them in the `images/` folder and update the table above if you change the file names.

## Key Decisions & Trade‑Offs

- **Separate security groups per tier** rather than a single group, to enforce least privilege and reduce blast radius.
- **Private subnets for the app and database tiers** to prevent direct internet exposure.
- **NAT Gateway** instead of direct internet connectivity for private instances.  This increases cost slightly but improves security.
- **Bastion host** for administrative access instead of opening SSH on every instance.  A VPN could further improve security in a real‑world deployment.

These choices align with AWS best practices for defence in depth and network segmentation.

## What I Learned

This project helped me understand how public and private networking work together inside a VPC, how Internet and NAT Gateways serve different purposes, how security groups control east–west and north–south traffic, and how to use a bastion host for controlled access.  I also practised launching RDS in private subnets and validated connectivity across tiers.

## Future Improvements & Roadmap

To build on this lab, I plan to:

1. **Rebuild the environment with Terraform or CloudFormation** to practice Infrastructure as Code.
2. **Add monitoring and alarms** with CloudWatch and SNS to detect anomalies.
3. **Implement stronger secret management**, using AWS Secrets Manager instead of hard‑coding DB credentials in user data.
4. **Extend the architecture** with a load balancer and auto‑scaling group to make the web tier highly available.


## About Me

I am an **AWS re/Start graduate** and a final‑year **Cybersecurity student** at FAST‑NUCES, Karachi.  I hold the **AWS Certified Cloud Practitioner** and **(ISC)² Certified in Cybersecurity (CC)** credentials.  Through internships at the **State Bank of Pakistan** and **MRBF Consulting**, I have gained experience in network segmentation, risk management, and compliance.  My goal is to become a cloud security engineer who builds compliant, resilient infrastructure with a security‑first mindset.  You can learn more about my experience and projects on [my LinkedIn](https://www.linkedin.com/in/adeen-ali/).

---

**Note:** If you’re cloning this repo to follow along, rename the screenshot files to match the table above or update the Markdown paths accordingly.  All instructions are written using the Asia/Karachi timezone (UTC+05:00) and date format as of 4 April 2026.