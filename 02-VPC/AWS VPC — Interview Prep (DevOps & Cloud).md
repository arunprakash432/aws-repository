# AWS VPC — Interview Prep (DevOps & Cloud)

Oct 15, 2026 · @varun

## 1. Core notes

A VPC (Virtual Private Cloud) is your own logically isolated network inside AWS. You control the IP range, subnets, routing, gateways and security. Every EC2, RDS, EKS node, Lambda-in-VPC and load balancer lives inside one.

&#91;embedded content: standard production VPC · 2 AZs, 3 subnet tiers\]

Each AZ repeats the same three tiers; each private app subnet uses the NAT Gateway in its own AZ, and the DB tier has no internet route at all. Draw this on the whiteboard when asked to design a VPC.

### 1.1 Scope and IP addressing

- **VPC is regional.** It spans all Availability Zones (AZs) in a region.
- **Subnet is AZ-scoped.** One subnet lives in exactly one AZ. For high availability, create subnets in at least 2 AZs.
- **CIDR size:** between /16 (65,536 IPs) and /28 (16 IPs). Same range applies to subnets.
- **Secondary CIDRs:** you can add more IPv4 blocks to an existing VPC later (useful when IPs run out).
- **Use private ranges (RFC 1918):** 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16.
- **CIDR cannot be changed** after creation (only add secondary blocks). Plan it so it does not overlap with on-prem or other VPCs you may peer with.
- **IPv6:** optional, /56 per VPC and /64 per subnet. All IPv6 addresses are globally unique (public).

**5 reserved IPs per subnet** (example 10.0.1.0/24):

| IP | Reserved for |
| --- | --- |
| 10.0.1.0 | Network address |
| 10.0.1.1 | VPC router |
| 10.0.1.2 | Amazon DNS server (VPC base + 2) |
| 10.0.1.3 | Reserved for future use |
| 10.0.1.255 | Broadcast (AWS does not support broadcast, still reserved) |

So a /24 gives 256 − 5 = **251 usable IPs**. A /28 gives 16 − 5 = **11 usable**.

Formula: usable IPs = 2^(32 − prefix) − 5.

### 1.2 Default VPC vs custom VPC

|  | Default VPC | Custom VPC |
| --- | --- | --- |
| Created by | AWS, one per region | You |
| CIDR | 172.31.0.0/16 | Your choice |
| Subnets | One public /20 per AZ | You design public/private |
| Internet Gateway | Attached | You attach |
| Public IP on launch | Yes | Off by default |
| Use in production | No | Yes |

### 1.3 Core components

- **Subnet:** a slice of the VPC CIDR in one AZ. "Public" or "private" only depends on its route table.
- **Route table:** decides where traffic goes. Every subnet is associated with one route table (main route table if not set). Every route table has a **local route** (VPC CIDR → local) that cannot be deleted. AWS picks the **most specific route** (longest prefix match).
- **Internet Gateway (IGW):** lets resources with a public IP talk to the internet, both directions. One IGW per VPC. Horizontally scaled, highly available, no bandwidth limit, free.
- **Public subnet:** route `0.0.0.0/0 → IGW`. Instance also needs a public IP or Elastic IP.
- **Private subnet:** no route to IGW. Reaches internet (outbound only) through a NAT Gateway.
- **NAT Gateway:** managed, placed in a **public** subnet, needs an Elastic IP. Allows outbound-only internet for private subnets. AZ-scoped, so create one per AZ for HA. Charged per hour + per GB processed. No security group on it.
- **NAT Instance:** old way (EC2 doing NAT). You manage it, must **disable source/destination check**. Mostly replaced by NAT Gateway.
- **Egress-only Internet Gateway:** NAT-like outbound-only for **IPv6**.
- **Elastic IP (EIP):** static public IPv4. Charged (AWS charges for all public IPv4 addresses).
- **ENI (Elastic Network Interface):** virtual network card. Holds private IP, public IP, MAC, security groups.
- **DHCP options set:** sets DNS servers, domain name for the VPC.
- **DNS attributes:** `enableDnsSupport` (use Amazon DNS) and `enableDnsHostnames` (give public DNS names to instances). Both needed for private hosted zones and interface endpoints with private DNS.

### 1.4 Security: Security Groups vs NACLs

|  | Security Group | Network ACL |
| --- | --- | --- |
| Level | Instance / ENI | Subnet |
| State | **Stateful** (return traffic auto-allowed) | **Stateless** (must allow return traffic) |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated together | In number order, lowest first, first match wins |
| Default | Inbound deny all, outbound allow all | Default NACL allows all; custom NACL denies all |
| Reference other | Can reference another SG as source | Only CIDR |
| Use for | Main application firewall | Block specific IPs, subnet-level guardrail |

**Ephemeral ports:** clients use random ports 1024–65535 for return traffic. With NACLs (stateless) you must allow these on the return direction, or connections hang.

Traffic path into an instance: **Route table → NACL (subnet) → Security Group (ENI) → instance.**

### 1.5 Connectivity options

- **VPC Peering:** private connection between 2 VPCs (same/different account, same/different region). **Not transitive** (A↔B and B↔C does not mean A↔C). **CIDRs must not overlap.** Add routes on both sides + allow in SGs.
- **Transit Gateway (TGW):** regional hub that connects many VPCs, VPNs and Direct Connect. **Transitive**, uses TGW route tables. Use when you have more than a handful of VPCs.
- **VPC Endpoints** (private access to AWS services without internet/NAT):
  - **Gateway endpoint:** only **S3 and DynamoDB**. Added as a route-table entry. **Free.**
  - **Interface endpoint (PrivateLink):** ENI with private IP in your subnet. Most AWS services (ECR, SSM, CloudWatch, Secrets Manager, STS…). Charged per hour + per GB. Protected by security groups.
- **AWS PrivateLink:** expose your own service (behind an NLB) to other VPCs/accounts privately, without peering. Works even with overlapping CIDRs.
- **Site-to-Site VPN:** encrypted IPsec tunnels over the internet to on-prem. Terminates on a **Virtual Private Gateway (VGW)** or TGW. Each connection has **2 tunnels** for redundancy. Quick to set up.
- **Direct Connect (DX):** dedicated private physical line to AWS. Consistent latency, high bandwidth, takes weeks to provision. Not encrypted by default (add VPN over DX or MACsec).
- **Client VPN:** managed OpenVPN for individual users.
- **VPC Sharing (via AWS RAM):** share subnets of one VPC with other accounts in the same Organization.

### 1.6 Monitoring and troubleshooting tools

- **VPC Flow Logs:** capture IP traffic metadata (src/dst IP, port, protocol, ACCEPT/REJECT). At VPC, subnet or ENI level. Destinations: CloudWatch Logs, S3, Kinesis Data Firehose. Not real-time packet capture (no payload).
- Flow logs **do not** capture: traffic to Amazon DNS server, instance metadata (169.254.169.254), DHCP, Amazon Time Sync, Windows license activation.
- **Reachability Analyzer:** tests whether path A → B is reachable and shows which component blocks it (SG, NACL, route).
- **Network Access Analyzer:** finds unintended network access (for compliance).
- **Traffic Mirroring:** copies actual packets from an ENI to a monitoring appliance (deep packet inspection).

### 1.7 Access to private instances

- **Bastion host (jump box):** EC2 in public subnet; SSH to it, then to private instances. Restrict its SG to your IP.
- **SSM Session Manager (preferred):** no open port 22, no bastion, no SSH keys. Needs SSM agent + IAM role + internet/NAT or SSM interface endpoints.
- **EC2 Instance Connect Endpoint:** SSH/RDP into private instances without public IP or bastion.

### 1.8 VPC for containers (DevOps angle)

- **EKS with VPC CNI:** each pod gets a real VPC IP. Large clusters can **exhaust subnet IPs**. Fixes: bigger subnets, add a secondary CIDR (often 100.64.0.0/10) with custom networking, or enable **prefix delegation**.
- EKS needs subnet tags for load balancers: `kubernetes.io/role/elb = 1` (public) and `kubernetes.io/role/internal-elb = 1` (private).
- Private EKS nodes pulling images need NAT, or interface endpoints for **ECR (api + dkr)** plus a **gateway endpoint for S3** (image layers are stored in S3).

## 2. Interview questions and answers

Answer in this pattern: definition in one line → how it works → when you use it. Keep each answer under 60 seconds.

### 2.1 Basic (asked in almost every fresher interview)

**Q1. What is a VPC?** A logically isolated virtual network in AWS where I define the IP range, subnets, route tables and gateways. It is regional and spans all AZs in that region.

**Q2. What is a subnet? Is it regional or AZ-level?** A subnet is a range of IPs carved out of the VPC CIDR. It is AZ-level: one subnet belongs to exactly one AZ and cannot span AZs.

**Q3. What makes a subnet public or private?** Only the route table. If the subnet's route table has `0.0.0.0/0 → Internet Gateway`, it is public. If not, it is private. The instance also needs a public IP to be reachable from the internet.

**Q4. What is the allowed CIDR range for a VPC?** From /16 (65,536 IPs) to /28 (16 IPs). The primary CIDR cannot be changed later, but secondary CIDRs can be added.

**Q5. How many IPs does AWS reserve in each subnet? Which ones?** Five: the network address, .1 for the VPC router, .2 for Amazon DNS, .3 for future use, and the last address (broadcast). A /24 gives 251 usable IPs.

**Q6. What is an Internet Gateway?** A horizontally scaled, highly available VPC component that allows communication between resources with public IPs and the internet. One IGW per VPC, no bandwidth limit, no cost.

**Q7. What is a NAT Gateway and why do we need it?** It lets instances in private subnets start outbound connections to the internet (for OS updates, pulling packages, calling APIs) while blocking inbound connections from the internet. It sits in a public subnet and uses an Elastic IP.

**Q8. Difference between Internet Gateway and NAT Gateway?** IGW allows two-way traffic for instances with public IPs. NAT Gateway allows only outbound traffic for private instances. IGW is attached to the VPC; NAT Gateway lives in a specific public subnet/AZ. IGW is free; NAT Gateway is charged per hour and per GB.

**Q9. What is a route table? What is the local route?** A set of rules that decides where subnet traffic goes. Every route table has a local route for the VPC CIDR so all subnets can talk to each other. It cannot be deleted.

**Q10. What is the main route table?** The default route table created with the VPC. Any subnet not explicitly associated with a custom route table uses it. Best practice: keep the main route table private and create a separate public route table.

**Q11. Security Group vs NACL?** Security group works at the instance/ENI level, is stateful and supports only allow rules. NACL works at the subnet level, is stateless, supports allow and deny, and evaluates rules in number order. I use SGs as the main firewall and NACLs to block specific IPs or as a subnet guardrail.

**Q12. What does stateful vs stateless mean?** Stateful (SG): if inbound request is allowed, the response is automatically allowed out. Stateless (NACL): inbound and outbound are checked separately, so I must allow the response traffic (ephemeral ports 1024–65535) explicitly.

**Q13. What are the default rules of a security group?** New SG: no inbound rules (all inbound denied), all outbound allowed. The default SG of a VPC also allows inbound from members of the same SG.

**Q14. What is an Elastic IP?** A static public IPv4 address that stays the same when I stop/start or move it between instances. AWS charges for all public IPv4 addresses, including EIPs.

**Q15. What is the default VPC?** AWS creates one in every region with CIDR 172.31.0.0/16, a public /20 subnet in each AZ and an attached IGW. Good for testing, not for production.

### 2.2 Intermediate

**Q16. What is VPC Peering? What are its limitations?** A private connection between two VPCs using AWS's network, across accounts and regions. Limits: not transitive, CIDRs cannot overlap, no edge-to-edge routing (cannot use the peer's IGW, NAT or VPN), and I must update route tables on both sides.

**Q17. What is a Transit Gateway? When to use it over peering?** A regional hub that connects many VPCs and on-prem networks in a hub-and-spoke model. It is transitive and has its own route tables. Peering 10 VPCs in full mesh needs 45 connections (n(n−1)/2); TGW needs 10 attachments. Use TGW when VPC count grows or you need central routing/inspection.

**Q18. What are VPC endpoints? Gateway vs Interface?** They let resources in a VPC reach AWS services privately without IGW or NAT. Gateway endpoint supports only S3 and DynamoDB, is added as a route in the route table and is free. Interface endpoint creates an ENI with a private IP in my subnet, supports most services, uses security groups and costs per hour + per GB.

**Q19. Why use VPC endpoints instead of NAT?** Security (traffic never leaves the AWS network), compliance, and cost (NAT data processing charges are high for heavy S3/ECR traffic; a gateway endpoint for S3 is free).

**Q20. What are VPC Flow Logs?** They capture metadata of IP traffic (source/destination IP and port, protocol, bytes, ACCEPT or REJECT) at VPC, subnet or ENI level. Sent to CloudWatch Logs, S3 or Firehose. Used for troubleshooting connectivity and security audits. They do not capture packet contents.

**Q21. How do you connect a VPC to an on-premises data center?** Site-to-Site VPN (IPsec over internet, quick, cheaper, 2 tunnels for redundancy) or Direct Connect (dedicated private line, stable latency, higher bandwidth, weeks to set up). For critical workloads: Direct Connect as primary with VPN as backup.

**Q22. What is a bastion host? Is there a better option?** A hardened EC2 in a public subnet used to SSH into private instances. Better option: SSM Session Manager — no port 22 open, no keys, access controlled by IAM and every session logged.

**Q23. How do NACL rules get evaluated?** In ascending rule number order. The first rule that matches wins, the rest are skipped. The `*` rule at the end denies anything not matched. Leave gaps (100, 200, 300) so you can insert rules later.

**Q24. Can a security group deny traffic?** No. Security groups only have allow rules. To deny a specific IP, use a NACL deny rule (or AWS WAF/Network Firewall).

**Q25. Can two subnets in the same VPC have overlapping CIDRs?** No. Subnet CIDRs must be inside the VPC CIDR and cannot overlap each other.

### 2.3 Advanced (for 1–3 year roles or strong freshers)

**Q26. What is AWS PrivateLink?** A way to expose a service (behind a Network Load Balancer) to other VPCs or accounts through interface endpoints. Consumers access it privately; no peering, no route table changes, and overlapping CIDRs are fine. Only the specific service is exposed, not the whole network.

**Q27. Peering vs Transit Gateway vs PrivateLink — how do you choose?** Peering: few VPCs, full network access, lowest cost. TGW: many VPCs plus on-prem, transitive, central control. PrivateLink: expose one service only, consumer-provider model, works with overlapping CIDRs.

**Q28. How do you design a highly available VPC?** Subnets in at least 2–3 AZs, public and private subnets in each AZ, load balancer across public subnets, app and DB in private subnets, one NAT Gateway per AZ with each private route table pointing to its own AZ's NAT, and multi-AZ RDS.

**Q29. How do you design a 3-tier VPC?** Public tier: ALB and NAT Gateways. App tier (private): EC2/EKS nodes; SG allows traffic only from the ALB's SG. DB tier (private, no internet route): RDS; SG allows port 3306/5432 only from the app SG. Repeat per AZ.

**Q30. A private EKS cluster has no internet access. What does it need to work?** Interface endpoints for ECR (api and dkr), STS, EC2, CloudWatch Logs and any services the apps use, plus a gateway endpoint for S3 (ECR image layers live in S3). Also private DNS enabled on the endpoints.

**Q31. What is Network Firewall?** A managed, stateful firewall for the VPC with deep packet inspection, domain filtering and IPS rules. Placed in a dedicated firewall subnet; route tables send traffic through it. Used when SGs and NACLs are not enough.

**Q32. What is Reachability Analyzer?** A tool that checks the configured path between a source and destination (e.g., EC2 to RDS) without sending packets, and tells you exactly which SG, NACL or route blocks it.

**Q33. What is VPC sharing?** Using AWS Resource Access Manager, one account owns the VPC and shares subnets with other accounts in the same Organization. Teams launch resources in shared subnets while networking is managed centrally.

**Q34. How do you create a VPC with Terraform? (common in DevOps rounds)** Resources: `aws_vpc`, `aws_subnet` (public + private per AZ), `aws_internet_gateway`, `aws_eip` + `aws_nat_gateway`, `aws_route_table` + `aws_route_table_association`, `aws_security_group`. Or use the community module `terraform-aws-modules/vpc/aws`. Mention you keep state in S3 with locking.

## 3. Scenario-based questions and answers

For troubleshooting scenarios, always check in this order and say it out loud: **public IP → route table → NACL → security group → OS firewall / app.** Interviewers score the method, not just the answer.

### 3.1 Troubleshooting

**S1. You launched an EC2 in a public subnet but cannot SSH into it. What do you check?**

1. Instance has a public IP or Elastic IP.
2. Subnet route table has `0.0.0.0/0 → IGW`, and the IGW is attached to the VPC.
3. Security group allows inbound TCP 22 from your IP.
4. NACL allows inbound 22 **and** outbound ephemeral ports 1024–65535.
5. Correct key pair and username (`ec2-user` for Amazon Linux, `ubuntu` for Ubuntu).
6. Instance status checks pass; sshd running; OS firewall not blocking.

**S2. EC2 in a private subnet cannot run `yum update` / `apt update`.**

1. Private route table has `0.0.0.0/0 → NAT Gateway`.
2. NAT Gateway is in a **public** subnet (whose route table points to IGW) and has an Elastic IP.
3. NAT Gateway state is Available.
4. SG allows outbound 80/443 (default SG allows all outbound).
5. NACLs on both private and public subnets allow outbound 80/443 and inbound ephemeral ports.
6. DNS works: `enableDnsSupport` is true.

**S3. Security group allows port 80, but the website still times out. Why?** Likely the NACL. NACLs are stateless — if inbound 80 is allowed but outbound ephemeral ports (1024–65535) are not, the response is dropped. Also check route table, that the app is listening on 0.0.0.0:80 (not 127.0.0.1), and OS firewall.

**S4. You peered VPC A and VPC B, but instances still cannot ping each other.**

1. Peering request is **accepted** (status Active).
2. Route tables on **both** sides have routes to the other VPC's CIDR via the peering connection (pcx-…).
3. SGs allow ICMP (and the needed ports) from the peer CIDR.
4. NACLs allow it.
5. CIDRs do not overlap (if they did, peering would not have been created).

**S5. VPC A is peered with B, and B is peered with C. Can A talk to C?** No — peering is not transitive. Either create a direct A↔C peering or move to a Transit Gateway.

**S6. The application cannot connect to RDS in a private subnet.**

1. RDS SG allows the DB port (3306 MySQL / 5432 PostgreSQL) with source = **app server's SG**.
2. App and RDS are in the same VPC (or routed via peering/TGW).
3. NACLs allow the DB port and ephemeral return ports.
4. App uses the RDS **endpoint DNS name**, not an IP.
5. Use Reachability Analyzer to confirm the path.

**S7. Your NAT Gateway bill is very high. How do you reduce it?**

1. Check VPC Flow Logs to see where traffic goes.
2. Most often it is S3 or ECR traffic — add a **gateway endpoint for S3** (free) and interface endpoints for ECR.
3. Avoid cross-AZ traffic: one NAT per AZ, each private subnet routes to its own AZ's NAT.
4. Cache dependencies/images instead of pulling repeatedly.

**S8. EKS pods are stuck in Pending/ContainerCreating with "failed to assign an IP address".** Subnet IPs are exhausted (VPC CNI gives each pod a VPC IP). Fixes: enable prefix delegation, add a secondary CIDR (100.64.0.0/10) with custom networking, or create larger subnets and new node groups in them.

**S9. A specific IP is attacking your web server. Block it quickly.** Add a **NACL deny rule** with a lower number than the allow rules (e.g., rule 50 deny that IP). Security groups cannot deny. For a longer-term fix, use AWS WAF on the ALB.

**S10. How do you find out who is connecting to an instance or why traffic is rejected?** Enable VPC Flow Logs on the ENI/subnet, send to CloudWatch Logs, and query with Logs Insights filtering `action = REJECT`. REJECT on inbound points to SG/NACL; check which side using the ports.

**S11. Interface endpoint for Secrets Manager exists, but the app still goes through NAT.**

1. **Private DNS** is enabled on the endpoint (so the normal service hostname resolves to the private IP).
2. VPC has `enableDnsSupport` and `enableDnsHostnames` set to true.
3. Endpoint SG allows inbound 443 from the app's SG/subnet.
4. Endpoint is in the same subnets/AZs the app uses.

### 3.2 Design

**S12. Design a VPC for a production 3-tier web app.**

- VPC 10.0.0.0/16 across 2–3 AZs.
- Per AZ: public subnet (ALB, NAT Gateway), private app subnet (EC2/EKS), private DB subnet (RDS, no internet route).
- SG chaining: ALB SG allows 443 from internet → App SG allows 8080 only from ALB SG → DB SG allows 3306 only from App SG.
- One NAT Gateway per AZ; RDS Multi-AZ.
- Gateway endpoint for S3; SSM Session Manager instead of bastion; Flow Logs on.

**S13. Your company will connect 15 VPCs and an on-prem data center. What do you use?** Transit Gateway. Attach all VPCs and a VPN/Direct Connect. Full-mesh peering would need 105 connections and is not transitive. Use TGW route tables to separate prod and non-prod.

**S14. Another team's account needs to call one internal API in your VPC. Their CIDR overlaps with yours.** Peering and TGW won't work with overlapping CIDRs. Put the API behind a Network Load Balancer, create a **PrivateLink endpoint service**, and let them create an interface endpoint in their VPC. Only that API is exposed.

**S15. How would you pick a CIDR for a new VPC?** Use a private range, size it for growth (a /16 is common), make sure it does not overlap with on-prem or other VPCs you may peer later, and keep a central IP plan (or use VPC IPAM). For EKS, plan larger subnets because every pod takes an IP.

**S16. Compliance says private instances must never touch the internet, but they need S3 and CloudWatch.** No NAT, no IGW route for those subnets. Use a gateway endpoint for S3 and interface endpoints for CloudWatch Logs/Monitoring, SSM (ssm, ssmmessages, ec2messages) and anything else needed. Add endpoint policies to restrict access to specific buckets.

**S17. You need admin access to private servers without opening port 22.** SSM Session Manager: install/verify SSM agent, attach an IAM instance role with `AmazonSSMManagedInstanceCore`, and allow reach to SSM via NAT or SSM interface endpoints. Sessions are logged to CloudWatch/S3.

**S18. One AZ goes down. How does your VPC design keep the app running?** Resources run in at least 2 AZs, ALB spans those AZs, Auto Scaling replaces lost capacity in healthy AZs, RDS Multi-AZ fails over, and each AZ has its own NAT so the surviving AZ still has outbound internet.

## 4. Quick revision

### 4.1 Connectivity comparison

|  | VPC Peering | Transit Gateway | PrivateLink | Site-to-Site VPN | Direct Connect |
| --- | --- | --- | --- | --- | --- |
| Connects | VPC ↔ VPC | Many VPCs + on-prem | One service → consumers | VPC ↔ on-prem | VPC ↔ on-prem |
| Transitive | No | Yes | N/A | Via TGW | Via TGW / DX gateway |
| Overlapping CIDR | Not allowed | Not allowed | Allowed | Not allowed | Not allowed |
| Path | AWS backbone | AWS backbone | AWS backbone | Internet (IPsec) | Dedicated line |
| Setup time | Minutes | Minutes | Minutes | Minutes | Weeks |
| Best for | Few VPCs | Large networks | Sharing one API | Quick/backup hybrid | Stable, high bandwidth |

### 4.2 Gateways at a glance

|  | Internet Gateway | NAT Gateway | Egress-only IGW | Virtual Private Gateway |
| --- | --- | --- | --- | --- |
| Direction | In + out | Out only | Out only | To on-prem |
| IP version | IPv4 + IPv6 | IPv4 | IPv6 | IPv4 |
| Placement | Attached to VPC | Public subnet, per AZ | Attached to VPC | Attached to VPC |
| Cost | Free | Hourly + per GB | Free | Charged via VPN connection |

### 4.3 Must-remember numbers and facts

- VPC CIDR: /16 to /28. Usable per subnet = 2^(32 − prefix) − 5.
- 5 reserved IPs: .0, .1 router, .2 DNS, .3 future, last = broadcast.
- Default VPC CIDR: 172.31.0.0/16, default subnets /20.
- Default limit: 5 VPCs per region (soft limit, can be raised).
- One IGW per VPC. NAT Gateway is AZ-level.
- Gateway endpoints: only S3 and DynamoDB, free.
- Ephemeral ports: 1024–65535.
- Full-mesh peering links = n(n − 1) / 2.
- Site-to-Site VPN = 2 tunnels per connection.
- Amazon DNS = VPC base + 2 (e.g., 10.0.0.2), also reachable at 169.254.169.253.
- Instance metadata = 169.254.169.254.

### 4.4 Traps interviewers use

- "Is a subnet regional?" → No, AZ-level. The VPC is regional.
- "Can a security group block an IP?" → No, no deny rules. Use NACL.
- "Does NAT Gateway allow inbound from the internet?" → No.
- "Is peering transitive?" → No.
- "Can I change the VPC CIDR?" → No, only add secondary CIDRs.
- "Where does the NAT Gateway go?" → Public subnet, not private.
- "Does a public subnet make an instance public?" → Only with a public IP/EIP as well.
- "Do Flow Logs show packet contents?" → No, metadata only. Use Traffic Mirroring for packets.

### 4.5 Hands-on before the interview

- [ ] Build a custom VPC by hand in the console: 2 AZs, public + private subnets, IGW, NAT, route tables
- [ ] Launch a web server in public subnet and an app server in private subnet; reach the private one with SSM
- [ ] Break things on purpose (remove IGW route, block NACL ephemeral ports) and fix them
- [ ] Peer two VPCs and test ping across them
- [ ] Add an S3 gateway endpoint and confirm S3 access without NAT
- [ ] Enable Flow Logs and find a REJECT entry in CloudWatch Logs Insights
- [ ] Rebuild the same VPC with Terraform and push it to GitHub as a portfolio project
- [ ] Delete NAT Gateways and Elastic IPs after practice to avoid charges
