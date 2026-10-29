# AWS PrivateLink Lab: Shared Services Across VPCs

Hands-on lab for the **AWS Certified Solutions Architect – Associate** exam.
It rebuilds a real exam scenario with **CloudFormation**, step by step, and links every decision to the **AWS Well-Architected Framework**.

> **Status:** ✅ Complete. All 6 steps built and tested: `curl` from VPC B returns `Hello from shared services` through PrivateLink.

---

## Table of contents

1. [The exam question](#1-the-exam-question)
2. [What is PrivateLink?](#2-what-is-privatelink)
3. [Architecture](#3-architecture)
4. [How PrivateLink works here](#4-how-privatelink-works-here)
5. [Step-by-step build](#5-step-by-step-build)
6. [CLI commands](#6-cli-commands)
7. [FAQ](#7-faq)
8. [Troubleshooting](#8-troubleshooting)
9. [Well-Architected mapping](#9-well-architected-mapping)
10. [Exam tips](#10-exam-tips)
11. [Cost and cleanup](#11-cost-and-cleanup)

---

## 1. The exam question

A company has **50+ AWS accounts**, each with **at least 4 VPCs** (200+ VPCs in total). Every VPC must reach **one shared services VPC**.

**How do they do it with the least operational overhead?**

| Option                                                         | Verdict | Why                                                                                                        |
| -------------------------------------------------------------- | ------- | ---------------------------------------------------------------------------------------------------------- |
| ALB + CAA record in Route 53                                   | ❌      | A CAA record only controls which certificate authorities can issue certificates. It doesn't route traffic. |
| VPC peering to each VPC                                        | ❌      | 200+ connections, each needing acceptance and route updates. Not transitive. CIDRs can't overlap.          |
| **NLB + PrivateLink (endpoint service + interface endpoints)** | ✅      | Publish the service **once**. Each VPC adds **one endpoint**. No routes. Overlapping CIDRs are fine.       |
| VPN per VPC                                                    | ❌      | 200+ VPN connections over the internet. The most overhead.                                                 |

This lab builds a mini version: **1 provider VPC + 1 consumer VPC**. PrivateLink works the same with 1 consumer or 200.

---

## 2. What is PrivateLink?

A way to **publish one service** from a VPC so other VPCs can use it **privately, without connecting the networks**.

**Analogy: a shop counter.** The consumer can ask for the one service at the counter, but can't walk into the warehouse (the rest of the provider VPC).

| Side                 | Builds                                                     | Role                                                      |
| -------------------- | ---------------------------------------------------------- | --------------------------------------------------------- |
| **Provider** (VPC A) | Service → **Network Load Balancer** → **Endpoint service** | Publishes the service                                     |
| **Consumer** (VPC B) | **Interface endpoint**                                     | A private IP inside its own VPC that leads to the service |

**Peering connects two networks. PrivateLink connects a consumer to one service.**

---

## 3. Architecture

```mermaid
flowchart LR
  subgraph B["VPC B (consumer) 10.0.0.0/16"]
    U["You (browser / CLI)"] -. "SSH via IAM" .-> EIC["EC2 Instance<br/>Connect Endpoint"] -.-> T
    T["Test EC2<br/>curl"] --> E["Interface endpoint<br/>(ENI, VPC B IP)"]
  end
  subgraph A["VPC A (provider) 10.0.0.0/16"]
    S["Endpoint service"] --> N["Internal NLB<br/>(2 AZs)"] --> I["EC2 :80<br/>Hello from shared services"]
  end
  E == "PrivateLink<br/>(AWS backbone)" ==> S
```

Both VPCs use the **same CIDR (10.0.0.0/16) on purpose**. VPC peering would be refused, but PrivateLink works fine.

| Step | What                                              | Where | Status |
| ---- | ------------------------------------------------- | ----- | ------ |
| 1    | Two VPCs, same CIDR, 2 subnets each in 2 AZs      | Both  | ✅     |
| 2    | "Shared service" EC2 + security group             | VPC A | ✅     |
| 3    | Internal NLB + target group + listener            | VPC A | ✅     |
| 4    | Endpoint service + allowed principals             | VPC A | ✅     |
| 5    | Interface endpoint + security group               | VPC B | ✅     |
| 6    | Test EC2 + EC2 Instance Connect Endpoint + `curl` | VPC B | ✅     |

---

## 4. How PrivateLink works here

1. **Test EC2** (VPC B) calls the endpoint's DNS name.
2. DNS resolves to the **endpoint ENI**, which has a **private IP from VPC B**. To the consumer, it looks local.
3. **AWS carries the traffic** over its private network to VPC A. No routes, no internet.
4. The **endpoint service** hands the traffic to the NLB it publishes.
5. The **NLB** sends it to a healthy target.
6. The **EC2 instance** replies, and the reply goes back the same way.

**The trick: AWS translates the addresses in the middle.**

- The consumer only ever sees an IP **in its own VPC** (the endpoint ENI).
- The instance only ever sees the **NLB's IP in its own VPC**.

So:

- **Overlapping CIDRs don't matter.**
- **No route tables change.** It's a service connection, not a network connection.
- **The consumer reaches only this service** (port 80 behind the NLB), not the rest of VPC A.
- **One direction only:** the consumer starts the connection. VPC A can never reach into VPC B.

**Scaling to 200+ VPCs:** the provider builds once (service, NLB, endpoint service, permissions). Each new consumer only adds one interface endpoint. Nothing changes in VPC A.

---

## 5. Step-by-step build

Everything lives in **one template** ([`privatelink-lab.yaml`](privatelink-lab.yaml)) that grows each step. You **update the same stack** each time, so you can see exactly what every step adds. The template is fully commented.

### Step 1: Two VPCs

- **VPC A (provider)** and **VPC B (consumer)**, both `10.0.0.0/16`.
- 2 subnets each (`10.0.1.0/24`, `10.0.2.0/24`) in **2 AZs**, using `!Select [0, !GetAZs '']` so the template works in any Region.
- `EnableDnsSupport` and `EnableDnsHostnames` turned on so endpoint DNS names resolve.
- **Fully private.** No route tables are defined, so every subnet uses the VPC's **main route table**, which has only `10.0.0.0/16 → local`. There's no Internet Gateway.

### Step 2: The shared service (VPC A)

- `t3.micro`, latest **Amazon Linux 2023**, read from SSM Parameter Store (no hard-coded AMI IDs).
- User data starts a tiny web server: `python3 -m http.server 80` serving "Hello from shared services". Python is pre-installed, so no internet is needed.
- **Security group:** TCP 80 only from `10.0.0.0/16` (VPC A).
  With PrivateLink, the instance never sees VPC B. Traffic arrives from the **NLB's private IPs in VPC A**, plus health checks.

### Step 3: Internal NLB (VPC A)

- **Target group:** TCP 80 → the EC2 instance, with a TCP health check every 10s.
- **NLB:** `Scheme: internal`, in **both VPC A subnets** (one node per AZ).
- **Listener:** TCP 80 → forward to the target group.
- **Cross-zone load balancing = on.** The only instance is in AZ1. Without cross-zone (the NLB default), the AZ2 node has no targets. The consumer's endpoint in AZ2 talks to that AZ2 node, so its requests would fail.
  In production, put targets in **every AZ** instead (cross-zone can add inter-AZ data charges).

### Step 4: Endpoint service (VPC A)

- **Endpoint service:** publishes the NLB through PrivateLink.
- **Two gates control access:**
  1. **Allowed principals:** which accounts can even request a connection. Here: `arn:aws:iam::<account-id>:root`. For cross-account, add each consumer account. **Never use `*`.**
  2. **Acceptance required = false:** connections from allowed principals are accepted automatically. With 200+ VPCs, manual approval would be a lot of overhead, and gate 1 already limits who can ask.
- **Service name** (in Outputs): `com.amazonaws.vpce.<region>.vpce-svc-...`. This is the "address" you share with consumers.
- The endpoint service is **free** for the provider.

| Exam question wording           | Template                    |
| ------------------------------- | --------------------------- |
| "NLB across the AZs"            | `ServiceNLB`                |
| "Service consumer roles in IAM" | `AllowedPrincipals`         |
| "Automatically accept"          | `AcceptanceRequired: false` |
| "Consumer endpoint in each VPC" | Step 5                      |

### Step 5: Interface endpoint (VPC B)

- **Interface endpoint** pointing to the service name from Step 4, in **both VPC B subnets**.
- AWS creates **one ENI per subnet (per AZ)**, each with a **private IP from VPC B**, plus DNS names:

  | DNS name                                            | Type         | Use                                                           |
  | --------------------------------------------------- | ------------ | ------------------------------------------------------------- |
  | `vpce-xxx.vpce-svc-yyy.<region>.vpce.amazonaws.com` | **Regional** | ✅ Default. Resolves to all AZs, so it survives an AZ failure |
  | `vpce-xxx-<az>.vpce-svc-yyy...`                     | Zonal        | Keep traffic in one AZ (e.g. avoid inter-AZ charges)          |

  The ID in brackets shown in the console (e.g. `Z38GZ743OKFT7T`) is the Route 53 hosted zone ID, not part of the name.

- **Security group on the endpoint:** TCP 80 from VPC B. It controls **who in VPC B** can use the endpoint. There are now two SGs on the path: one in VPC B (who can use the door) and one in VPC A (NLB → instance).
- `PrivateDnsEnabled: false`, because no custom private DNS name was set on the endpoint service.
- **`DependsOn: EndpointServicePermissions`.** CloudFormation builds in parallel, so if the endpoint asks to connect before the account is allowed, the request is rejected. In a real multi-account setup the order is the same: the provider grants access, then the consumer creates its endpoint.
- **Still no route tables changed.** The endpoint IPs are inside VPC B, so the `local` route already reaches them.
- **Interface vs Gateway endpoints:** Interface = ENIs with private IPs (any PrivateLink service). **Gateway** endpoints exist only for **S3 and DynamoDB** and work through route table entries.

### Step 6: Test from VPC B

- **Test EC2** in a private VPC B subnet, playing a "country account" client.
- **EC2 Instance Connect Endpoint (EIC):** lets you SSH into a **private** instance with no public IP, no Internet Gateway and no bastion host. Access is authenticated with **IAM**. It's free.
- Security groups: the test instance allows **SSH only from the EIC Endpoint SG**, and the EIC SG allows **SSH out only to the test instance**. A separate `SecurityGroupIngress` resource avoids a circular dependency between the two SGs.

**Connect**, in either of two ways:

- **Console:** EC2 → `test-client-ec2` → **Connect** → **EC2 Instance Connect** → _Connect using EC2 Instance Connect Endpoint_ → user `ec2-user`.
- **CLI v2 only:** `aws ec2-instance-connect ssh --instance-id <TestInstanceId> --connection-type eice`

**Test** (use the `EndpointDnsName` output, the regional name):

```bash
getent hosts <EndpointDnsName>                # → 10.0.1.x / 10.0.2.x (VPC B's own IPs)
curl http://<EndpointDnsName>                 # → Hello from shared services ✅
curl --max-time 5 http://<ServicePrivateIp>   # → times out ❌
```

The last test proves it's **not a network connection**: `10.0.1.x` inside VPC B means VPC B's own subnet, not VPC A. The consumer reaches **only the service**, and the overlapping CIDR didn't break anything.

---

## 6. CLI commands

Run from the folder that contains the template.

```bash
# Check your setup
aws sts get-caller-identity
aws configure get region

# Validate
aws cloudformation validate-template --template-body file://privatelink-lab.yaml

# Preview changes (change set), then deploy (creates the 1st time, updates after)
aws cloudformation deploy --template-file privatelink-lab.yaml \
  --stack-name privatelink-lab --no-execute-changeset
aws cloudformation deploy --template-file privatelink-lab.yaml \
  --stack-name privatelink-lab

# Status and outputs
aws cloudformation describe-stacks --stack-name privatelink-lab \
  --query "Stacks[0].StackStatus"
aws cloudformation describe-stacks --stack-name privatelink-lab \
  --query "Stacks[0].Outputs" --output table

# Why did it fail?
aws cloudformation describe-stack-events --stack-name privatelink-lab \
  --query "StackEvents[?contains(ResourceStatus,'FAILED')].[LogicalResourceId,ResourceStatusReason]" \
  --output table

# Step 1: confirm the subnets are private (only a "local" route)
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<VpcId>" \
  --query "RouteTables[].Routes[]" --output table

# Step 3: is the target healthy?
aws elbv2 describe-target-health --target-group-arn <ServiceTargetGroupArn> \
  --query "TargetHealthDescriptions[].TargetHealth.State"

# Step 4: endpoint service state and permissions
aws ec2 describe-vpc-endpoint-service-configurations \
  --query "ServiceConfigurations[].[ServiceName,ServiceState,AcceptanceRequired]" --output table
aws ec2 describe-vpc-endpoint-service-permissions --service-id <EndpointServiceId>

# Step 5: interface endpoint state and its ENIs
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <ConsumerEndpointId> \
  --query "VpcEndpoints[].[State,NetworkInterfaceIds]"

# Step 6: connect to the test client (CLI v2 only)
aws ec2-instance-connect ssh --instance-id <TestInstanceId> --connection-type eice

# Get just the values needed for testing
aws cloudformation describe-stacks --stack-name privatelink-lab \
  --query "Stacks[0].Outputs[?OutputKey=='EndpointDnsName' || OutputKey=='ServicePrivateIp'].[OutputKey,OutputValue]" \
  --output table

# Cleanup
aws cloudformation delete-stack --stack-name privatelink-lab
aws cloudformation wait stack-delete-complete --stack-name privatelink-lab
```

---

## 7. FAQ

**Are the subnets private?**
Yes. There's no Internet Gateway, and the main route table only has the `local` route. The two VPCs can't talk to each other either. PrivateLink solves that **without adding any route**.

**Does the NLB open the VPC to the internet?**
No, because it's **internal** (private IPs only). VPC A also has no Internet Gateway. The only way in is the endpoint service, and only for allowed principals.

| Scheme          | Gets        | Reachable from                           |
| --------------- | ----------- | ---------------------------------------- |
| Internet-facing | Public IPs  | The internet (needs an Internet Gateway) |
| **Internal**    | Private IPs | Inside the VPC, and through PrivateLink  |

**Why an NLB and not an ALB?**
PrivateLink endpoint services **only accept an NLB** (or a Gateway Load Balancer for appliances). PrivateLink works at **layer 4** (TCP/UDP), and the NLB is a layer-4 load balancer. If you need ALB features, put the ALB **behind** the NLB (an NLB target group of type `alb`).

**In which VPC is the NLB?**
**VPC A (provider)**, next to the service it fronts. Consumers never get an NLB, only an interface endpoint.

**How is the DNS name created?**
Automatically. AWS generates `vpce-...vpce.amazonaws.com` when the interface endpoint is created. For a friendly name like `shared.mycompany.com`, you have two options:

- The provider sets a **private DNS name** on the endpoint service (domain ownership is verified with a TXT record), and consumers enable private DNS.
- The consumer creates a **Route 53 private hosted zone** with a record pointing to the endpoint.

**Do we need SSL/TLS?**
It's **not needed for PrivateLink to work**, because the traffic stays on the AWS network. This lab uses HTTP to keep things simple. For **production it's recommended (SEC09-BP02)**. Options:

- A **TLS listener on the NLB** with an **ACM certificate** (offloads encryption, and ACM auto-renews it, per SEC09-BP01).
- **TCP 443 pass-through**, where the instance handles TLS itself.

The certificate must match the name clients use, so in practice you pair TLS with a friendly private DNS name.

**Why an EC2 Instance Connect Endpoint instead of a public subnet?**
A public subnet needs an Internet Gateway, a public IP and SSH open to your IP, which opens VPC B to the internet. The EIC Endpoint keeps VPC B **100% private** and controls access with IAM instead of open ports.

---

## 8. Troubleshooting

| Problem                                                                                      | Cause                                                                             | Fix                                                                                                                                                                           |
| -------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `aws ec2-instance-connect ssh` → `Invalid choice, valid choices are: send-ssh-public-key...` | You're on **AWS CLI v1**. `ssh` / `open-tunnel` exist only in **v2**              | Connect from the console, or install v2 (`brew install awscli`, or the official AWS `.pkg`)                                                                                   |
| `aws --version` still shows 1.x after installing v2                                          | An old v1 (installed with pip) comes first in `PATH`                              | `which -a aws` to see every copy. `head -1 <path>` shows which Python owns it. Uninstall with `<that-python> -m pip uninstall awscli --break-system-packages`, then `hash -r` |
| `getent hosts eic-endpoint-vpc-b` returns nothing                                            | That's the **tag** of the Instance Connect Endpoint, not a DNS name               | Use the **interface endpoint's** regional DNS name (`EndpointDnsName` output)                                                                                                 |
| Interface endpoint creation fails                                                            | Your account isn't in the **allowed principals** yet                              | Check Step 4 permissions, and keep `DependsOn: EndpointServicePermissions`                                                                                                    |
| Endpoint stuck in `pendingAcceptance`                                                        | `AcceptanceRequired: true`                                                        | Accept the connection on the endpoint service, or set it to `false`                                                                                                           |
| Target `unhealthy`                                                                           | The instance SG doesn't allow the NLB                                             | Allow TCP 80 from VPC A's CIDR                                                                                                                                                |
| `curl` hangs                                                                                 | Endpoint SG doesn't allow TCP 80 from the client, or there's no target in that AZ | Check the endpoint SG, and enable cross-zone or add targets in every AZ                                                                                                       |
| Can't connect with Instance Connect                                                          | The EIC Endpoint is still being created, or you're in the wrong Region            | Wait until it's **Available** (VPC → Endpoints), and check the Region                                                                                                         |

---

## 9. Well-Architected mapping

| Pillar / best practice                                         | Where it shows up                                                                                                          |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **OPS** – Perform operations as code                           | Whole lab in one CloudFormation template                                                                                   |
| **OPS** – Make frequent, small, reversible changes             | One step per update, change-set preview, one-command delete                                                                |
| **SEC03** – Grant least privilege                              | Allowed principals = only the accounts that need the service, never `*`                                                    |
| **SEC05-BP01** – Create network layers                         | Private subnets, no Internet Gateway                                                                                       |
| **SEC05-BP02** – Control traffic at all layers                 | SG opens only TCP 80 from VPC A. Internal NLB. PrivateLink: only the consumer initiates, traffic stays on the AWS backbone |
| **SEC06** – Protect compute                                    | No public IPs, no internet. Access through the EC2 Instance Connect Endpoint with IAM, not bastion hosts or open ports     |
| **SEC09-BP01 / BP02** – Certificates and encryption in transit | Production: TLS on the NLB with ACM                                                                                        |
| **REL02-BP03** – IP allocation for expansion and availability  | Subnets in 2 AZs, `/16` leaves room to grow                                                                                |
| **REL02-BP04** – Prefer hub-and-spoke over many-to-many mesh   | One endpoint service (hub), many consumer endpoints (spokes). Peering among many VPCs is an anti-pattern                   |
| **REL10** – Use fault isolation (multiple AZs)                 | NLB and endpoint ENIs in 2 AZs, regional DNS name. Production: targets in every AZ                                         |
| **PERF05** – Use load balancing                                | NLB for layer 4, high performance, low latency                                                                             |
| **COST** – Avoid unused resources                              | Delete the stack when finished                                                                                             |

---

## 10. Exam tips

- **Many VPCs/accounts → one service, least overhead** → **PrivateLink (NLB + endpoint service)**.
- **All VPCs must talk to each other** → **Transit Gateway**.
- **Overlapping CIDRs** → PrivateLink works. Peering and Transit Gateway routing don't.
- **PrivateLink endpoint service** → always an **NLB** (or GWLB). An option with an **ALB** as the endpoint service is wrong.
- **Peering** is not transitive and gets heavy beyond a few VPCs.
- **Interface endpoint** = ENIs with private IPs (PrivateLink). **Gateway endpoint** = route table entry, **only S3 and DynamoDB**.
- **SSH to private instances without a bastion or public IP** → **EC2 Instance Connect Endpoint** (or Systems Manager Session Manager).
- Ask yourself: **"How many connections do I have to create and maintain?"**

---

## 11. Cost and cleanup

| Resource                                                                        | Billing                                       |
| ------------------------------------------------------------------------------- | --------------------------------------------- |
| VPCs, subnets, security groups, endpoint service, EC2 Instance Connect Endpoint | Free                                          |
| 2 × EC2 `t3.micro`                                                              | Per hour (Free Tier eligible on new accounts) |
| Network Load Balancer                                                           | Per hour + usage                              |
| Interface endpoint                                                              | Per hour per AZ + per GB                      |

⚠️ **Delete the stack when you finish:**

```bash
aws cloudformation delete-stack --stack-name privatelink-lab
aws cloudformation wait stack-delete-complete --stack-name privatelink-lab
```
