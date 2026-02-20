# PSC Test Procedure — SRT to Real Harmonic LRS via Private Service Connect

## Objective

Send an SRT test stream from a customer-side GCP project through **Private Service Connect (PSC)** to the **real Harmonic LRS** environment. This procedure validates the PSC connectivity model end-to-end and serves as the **production template** for the Globo deployment.

```
  CUSTOMER PROJECT: globo-customer           HARMONIC PROJECT: saas-vos360-dev-vos360admin

┌──────────────────────────┐              ┌──────────────────────────┐
│  globo-vpc               │     PSC      │  lrs-us-west1-0          │
│  10.100.0.0/24           │─────────────►│                          │
│                          │              │                          │
│  ┌────────────────────┐  │              │  ┌────────────────────┐  │
│  │ globo-srt-source   │──┼── SRT ──────►│  │ LRS Cloud Source   │  │
│  │ (Caller :7086)     │  │              │  │ (Listener :7086)   │  │
│  └────────────────────┘  │              │  └────────────────────┘  │
│                          │              │                          │
│  Network Attachment      │              │  PSC Interface           │
│  (accepts Harmonic       │              │  (gets IP from customer  │
│   project)               │              │   subnet: 10.100.0.x)   │
└──────────────────────────┘              └──────────────────────────┘
```

---

## Environment

| | Customer Side | Harmonic Side (LRS) |
|---|---|---|
| **Project ID** | `globo-customer` | `saas-vos360-dev-vos360admin` |
| **VPC** | `globo-vpc` | `lrs-us-west1-0` |
| **Subnet** | `globo-subnet` (10.100.0.0/24) | (managed by Harmonic) |
| **Region** | `us-west1` | `us-west1` (must match) |
| **Role** | Consumer / SRT Caller | Producer / SRT Listener |

---

## Prerequisites

- GCP account with billing enabled
- Cloud Shell access
- Harmonic LRS project ID: `saas-vos360-dev-vos360admin`
- Harmonic LRS VPC: `lrs-us-west1-0`
- Coordination with the LRS team for PSC Interface creation (Step 9)

---

## Part 1 — Customer Project Setup

> **What we're building:** The customer-side GCP environment — a VPC with a subnet, firewall rules, a VM to act as the SRT source, and the networking (Cloud NAT, IAP) to support it. For production, replace `globo-customer` with the actual customer GCP project ID.

### Step 1: Create the Project

```bash
gcloud projects create globo-customer --name="Globo Customer"
```

Link to billing:

```bash
# Find your billing account ID
gcloud billing accounts list

# Link the project
gcloud billing projects link globo-customer \
  --billing-account=YOUR_BILLING_ACCOUNT_ID
```

### Step 2: Enable Required APIs

> **What:** Compute Engine API is needed for VPCs, VMs, and firewall rules. IAP API is needed for SSH access to VMs without public IPs.

```bash
gcloud services enable compute.googleapis.com --project=globo-customer
gcloud services enable iap.googleapis.com --project=globo-customer
```

### Step 3: Set Environment Variables

```bash
export GLOBO_PROJECT="globo-customer"
export LRS_PROJECT="saas-vos360-dev-vos360admin"
export REGION="us-west1"
export ZONE="us-west1-a"
```

### Step 4: Create VPC and Subnet

> **What:** The VPC is the customer's isolated network. The subnet defines the IP range and region. The PSC Interface will get its IP from this subnet, making LRS reachable from within the customer's network.

```bash
gcloud compute networks create globo-vpc \
  --project=$GLOBO_PROJECT \
  --subnet-mode=custom

gcloud compute networks subnets create globo-subnet \
  --project=$GLOBO_PROJECT \
  --network=globo-vpc \
  --region=$REGION \
  --range=10.100.0.0/24
```

### Step 5: Create Firewall Rules

> **What:** GCP VPCs block all ingress traffic by default. We need rules for:
> - **IAP SSH:** Google's Identity-Aware Proxy range (`35.235.240.0/20`) so we can SSH into VMs without public IPs
> - **Internal traffic:** Allow communication within the subnet (including to PSC Interface IPs)

```bash
# Allow SSH via IAP
gcloud compute firewall-rules create globo-allow-iap-ssh \
  --project=$GLOBO_PROJECT \
  --network=globo-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --description="Allow SSH via IAP"

# Allow all internal traffic within the subnet
gcloud compute firewall-rules create globo-allow-internal \
  --project=$GLOBO_PROJECT \
  --network=globo-vpc \
  --allow=all \
  --source-ranges=10.100.0.0/24 \
  --description="Allow internal VPC traffic"
```

### Step 6: Create the SRT Source VM

> **What:** This VM simulates the customer's SRT encoder. It has no public IP (`no-address`) — in production, the real encoder connects via Google Direct Connect. We'll use `ffmpeg` to generate a test stream.

```bash
gcloud compute instances create globo-srt-source \
  --project=$GLOBO_PROJECT \
  --zone=$ZONE \
  --machine-type=e2-medium \
  --network-interface=network=globo-vpc,subnet=globo-subnet,no-address \
  --no-service-account \
  --no-scopes \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --metadata=enable-oslogin=true
```

### Step 7: Create Cloud NAT

> **What:** The VM has no public IP, so it cannot reach the internet to install packages. Cloud NAT provides outbound-only internet access without exposing the VM to inbound connections.

```bash
gcloud compute routers create globo-router \
  --project=$GLOBO_PROJECT \
  --network=globo-vpc \
  --region=$REGION

gcloud compute routers nats create globo-nat \
  --project=$GLOBO_PROJECT \
  --router=globo-router \
  --region=$REGION \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

### Step 8: Install SRT Tools

> **What:** SSH into the VM via IAP and install `srt-tools` (for SRT streaming) and `ffmpeg` (to generate a test video pattern).

```bash
gcloud compute ssh globo-srt-source \
  --project=$GLOBO_PROJECT \
  --zone=$ZONE \
  --tunnel-through-iap
```

Inside the VM:

```bash
sudo apt-get update && sudo apt-get install -y srt-tools ffmpeg
exit
```

---

## Part 2 — Set Up Private Service Connect

> **What we're building:** The PSC connection between the customer project and the Harmonic LRS project. The customer creates a Network Attachment (consumer side), shares the URL with Harmonic, and Harmonic creates a PSC Interface (producer side) on the LRS VM.

### Step 9: Create the Network Attachment

> **What:** The Network Attachment is the customer's side of PSC. It specifies which GCP projects are allowed to create PSC Interfaces into this subnet. We use `ACCEPT_MANUAL` so only the explicitly listed Harmonic LRS project can connect.

```bash
gcloud compute network-attachments create globo-psc-attachment \
  --project=$GLOBO_PROJECT \
  --region=$REGION \
  --connection-preference=ACCEPT_MANUAL \
  --producer-accept-list=$LRS_PROJECT \
  --subnets=globo-subnet
```

Verify:

```bash
gcloud compute network-attachments describe globo-psc-attachment \
  --project=$GLOBO_PROJECT \
  --region=$REGION
```

Confirm `producerAcceptLists` includes `saas-vos360-dev-vos360admin`.

### Step 10: Get the Network Attachment URL

> **What:** The Harmonic LRS team needs this URL to create the PSC Interface on their side. This is the handoff point between customer and Harmonic.

```bash
gcloud compute network-attachments describe globo-psc-attachment \
  --project=$GLOBO_PROJECT \
  --region=$REGION \
  --format="value(selfLink)"
```

Output example:

```
https://www.googleapis.com/compute/v1/projects/globo-customer/regions/us-west1/networkAttachments/globo-psc-attachment
```

**Save this URL — provide it to the LRS team in the next step.**

### Step 11: Harmonic Creates PSC Interface (LRS Side)

> **What:** This step is performed by the LRS team (or anyone with access to `saas-vos360-dev-vos360admin`). They add a PSC Interface to the LRS Cloud Source VM, pointing to the customer's Network Attachment. This gives the LRS VM an IP address in the customer's subnet (10.100.0.x), making it reachable from the customer's VPC.

**Provide the following to the LRS team:**

> **Request:** Create a PSC Interface for SRT-over-PSC testing.
>
> **Network Attachment URL:**
> `projects/globo-customer/regions/us-west1/networkAttachments/globo-psc-attachment`
>
> **What's needed:** Add a PSC Interface (additional network interface) to the LRS Cloud Source VM in project `saas-vos360-dev-vos360admin`, VPC `lrs-us-west1-0`, referencing the Network Attachment URL above.
>
> The PSC Interface will be assigned an IP from the customer subnet (10.100.0.0/24). Please share the assigned IP once created.

If you have access to the LRS project, the command depends on how LRS Cloud Source VMs are managed:

```bash
# Option A: If creating a new VM with both interfaces
gcloud compute instances create <LRS_VM_NAME> \
  --project=$LRS_PROJECT \
  --zone=$ZONE \
  --machine-type=<MACHINE_TYPE> \
  --network-interface=network=projects/$LRS_PROJECT/global/networks/lrs-us-west1-0,subnet=<LRS_SUBNET>,no-address \
  --network-interface=network-attachment=projects/$GLOBO_PROJECT/regions/$REGION/networkAttachments/globo-psc-attachment

# Option B: If adding an interface to an existing VM (VM must be stopped first)
# Coordinate with LRS team — this depends on their VM lifecycle management
```

### Step 12: Get the PSC Interface IP

> **What:** Once the LRS team creates the PSC Interface, they provide the IP assigned from the customer's subnet. This is the destination IP for the SRT stream.

If you have access to describe the LRS VM:

```bash
gcloud compute instances describe <LRS_VM_NAME> \
  --project=$LRS_PROJECT \
  --zone=$ZONE \
  --format="yaml(networkInterfaces)"
```

Look for the interface with a `networkAttachment` field pointing to `globo-psc-attachment`. The `networkIP` is the PSC IP.

```bash
# Save the PSC IP
export PSC_IP="<IP_FROM_LRS_TEAM>"   # e.g. 10.100.0.2
```

---

## Part 3 — Configure Firewall Rules for SRT

> **What we're doing:** SRT uses bidirectional UDP on port 7086. Even though the customer's encoder is the Caller (initiator), the LRS Listener sends back NAKs, ACKs, and retransmit requests. Both egress and ingress rules are required.

### Step 13: Add SRT Firewall Rules

```bash
# Egress: allow SRT packets from customer VM to LRS PSC IP
gcloud compute firewall-rules create globo-allow-srt-egress \
  --project=$GLOBO_PROJECT \
  --network=globo-vpc \
  --direction=EGRESS \
  --allow=udp:7086 \
  --destination-ranges=$PSC_IP/32 \
  --description="Allow SRT egress to LRS via PSC"

# Ingress: allow SRT return traffic (NAKs, ACKs) from LRS PSC IP
gcloud compute firewall-rules create globo-allow-srt-ingress \
  --project=$GLOBO_PROJECT \
  --network=globo-vpc \
  --direction=INGRESS \
  --allow=udp:7086 \
  --source-ranges=$PSC_IP/32 \
  --description="Allow SRT return traffic from LRS via PSC"
```

---

## Part 4 — Send SRT Stream and Verify

> **What we're doing:** Send a test video stream from the customer VM to the real LRS Cloud Source via PSC. If LRS receives the stream, PSC is working end-to-end.

### Step 14: Send the SRT Test Stream

SSH into the customer VM:

```bash
gcloud compute ssh globo-srt-source \
  --project=$GLOBO_PROJECT \
  --zone=$ZONE \
  --tunnel-through-iap
```

Send a test stream (colour bar pattern with 1kHz tone):

```bash
ffmpeg -re -f lavfi -i "testsrc=size=1280x720:rate=30" \
  -f lavfi -i "sine=frequency=1000:sample_rate=48000" \
  -c:v libx264 -b:v 2M -c:a aac \
  -f mpegts "srt://PSC_IP_HERE:7086?mode=caller"
```

> Replace `PSC_IP_HERE` with the actual PSC Interface IP from Step 12.

### Step 15: Verify on LRS Side

Confirm with the LRS team or via the LRS UI:

- [ ] LRS Cloud Source shows stream as **Active**
- [ ] SRT statistics show **0% packet loss** (private Google backbone)
- [ ] Connection source shows the PSC IP from the customer's subnet
- [ ] Stream content is valid (colour bars + tone)

---

## Cleanup

When testing is complete, delete the billable resources:

```bash
# Delete VM
gcloud compute instances delete globo-srt-source \
  --project=$GLOBO_PROJECT --zone=$ZONE --quiet

# Delete Cloud NAT
gcloud compute routers nats delete globo-nat \
  --router=globo-router --project=$GLOBO_PROJECT --region=$REGION --quiet

# Delete SRT firewall rules
gcloud compute firewall-rules delete globo-allow-srt-egress \
  --project=$GLOBO_PROJECT --quiet
gcloud compute firewall-rules delete globo-allow-srt-ingress \
  --project=$GLOBO_PROJECT --quiet
```

The following resources are free and can be kept for future tests or production use:
- VPC (`globo-vpc`)
- Subnet (`globo-subnet`)
- Cloud Router (`globo-router`)
- Network Attachment (`globo-psc-attachment`)
- IAP and internal firewall rules

---

## Production Adaptation

To use this procedure for the real Globo deployment:

| This Test | Production |
|---|---|
| Project: `globo-customer` | Project: Globo's actual GCP project ID |
| VM: `globo-srt-source` with ffmpeg | Real SRT encoders connected via Google Direct Connect |
| Single test stream | 10+ SRT services, each with its own LRS Cloud Source + PSC Interface |
| Manual ffmpeg command | Encoder hardware/software configured as SRT Caller to PSC IP:7086 |
| LRS project: `saas-vos360-dev-vos360admin` | LRS production project ID (from Harmonic) |

The Network Attachment is created **once** by the customer. Each additional SRT stream requires Harmonic to provision a **new LRS Cloud Source** with a **new PSC Interface** (same Network Attachment, new IP from the customer's subnet).
