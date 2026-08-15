# AWS Cloud Security Lab

A hands-on cloud security laboratory built in Amazon Web Services (AWS) to demonstrate practical skills in cloud networking, Linux administration, reconnaissance, penetration testing, security hardening, detection, and incident response.

This project is being built from the ground up and documented throughout the process.

---

## Project Objective

The goal of this project is to design, deploy, test, attack, defend, and document a segmented AWS environment.

Rather than following a purely theoretical approach, this environment is used to demonstrate the complete security lifecycle:

**Build → Reconnaissance → Attack → Detection → Defense → Validation**

The project demonstrates both attacker and defender perspectives through controlled testing against infrastructure owned and operated for this project.

---

## Environment

### Cloud Infrastructure

- Cloud provider: AWS
- VPC: `cloudlab-vpc`
- VPC CIDR: `10.0.0.0/16`
- Public subnet: `cloudlab-public-subnet`
- Public subnet CIDR: `10.0.1.0/24`
- Private subnet: `cloudlab-private-subnet`
- Private subnet CIDR: `10.0.2.0/24`
- Internet Gateway: `cloudlab-igw`
- NAT Gateway: `cloudlab-nat`
- Public route table
- Private route table
- Security group: `cloudlab-web-sg`

### Compute

- EC2 instance: `cloudlab-web-server`
- Operating system: Amazon Linux 2023
- Instance type: `t3.micro`

### Security Tools

- Kali Linux
- Nmap
- SSH
- Linux command-line security tools
- AWS security controls and logging

---

## Network Architecture

The environment uses separate public and private subnets.

```text
                         INTERNET
                            |
                            |
                    Internet Gateway
                       cloudlab-igw
                            |
                    +-------+-------+
                    |               |
                    |  cloudlab-vpc |
                    | 10.0.0.0/16   |
                    |               |
             PUBLIC SUBNET     PRIVATE SUBNET
              10.0.1.0/24       10.0.2.0/24
                    |               |
                    |               |
               EC2 / NAT        Private
                Gateway         Resources
                    |
                    |
              Outbound Internet
