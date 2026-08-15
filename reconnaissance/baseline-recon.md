# Baseline Reconnaissance

## Objective

Establish the external attack surface of the AWS cloud environment before making any intentional security changes.

## Environment

- Cloud provider: AWS
- VPC: cloudlab-vpc
- VPC CIDR: 10.0.0.0/16
- Public subnet: cloudlab-public-subnet (10.0.1.0/24)
- Private subnet: cloudlab-private-subnet (10.0.2.0/24)
- Testing system: Kali Linux
- Target: cloudlab-web-server
- Operating system: Amazon Linux 2023
- Testing perspective: External attacker

---

## 1. ICMP Connectivity Test

### Command

    ping -c 4 <EC2_PUBLIC_IP>

### Result

    4 packets transmitted, 0 received, 100% packet loss

### Observation

The EC2 instance did not respond to ICMP echo requests.

This does not prove that the host is offline. ICMP may be blocked by the AWS security group.

---

## 2. TCP Scan — Top 1,000 Ports

### Command

    nmap -Pn <EC2_PUBLIC_IP>

### Result

    Host is up.
    All 1000 scanned ports are in ignored states.
    Not shown: 1000 filtered tcp ports (no-response)

### Observation

No commonly used TCP services were externally reachable through the current AWS security-group configuration.

---

## 3. TCP Scan — All 65,535 Ports

### Command

    nmap -Pn -p- --min-rate 1000 <EC2_PUBLIC_IP>

### Result

    Host is up.
    All 65535 scanned ports are in ignored states.
    Not shown: 65535 filtered tcp ports (no-response)

### Observation

The complete TCP port range produced no externally reachable services.

---

## 4. UDP Scan — Top 100 Ports

### Command

    sudo nmap -Pn -sU --top-ports 100 <EC2_PUBLIC_IP>

### Result

    All 100 scanned ports are in ignored states.
    Not shown: 100 open|filtered udp ports

### Observation

Nmap did not identify any confirmed open UDP services.

The `open|filtered` result means Nmap could not determine whether the ports were open or filtered because UDP does not provide the same connection-oriented response behavior as TCP.

---

# Baseline Security Assessment

The initial external reconnaissance identified an extremely limited attack surface.

No TCP services were externally reachable, ICMP was non-responsive, and no UDP services were confirmed as open.

The AWS security-group configuration is currently preventing external access to TCP services on the EC2 instance.

This baseline will be used for comparison during subsequent controlled attack and defense exercises.

---

# Security Significance

The baseline demonstrates the importance of reducing the external attack surface before exposing services.

A host can be online while presenting no externally reachable TCP services.

The failed ICMP test also demonstrates that lack of ping response does not prove that a host is offline. Nmap was able to determine that the host was up even though ICMP did not respond.

---

# Next Phase

The next phase will introduce controlled vulnerabilities into the lab environment.

The workflow will be:

**Baseline → Vulnerability → Attack → Evidence → Remediation → Validation**
