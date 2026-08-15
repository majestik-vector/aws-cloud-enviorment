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

* Cloud provider: AWS
* VPC: `cloudlab-vpc`
* VPC CIDR: `10.0.0.0/16`
* Public subnet: `cloudlab-public-subnet`
* Public subnet CIDR: `10.0.1.0/24`
* Private subnet: `cloudlab-private-subnet`
* Private subnet CIDR: `10.0.2.0/24`
* Internet Gateway: `cloudlab-igw`
* NAT Gateway: `cloudlab-nat`
* Public route table
* Private route table
* Security group: `cloudlab-web-sg`

### Compute

* EC2 instance: `cloudlab-web-server`
* Operating system: Amazon Linux 2023
* Instance type: `t3.micro`

### Security Tools

* Kali Linux
* Nmap
* SSH
* Netcat
* Linux command-line security tools
* AWS security controls and logging

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
```

---

# AWS Access and Kali Security Workflow

## VirtualBox Shared Folder

A VirtualBox shared folder was configured between the Windows host and Kali Linux.

Shared folder:

```text
Kali-Share
```

The folder was configured with:

* Auto-mount: enabled
* Make Permanent: enabled

Kali mounted the shared folder as:

```text
/media/sf_Kali_Share/
```

The AWS private key was stored in the shared folder temporarily so it could be transferred into Kali.

The key file was:

```text
cloudlab-key.pem
```

---

## Private Key Handling

The private key was copied from the VirtualBox shared folder into the Kali user's home directory.

Command used:

```bash
cp /media/sf_Kali_Share/cloudlab-key.pem ~/
```

The key was then checked:

```bash
ls -l ~/cloudlab-key.pem
```

Initially, the file inherited broader shared-folder permissions.

The permissions were restricted using:

```bash
chmod 400 ~/cloudlab-key.pem
```

Final permissions:

```text
-r--------
```

This restricts the private key so only the owner can read it.

### Security Principle

Private SSH keys should never be committed to source control.

This repository's `.gitignore` explicitly excludes private keys and certificates:

```text
*.pem
*.key
*.p12
*.pfx
```

The actual contents of `cloudlab-key.pem` are never stored in this repository.

---

# AWS EC2 SSH Access

The EC2 instance is running Amazon Linux 2023.

The instance was accessed from Kali using the AWS private key and the default Amazon Linux SSH account:

```text
ec2-user
```

SSH connection format:

```bash
ssh -i ~/cloudlab-key.pem ec2-user@<EC2_PUBLIC_IP>
```

The first connection required host-key verification.

After authentication, Kali successfully established an interactive SSH session on the EC2 instance.

Successful shell:

```text
[ec2-user@ip-10-0-1-65 ~]$
```

---

# Host Verification

After establishing the SSH session, the identity of the remote host was verified.

## User Verification

Command:

```bash
whoami
```

Result:

```text
ec2-user
```

## Hostname Verification

Command:

```bash
hostname
```

Observed hostname:

```text
ip-10-0-1-65.us-east-2.compute.internal
```

## Network Verification

Command:

```bash
ip addr
```

The EC2 host was confirmed to have:

* Interface: `ens5`
* IPv4: `10.0.1.65/24`
* Subnet: `10.0.1.0/24`

This confirmed that the SSH session was operating directly on the AWS EC2 instance rather than on the Kali workstation.

---

# SSH Authentication Security

The SSH server was configured to use public-key authentication rather than password-based authentication.

The effective SSH configuration was checked directly on the EC2 host.

Command:

```bash
sudo sshd -T | grep -Ei '^(passwordauthentication|pubkeyauthentication|permitrootlogin|maxauthtries|allowusers|allowgroups)'
```

Observed configuration:

```text
maxauthtries 6
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication no
```

## Security Findings

### Public-Key Authentication

```text
pubkeyauthentication yes
```

SSH key authentication is enabled.

### Password Authentication

```text
passwordauthentication no
```

Password-based SSH authentication is disabled.

This removes the normal SSH password-login path and significantly reduces exposure to password-based brute-force attacks.

### Authentication Attempts

```text
maxauthtries 6
```

The SSH daemon limits authentication attempts per connection.

### Root Login

```text
permitrootlogin without-password
```

Root password authentication is disabled, while non-password authentication methods remain governed by the SSH configuration.

---

# AWS Security Group

The EC2 instance is protected by the AWS Security Group:

```text
cloudlab-web-sg
```

The SSH inbound rule was intentionally restricted to the authorized public IP address of the Kali workstation using a `/32` CIDR.

Configuration:

```text
Type: SSH
Protocol: TCP
Port: 22
Source: <AUTHORIZED_KALI_PUBLIC_IP>/32
```

A `/32` represents a single IPv4 address.

This means the Security Group is designed to permit SSH access from only the authorized Kali public IP rather than from the entire internet.

The Security Group therefore provides an additional security layer before traffic reaches the EC2 operating system.

---

# Security Group Access Validation

The Kali public IP was verified using:

```bash
curl -4 ifconfig.me
```

The returned address matched the `/32` source configured in the Security Group.

Network connectivity to SSH was then tested from Kali:

```bash
nc -vz <EC2_PUBLIC_IP> 22
```

Successful result:

```text
22 (ssh) open
```

This confirmed that the authorized Kali source could reach the EC2 SSH service.

---

# Controlled Security Group Blocking Test

A controlled defensive test was performed to verify that the Security Group was actually enforcing the intended restriction.

The SSH inbound source was temporarily changed from the authorized Kali address to an unrelated `/32` address.

The SSH connection was then tested from Kali:

```bash
nc -vz -w 5 <EC2_PUBLIC_IP> 22
```

Observed result:

```text
Connection timed out
```

The timeout demonstrated that the Security Group blocked the connection before the SSH service could be reached.

The Security Group rule was then restored to the authorized Kali `/32` source.

The same connectivity test was repeated:

```bash
nc -vz -w 5 <EC2_PUBLIC_IP> 22
```

Observed result:

```text
22 (ssh) open
```

This demonstrated both states of the control:

```text
AUTHORIZED SOURCE
        |
        v
AWS Security Group
        |
        | TCP/22 ALLOWED
        v
      SSH


UNAUTHORIZED SOURCE
        |
        v
AWS Security Group
        |
        | TCP/22 BLOCKED
        X
      SSH
```

---

# Security Control Validation

The Security Group test provided direct evidence that the AWS network control was functioning as intended.

Rather than assuming the inbound rule was effective, the rule was intentionally changed and tested.

### Authorized state

```text
Kali
  |
  v
AWS Security Group
  |
  | TCP/22 ALLOWED
  v
EC2 SSH Service
```

### Unauthorized state

```text
External Source
  |
  v
AWS Security Group
  |
  | TCP/22 BLOCKED
  X
EC2 SSH Service
```

This demonstrates layered security:

1. AWS controls the network path.
2. SSH controls authentication.
3. The private key controls identity.

An attacker would need to overcome multiple security layers rather than a single control.

---

# Host-Level Security Validation

The EC2 host was also inspected from inside the operating system using Linux security and networking tools.

Commands used included:

```bash
ss -tuln
```

and:

```bash
sudo ss -tulpn
```

This provided visibility into services and sockets active on the host.

The exercise reinforced an important cloud-security principle:

> A service existing on a server does not automatically mean the service is reachable from the public internet.

External reachability depends on multiple layers, including:

* AWS Security Groups
* Routing
* Service bind addresses
* Host configuration
* Network filtering
* Authentication controls

The lab therefore compares both the **internal host perspective** and the **external attacker perspective**.

---

# SSH Security Assessment

The current SSH security posture includes several defensive controls.

### Authentication

* Public-key authentication enabled
* Password authentication disabled
* Authentication attempts limited

### Network Access

* SSH restricted through the AWS Security Group
* SSH access limited to an authorized `/32` source
* Unauthorized source testing produced a timeout

### Key Security

* Private SSH key stored locally
* Private key permissions restricted using `chmod 400`
* Private keys excluded from Git
* No private key contents stored in the repository

These controls work together to reduce the likelihood of unauthorized SSH access.

---

# Attack vs. Defense Perspective

## Attacker Perspective

The primary security concern for the SSH access path is not simply guessing a password because password authentication has been disabled.

A realistic attacker would instead be interested in questions such as:

* Can the attacker reach the SSH service?
* Is the Security Group too permissive?
* Can a valid private key be obtained?
* Can a stolen key be used from an allowed network source?
* Are administrative authentication methods overly exposed?
* Are there weaknesses in the host configuration?

The lab demonstrates how network controls and authentication controls change the attack path before exploitation even begins.

## Defender Perspective

The current controls reduce exposure through multiple layers:

```text
Internet
   |
   v
AWS Security Group
   |
   | Restrict source to authorized /32
   v
EC2 SSH Service
   |
   | Public-key authentication
   | Password authentication disabled
   | Authentication attempts limited
   v
Authorized User
```

This is an example of **defense in depth**.

No single security control is relied upon to provide the entire security boundary.

---

# Lessons Demonstrated

This phase of the lab demonstrates several practical cloud-security concepts:

1. A public IP address does not automatically make every host service publicly reachable.
2. AWS Security Groups provide network-layer access control before traffic reaches the host.
3. `/32` CIDR notation can restrict access to a single IPv4 address.
4. SSH key authentication removes dependence on password authentication.
5. Disabling password authentication reduces exposure to password-based SSH attacks.
6. Private key permissions are an important local security control.
7. Security controls should be validated through testing rather than assumed to be effective.
8. Network controls and host controls work together as layers of defense.
9. Cloud security requires protecting both infrastructure and credentials.
10. A strong security design combines prevention, authentication, validation, and monitoring.

---

# Current Project Status

The AWS Cloud Security Lab currently demonstrates:

**Build → Reconnaissance → Access → Security Testing → Defense Validation**

Detailed reconnaissance results are maintained separately in the repository's reconnaissance documentation.

Future phases will expand the environment into additional controlled security scenarios involving:

* Detection
* Logging
* Monitoring
* Network segmentation
* Vulnerability assessment
* Attack simulation
* Hardening
* Incident response
* Security validation

All testing is performed against infrastructure created and controlled for this project.

---

# Repository Security

Sensitive material is intentionally excluded from this repository.

The `.gitignore` file excludes:

```text
*.pem
*.key
*.p12
*.pfx
.env
.env.*
credentials/
keys/
.ssh/
known_hosts
authorized_keys
```

No AWS private keys, passwords, API secrets, access keys, or other credentials should ever be committed to this repository.
