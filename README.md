# AWS NAT Instance Configuration

## Project Overview

This project demonstrates how to configure an Amazon EC2 instance as a Network Address Translation (NAT) instance to provide outbound internet connectivity to resources deployed in a private subnet.

The NAT instance uses Linux IP forwarding and `iptables` masquerading to forward traffic from the private subnet to the internet through a public subnet.

## Architecture

```text
                 Internet
                    |
              Internet Gateway
                    |
              Public Subnet
                    |
             NAT EC2 Instance
              (Amazon Linux)
                    |
          IP Forwarding + iptables
                    |
              Private Subnet
                    |
             Private EC2 Instance
                    |
             Outbound Internet
```

## Technologies Used

- Amazon Web Services (AWS)
- Amazon EC2
- Amazon VPC
- Public and private subnets
- Internet Gateway
- Route Tables
- Security Groups
- Amazon Linux 2023
- Linux IP Forwarding
- iptables
- AWS CLI

## Implementation Steps

### 1. Create the VPC and Subnets

- Create a VPC with public and private subnets.
- Attach an Internet Gateway to the VPC.
- Configure the public subnet route table to route internet-bound traffic through the Internet Gateway.
- Launch a NAT instance in the public subnet.

### 2. Configure the NAT Instance

Enable IPv4 forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

To make the setting persistent, configure the following in `/etc/sysctl.conf` or an appropriate file under `/etc/sysctl.d/`:

```text
net.ipv4.ip_forward = 1
```

Apply the persistent configuration:

```bash
sudo sysctl --system
```

### 3. Configure IP Masquerading

Identify the network interface:

```bash
ip -br addr
```

For an instance whose outbound interface is `ens5`, configure NAT masquerading:

```bash
sudo iptables -t nat -A POSTROUTING \
  -o ens5 -j MASQUERADE
```

Allow forwarding from the private subnet, replacing the example CIDR with the actual private subnet CIDR:

```bash
sudo iptables -A FORWARD \
  -s <PRIVATE_SUBNET_CIDR> \
  -o ens5 -j ACCEPT
```

Allow return traffic for established connections:

```bash
sudo iptables -A FORWARD \
  -d <PRIVATE_SUBNET_CIDR> \
  -i ens5 \
  -m conntrack \
  --ctstate ESTABLISHED,RELATED \
  -j ACCEPT
```

**Note:** These commands illustrate the configuration. Review your existing firewall rules and interface names before applying them. Configure rule persistence separately if the NAT instance must retain these rules after reboot.

### 4. Configure AWS Networking

- Disable the source/destination check on the NAT instance.
- Configure the private subnet route table with a default route (`0.0.0.0/0`) targeting the NAT instance.
- Verify that the NAT instance's security group permits the required traffic from the private subnet and outbound traffic.
- Verify that network ACLs and operating-system firewall rules permit the necessary traffic.

### 5. Validate Connectivity

From the private EC2 instance, test connectivity:

```bash
ip route
```

Check DNS resolution:

```bash
getent hosts example.com
```

Test outbound HTTPS connectivity:

```bash
curl -I --connect-timeout 10 https://example.com
```

Inspect NAT rules on the NAT instance:

```bash
sudo iptables -t nat -L -n -v
```

Inspect forwarding rules:

```bash
sudo iptables -L FORWARD -n -v
```

Successful DNS resolution and outbound HTTPS connectivity from the private instance, together with increasing NAT rule counters, help validate the configuration.

## Troubleshooting

Common issues investigated during implementation:

- Incorrect network interface in NAT rules.
- IP forwarding disabled.
- Source/destination check enabled on the NAT instance.
- Missing private subnet route to the NAT instance.
- Incorrect security group or network ACL rules.
- Missing or incorrect iptables forwarding and masquerading rules.
- DNS resolution or outbound connectivity failures.

## Key Learnings

- Configuring AWS VPC public and private subnet routing.
- Understanding NAT instance traffic flow.
- Enabling Linux kernel IP forwarding.
- Configuring iptables NAT masquerading.
- Troubleshooting Linux and AWS network connectivity.
- Validating outbound connectivity from private EC2 instances.

## Security Considerations

- Restrict SSH access to trusted source addresses.
- Apply least-privilege security group rules.
- Avoid exposing private instances directly to the internet.
- Do not commit credentials, private keys, or sensitive infrastructure details.
- For production environments, evaluate AWS NAT Gateway as a managed alternative, including availability, scalability, cost, and operational requirements.

