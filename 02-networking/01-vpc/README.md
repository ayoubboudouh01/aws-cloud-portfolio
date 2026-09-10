# AWS VPC Networking Lab

## Overview

This lab demonstrates the deployment and testing of a basic secure AWS network architecture using Amazon VPC, public/private subnets, an Internet Gateway, NAT Gateway, Security Groups, a Bastion host, and private EC2 instances.

The objective was to understand how public and private resources communicate with the internet and how access to private instances can be restricted.

---

## Architecture

```text
                         INTERNET
                            │
                            ▼
                    ┌───────────────┐
                    │ Internet      │
                    │ Gateway       │
                    └───────┬───────┘
                            │
                 ┌──────────┴──────────┐
                 │                     │
          Public Subnet A        Public Subnet B
          10.0.1.0/24            10.0.2.0/24
                 │
                 │ SSH
                 ▼
          ┌───────────────┐
          │ Bastion EC2   │
          │ 10.0.1.170    │
          └───────┬───────┘
                  │
             Private SSH
                  │
                  ▼
          ┌───────────────┐
          │ App-A EC2     │
          │ 10.0.11.151   │
          │ Private only  │
          └───────┬───────┘
                  │
                  │ Default route
                  ▼
             NAT Gateway
                  │
                  ▼
               Internet
```

---

## Network Design

| Component        | Configuration                                       |
| ---------------- | --------------------------------------------------- |
| VPC              | `10.0.0.0/16`                                       |
| Public Subnet A  | `10.0.1.0/24`                                       |
| Public Subnet B  | `10.0.2.0/24`                                       |
| Private Subnet A | `10.0.11.0/24`                                      |
| Private Subnet B | `10.0.12.0/24`                                      |
| Bastion          | Public Subnet A                                     |
| App-A            | Private Subnet A                                    |
| Internet Gateway | Attached to VPC                                     |
| NAT Gateway      | Provides outbound internet access to private subnet |

---

## Security Groups

### SG-Bastion

The Bastion is intended to be the controlled entry point into the private network.

SSH access is restricted to my public IP using TCP port 22 and `/32`.

SSH should **not** be exposed using:

```text
0.0.0.0/0
```

### SG-App

App-A uses a separate security group.

Inbound SSH:

```text
Protocol: TCP
Port: 22
Source: SG-Bastion
```

This means App-A accepts SSH connections from instances associated with the Bastion security group rather than from arbitrary internet addresses.

HTTP was initially present as:

```text
TCP 80 → 0.0.0.0/0
```

It was removed because HTTP access was not required for the current private-server lab.

---

## Bastion Connectivity Test

The Bastion EC2 was successfully accessed from my local Windows machine using SSH.

The Bastion is located in a public subnet and has internet connectivity through the Internet Gateway.

SSH path:

```text
Local PC
   ↓ SSH
Bastion EC2
```

---

## Private EC2 Connectivity Test

App-A was launched in:

```text
Private Subnet A
10.0.11.0/24
```

App-A private IP:

```text
10.0.11.151
```

App-A does not have a public IPv4 address.

I successfully connected from the Bastion to App-A:

```bash
ssh -i ~/Bastion\ EC2.pem ec2-user@10.0.11.151
```

This demonstrated private subnet connectivity.

SSH path:

```text
Local PC
   ↓
Bastion
   ↓ private IP
App-A
```

---

## NAT Gateway Test

From App-A, I tested outbound internet connectivity:

```bash
curl -I https://aws.amazon.com
```

Result:

```text
HTTP/2 200
```

This confirms that the private EC2 instance can reach the internet through the configured NAT path.

I also checked the public IP visible to external services:

```bash
curl -s https://checkip.amazonaws.com
```

Result:

```text
3.217.109.162
```

The private EC2 therefore reaches the internet using the NAT Gateway's public-facing address rather than having a public IPv4 address of its own.

---

## App-A Routing

The route table inside App-A showed:

```text
default via 10.0.11.1 dev ens5
10.0.0.2 via 10.0.11.1 dev ens5
10.0.11.0/24 dev ens5
10.0.11.1 dev ens5
```

The important route is:

```text
default via 10.0.11.1
```

This sends traffic outside the local subnet toward the subnet's routing infrastructure.

---

## Troubleshooting

### SSH private-key permissions

When connecting from the Bastion to App-A, SSH initially failed with:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0664 for '/home/ec2-user/Bastion EC2.pem' are too open.
```

SSH ignored the private key because it was accessible to other users.

I corrected the permissions with:

```bash
chmod 400 ~/Bastion\ EC2.pem
```

After changing the permissions, the SSH connection to App-A succeeded.

This demonstrated an important Linux/SSH security requirement: private keys must have restrictive filesystem permissions.

---

## Validation Results

| Test                              | Result                 |
| --------------------------------- | ---------------------- |
| PC → Bastion SSH                  |  Passed               |
| Bastion → App-A SSH               |  Passed               |
| App-A has public IPv4             |  No public IP         |
| App-A → Internet                  |  Passed               |
| App-A → AWS website               |  HTTP 200             |
| NAT public IP detected            | `3.217.109.162`        |
| App-A SSH from arbitrary internet | Intended to be blocked |
| App-A SSH source                  | `SG-Bastion`           |

---

## Lessons Learned

### Security Groups vs Route Tables

A Security Group controls whether traffic is allowed to reach an instance.

A route table determines where network traffic is sent.

Changing a Security Group does not attach that Security Group to an EC2 instance.

### Public vs Private Subnets

A private EC2 instance does not need a public IPv4 address to access the internet.

The NAT Gateway provides outbound internet connectivity while preventing unsolicited inbound connections from the internet.

### Bastion Host

The Bastion provides a controlled SSH entry point to private instances.

The resulting access model is:

```text
Internet
   ↓
Bastion
   ↓
Private EC2
```

rather than exposing the private EC2 directly to the internet.

---

## Next Steps

* Launch App-B in Private Subnet B
* Configure App-B with the same security model
* Test Bastion → App-B
* Test App-A ↔ App-B private connectivity where appropriate
* Verify private instances cannot be directly reached from the internet
* Configure VPC Flow Logs
* Analyze traffic logs
* Improve the architecture using more restrictive security rules
* Reproduce the infrastructure with Terraform

