# PSC Lab Guide — Self-Contained SRT-over-PSC Test

## What Are We Building?

We're simulating the **Globo ↔ LRS** Private Service Connect setup entirely within one GCP project. Instead of needing access to the real LRS project, we create **two isolated VPCs** — one pretending to be Globo (customer), the other pretending to be LRS (Harmonic) — and connect them via PSC.

```
┌──────────────────────────┐              ┌──────────────────────────┐
│  "Globo" VPC             │     PSC      │  "LRS" VPC               │
│  10.100.0.0/24           │◄────────────►│  10.200.0.0/24           │
│                          │              │                          │
│  ┌────────────────────┐  │              │  ┌────────────────────┐  │
│  │ globo-srt-source   │──┼── SRT ──────►│  │ lrs-srt-listener   │  │
│  │ (Caller :7086)     │  │              │  │ (Listener :7086)   │  │
│  └────────────────────┘  │              │  └────────────────────┘  │
│                          │              │                          │
│  Network Attachment      │              │  PSC Interface           │
│  (Consumer)              │              │  (Producer — gets IP     │
│                          │              │   from Globo's subnet)   │
└──────────────────────────┘              └──────────────────────────┘

Both VPCs in project: srt-sources
```

**Why this works as a simulation:** In the real setup, the "LRS" VPC would be in Harmonic's project and the "Globo" VPC in the customer's project. PSC works the same way whether the VPCs are in the same project or different projects. Once this lab works, the exact same steps apply to the real deployment.

---

## Prerequisites

- GCP account with free trial credits ($300)
- Cloud Shell access (or `gcloud` CLI locally)
- Project: `srt-sources`

---

## Setup — Environment Variables

> **What:** Set shell variables used by every command in the guide. This avoids hardcoding project/region in each command.

```bash
export PROJECT_ID="srt-sources"
export REGION="us-west1"
export ZONE="us-west1-a"
```

> **What:** Enable the Compute Engine API. Required before creating any VMs, VPCs, or firewall rules.

```bash
gcloud services enable compute.googleapis.com --project=$PROJECT_ID
```

---

## Part 1 — Create the "Customer" VPC (Simulating Globo)

> **What we're achieving:** Building Globo's network environment. In the real world, Globo already has a VPC with their SRT encoders. Here we create a fresh VPC to simulate it.

### Step 1: Create the Customer VPC

> **What:** A VPC (Virtual Private Cloud) is an isolated network. `--subnet-mode=custom` means we manually define subnets (no auto-created ones). This gives us full control over IP ranges and regions.

```bash
gcloud compute networks create globo-vpc \
  --project=$PROJECT_ID \
  --subnet-mode=custom
```

### Step 2: Create a subnet in the Customer VPC

> **What:** A subnet defines an IP range within a VPC, tied to a specific region. This is where the Globo SRT source VM will get its IP, and where the **Network Attachment** will be created later. The PSC Interface on the LRS VM will also get its IP from this subnet — that's how the two VPCs talk to each other.

```bash
gcloud compute networks subnets create globo-subnet \
  --project=$PROJECT_ID \
  --network=globo-vpc \
  --region=$REGION \
  --range=10.100.0.0/24
```

> This gives us 254 usable IPs (10.100.0.1 – 10.100.0.254).

### Step 3: Create firewall rules for the Customer VPC

> **What:** VPCs block all traffic by default. We need rules to allow: (1) SSH access via Google IAP so we can log into the VM, and (2) internal traffic between VMs in the subnet.

```bash
# Rule 1: Allow SSH via IAP (Identity-Aware Proxy)
# 35.235.240.0/20 is Google's IAP IP range. This lets us SSH into VMs
# that have NO public IP — Google tunnels the SSH connection through IAP.
gcloud compute firewall-rules create globo-allow-iap-ssh \
  --project=$PROJECT_ID \
  --network=globo-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --description="Allow SSH via IAP"

# Rule 2: Allow all traffic within the subnet (VM to VM)
gcloud compute firewall-rules create globo-allow-internal \
  --project=$PROJECT_ID \
  --network=globo-vpc \
  --allow=all \
  --source-ranges=10.100.0.0/24 \
  --description="Allow internal VPC traffic"
```

---

## Part 2 — Create the "LRS" VPC (Simulating Harmonic LRS)

> **What we're achieving:** Building the Harmonic LRS network. In reality, this is a separate GCP project managed by Harmonic. Here we simulate it as a second VPC in the same project. The key point: **the two VPCs are completely isolated from each other** — just like two VPCs in different projects. The only way they'll communicate is through PSC.

### Step 4: Create the LRS VPC

> **What:** A separate isolated network for the LRS side. Uses a different IP range (10.200.x.x) to make it clear these are two different networks.

```bash
gcloud compute networks create lrs-vpc \
  --project=$PROJECT_ID \
  --subnet-mode=custom
```

### Step 5: Create a subnet in the LRS VPC

> **What:** The LRS VM's primary network interface will sit in this subnet. Note the different CIDR range (10.200.0.0/24) — completely separate from Globo's 10.100.0.0/24.

```bash
gcloud compute networks subnets create lrs-subnet \
  --project=$PROJECT_ID \
  --network=lrs-vpc \
  --region=$REGION \
  --range=10.200.0.0/24
```

### Step 6: Create firewall rules for the LRS VPC

> **What:** Same pattern as Globo — SSH via IAP and internal traffic. Plus one additional rule: allow SRT (UDP port 7086) traffic arriving from the **customer's subnet via PSC**. The PSC interface will have an IP from 10.100.0.0/24 (Globo's subnet), so we allow that range.

```bash
# Allow SSH via IAP
gcloud compute firewall-rules create lrs-allow-iap-ssh \
  --project=$PROJECT_ID \
  --network=lrs-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --description="Allow SSH via IAP"

# Allow all internal traffic within LRS VPC
gcloud compute firewall-rules create lrs-allow-internal \
  --project=$PROJECT_ID \
  --network=lrs-vpc \
  --allow=all \
  --source-ranges=10.200.0.0/24 \
  --description="Allow internal VPC traffic"

# Allow SRT traffic from customer VPC via PSC
# The PSC interface gets an IP from Globo's subnet (10.100.0.0/24),
# so SRT packets arriving on the LRS VM will have a source IP from that range.
gcloud compute firewall-rules create lrs-allow-srt-from-psc \
  --project=$PROJECT_ID \
  --network=lrs-vpc \
  --allow=udp:7086 \
  --source-ranges=10.100.0.0/24 \
  --description="Allow SRT traffic from customer VPC via PSC"
```

---

## Part 3 — Create the VMs

> **What we're achieving:** Creating the two endpoints of our SRT stream — the Listener (simulating LRS Cloud Source) and the Caller (simulating Globo's SRT encoder). Both VMs have **no public IP** (`no-address`), which mirrors the real scenario where traffic flows privately via PSC, not over the public internet.

### Step 7: Create the SRT Listener VM (simulating LRS Cloud Source)

> **What:** This VM sits in the LRS VPC and will run `srt-live-transmit` in listener mode on port 7086 — exactly what the real LRS Cloud Source does. `no-address` means no public IP.

```bash
gcloud compute instances create lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --machine-type=e2-medium \
  --network-interface=network=lrs-vpc,subnet=lrs-subnet,no-address \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB
```

### Step 8: Create the SRT Caller VM (simulating Globo SRT Source)

> **What:** This VM sits in the Globo VPC and will run `ffmpeg` or `srt-live-transmit` in caller mode — simulating Globo's SRT encoder sending a stream to LRS.

```bash
gcloud compute instances create globo-srt-source \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --machine-type=e2-medium \
  --network-interface=network=globo-vpc,subnet=globo-subnet,no-address \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB
```

### Step 9: Add Cloud NAT for both VPCs

> **What:** Our VMs have no public IP, so they can't reach the internet directly. But we need internet access to install packages (`apt-get`). Cloud NAT lets VMs access the internet for outbound connections (like downloading packages) without having a public IP. We need one per VPC.

```bash
# Cloud NAT for Globo VPC
gcloud compute routers create globo-router \
  --project=$PROJECT_ID \
  --network=globo-vpc \
  --region=$REGION

gcloud compute routers nats create globo-nat \
  --project=$PROJECT_ID \
  --router=globo-router \
  --region=$REGION \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges

# Cloud NAT for LRS VPC
gcloud compute routers create lrs-router \
  --project=$PROJECT_ID \
  --network=lrs-vpc \
  --region=$REGION

gcloud compute routers nats create lrs-nat \
  --project=$PROJECT_ID \
  --router=lrs-router \
  --region=$REGION \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

### Step 10: Install SRT tools on both VMs

> **What:** Install the SRT command-line tools (`srt-live-transmit`) and `ffmpeg` (for generating a test video stream). We SSH in via IAP tunnel since the VMs have no public IP.

**SSH into the LRS Listener VM:**

```bash
gcloud compute ssh lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --tunnel-through-iap
```

Inside the VM, run:

```bash
sudo apt-get update && sudo apt-get install -y srt-tools
exit
```

**SSH into the Globo Caller VM:**

```bash
gcloud compute ssh globo-srt-source \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --tunnel-through-iap
```

Inside the VM, run:

```bash
sudo apt-get update && sudo apt-get install -y srt-tools ffmpeg
exit
```

---

## Part 4 — Set Up Private Service Connect

> **What we're achieving:** This is the core of the lab. We create the PSC plumbing that connects the two isolated VPCs. In the real Globo setup, this is exactly what Globo (customer) and Harmonic (LRS) would do — Globo creates a Network Attachment in their VPC, and Harmonic attaches a PSC Interface to the LRS VM.

### Step 11: Create the Network Attachment (Customer/Globo side)

> **What:** A Network Attachment is the customer's side of PSC. It's a resource in the customer's subnet that says: "I allow project X to create a PSC interface that connects into my network." `ACCEPT_MANUAL` means only explicitly approved projects can connect — this is the security control that makes PSC better than VPC peering.
>
> **In the real setup:** Globo creates this in their VPC and adds Harmonic's LRS project ID to the accepted list. Here, both VPCs are in the same project, so we accept our own project ID.

```bash
gcloud compute network-attachments create globo-psc-attachment \
  --project=$PROJECT_ID \
  --region=$REGION \
  --subnets=globo-subnet \
  --connection-preference=ACCEPT_MANUAL \
  --producer-accept-list=$PROJECT_ID
```

### Step 12: Get the Network Attachment URL

> **What:** Every Network Attachment has a unique URL. The LRS side needs this URL to create a PSC Interface. In the real setup, Globo would send this URL to Harmonic.

```bash
gcloud compute network-attachments describe globo-psc-attachment \
  --project=$PROJECT_ID \
  --region=$REGION \
  --format="value(selfLink)"
```

Save the output — it looks like:
```
projects/srt-sources/regions/us-west1/networkAttachments/globo-psc-attachment
```

### Step 13: Stop the LRS VM and add the PSC Interface

> **What:** This is the Harmonic/LRS side of PSC. We add a **second network interface (PSC Interface)** to the LRS VM. This interface connects to the customer's Network Attachment, and GCP assigns it an IP address from the **customer's subnet** (10.100.0.0/24). After this, the LRS VM has two NICs:
>
> - **NIC 1 (primary):** Connected to the LRS VPC (10.200.0.x)
> - **NIC 2 (PSC):** Connected to the Globo VPC via Network Attachment (10.100.0.x)
>
> The VM must be stopped to add a new network interface — this is a GCP requirement.

```bash
# Stop the LRS Listener VM
gcloud compute instances stop lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE

# Add the PSC interface — this creates NIC 2 connected to Globo's VPC
gcloud compute instances add-network-interface lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --network-attachment=projects/$PROJECT_ID/regions/$REGION/networkAttachments/globo-psc-attachment

# Start the VM
gcloud compute instances start lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE
```

### Step 14: Get the PSC Interface IP

> **What:** Find the IP that GCP assigned to the PSC Interface. This IP is from the **Globo subnet** (10.100.0.x), which means it's reachable from VMs in the Globo VPC. This is the IP that Globo's SRT sources will connect to.

```bash
gcloud compute instances describe lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --format="json(networkInterfaces)"
```

In the output, look for the network interface that has a `networkAttachment` field. The `networkIP` is the PSC IP. Example: `10.100.0.2`

```bash
export PSC_IP="10.100.0.2"   # ← replace with the actual IP from the output above
```

---

## Part 5 — Configure SRT Firewall Rules

> **What we're achieving:** SRT uses bidirectional UDP. The Caller (Globo) initiates the connection, but SRT's handshake, ACKs, NAKs, and retransmitted packets flow in both directions. We need firewall rules allowing UDP port 7086 both out (to the PSC IP) and back in (from the PSC IP).

### Step 15: Allow SRT traffic in the Customer VPC

```bash
# Egress: allow Globo VM to send SRT packets to the PSC IP
gcloud compute firewall-rules create globo-allow-srt-egress \
  --project=$PROJECT_ID \
  --network=globo-vpc \
  --direction=EGRESS \
  --allow=udp:7086 \
  --destination-ranges=$PSC_IP/32 \
  --description="Allow SRT egress to LRS via PSC"

# Ingress: allow SRT return traffic (ACKs, NAKs, retransmits) from the PSC IP
gcloud compute firewall-rules create globo-allow-srt-ingress \
  --project=$PROJECT_ID \
  --network=globo-vpc \
  --direction=INGRESS \
  --allow=udp:7086 \
  --source-ranges=$PSC_IP/32 \
  --description="Allow SRT return traffic from LRS via PSC"
```

---

## Part 6 — Stream and Verify

> **What we're achieving:** The moment of truth — send an SRT stream from the Globo VM (Caller) through the PSC connection to the LRS VM (Listener). If this works, it proves that SRT over PSC is viable and we can use the same pattern for the real Globo deployment.

### Step 16: Start the SRT Listener on the LRS VM

> **What:** Start an SRT Listener on the LRS VM. It must bind to `0.0.0.0` (all interfaces) — not just the primary NIC IP — because SRT traffic arrives on the PSC interface (NIC 2).

SSH into the LRS VM:

```bash
gcloud compute ssh lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --tunnel-through-iap
```

Inside the VM, start the listener:

```bash
srt-live-transmit "srt://0.0.0.0:7086?mode=listener" "file:///dev/null" -v
```

> You should see it waiting for a connection. **Leave this running.** Open a **second Cloud Shell tab** for the next step.

### Step 17: Start the SRT Caller on the Globo VM

> **What:** Send a test video stream from the Globo VM to the LRS Listener via PSC. `ffmpeg` generates a colour bar test pattern with a 1kHz tone, encodes it as H.264/AAC in MPEG-TS, and sends it over SRT in Caller mode.

In the **second Cloud Shell tab**, set variables again (new shell session):

```bash
export PROJECT_ID="srt-sources"
export ZONE="us-west1-a"
export PSC_IP="10.100.0.2"   # ← use your actual PSC IP
```

SSH into the Globo VM:

```bash
gcloud compute ssh globo-srt-source \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --tunnel-through-iap
```

Inside the VM, send the test stream (replace the IP if different):

```bash
ffmpeg -re -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=48000 \
  -c:v libx264 -preset ultrafast -b:v 2M \
  -c:a aac -b:a 128k \
  -f mpegts "srt://10.100.0.2:7086?mode=caller"
```

### Step 18: Verify — What Success Looks Like

**Terminal 1 (LRS Listener)** should show:
```
SRT connected
...receiving data bytes...
```

**Terminal 2 (Globo Caller / ffmpeg)** should show:
```
frame=  120 fps= 30 q=20.0 size=    1024kB time=00:00:04.00 bitrate=2048.0kbits/s
```

If both terminals show this — **PSC + SRT is validated!**

### Troubleshooting

If the connection doesn't establish:

```bash
# From the Globo VM — check if PSC IP is reachable
ping -c 3 10.100.0.2

# Check UDP port
nc -zuv 10.100.0.2 7086

# On the LRS VM — verify the PSC interface exists (should see two NICs)
ip addr show
# Expected:
#   ens4: 10.200.0.x  ← primary NIC (LRS VPC)
#   ens5: 10.100.0.x  ← PSC interface (Globo VPC)
```

**Common issues:**

| Problem | Cause | Fix |
|---------|-------|-----|
| Connection timeout | Firewall rules missing | Check both VPCs have the SRT rules (Part 5) |
| Connection refused | Listener not running | Make sure `srt-live-transmit` is running on LRS VM |
| Listener doesn't see traffic | Bound to wrong interface | Must use `0.0.0.0`, not `10.200.0.x` |
| PSC Interface not created | Network Attachment rejected | Check: `gcloud compute network-attachments describe globo-psc-attachment --region=$REGION` |

---

## What This Proves

| What | Validated | Real-World Equivalent |
|------|-----------|----------------------|
| Network Attachment creation | ✓ | Globo creates this in their VPC |
| PSC Interface provisioning | ✓ | Harmonic creates this on the LRS VM |
| PSC gives LRS VM an IP in customer's subnet | ✓ | LRS becomes reachable from Globo's network |
| SRT Caller → PSC → SRT Listener | ✓ | Globo encoder → PSC → LRS Cloud Source |
| Bidirectional UDP flows through PSC | ✓ | SRT handshake + ARQ works over PSC |
| Firewall rules for SRT over PSC | ✓ | Globo configures these in their VPC |

**What's different in production:**
- The "LRS" VPC is in Harmonic's project (different from Globo's project)
- LRS Cloud Source handles the SRT Listener automatically (no need to run `srt-live-transmit` manually)
- LRS connects to VOS 360 via VPC Peering (Harmonic's internal setup — not part of PSC)
- The customer (Globo) side — Network Attachment, firewall rules, SRT Caller config — is **identical** to what we tested here

---

## Cleanup

> **Important:** Run this when you're done testing to stop incurring charges. The VMs and Cloud NAT cost ~$0.16/hr total.

```bash
export PROJECT_ID="srt-sources"
export REGION="us-west1"
export ZONE="us-west1-a"

# Delete VMs
gcloud compute instances delete globo-srt-source lrs-srt-listener \
  --project=$PROJECT_ID --zone=$ZONE --quiet

# Delete firewall rules
gcloud compute firewall-rules delete \
  globo-allow-iap-ssh globo-allow-internal globo-allow-srt-egress globo-allow-srt-ingress \
  lrs-allow-iap-ssh lrs-allow-internal lrs-allow-srt-from-psc \
  --project=$PROJECT_ID --quiet

# Delete Cloud NAT and Routers
gcloud compute routers nats delete globo-nat --router=globo-router \
  --project=$PROJECT_ID --region=$REGION --quiet
gcloud compute routers delete globo-router \
  --project=$PROJECT_ID --region=$REGION --quiet
gcloud compute routers nats delete lrs-nat --router=lrs-router \
  --project=$PROJECT_ID --region=$REGION --quiet
gcloud compute routers delete lrs-router \
  --project=$PROJECT_ID --region=$REGION --quiet

# Delete Network Attachment
gcloud compute network-attachments delete globo-psc-attachment \
  --project=$PROJECT_ID --region=$REGION --quiet

# Delete subnets
gcloud compute networks subnets delete globo-subnet lrs-subnet \
  --project=$PROJECT_ID --region=$REGION --quiet

# Delete VPCs
gcloud compute networks delete globo-vpc lrs-vpc \
  --project=$PROJECT_ID --quiet
```

After cleanup, verify nothing is left:

```bash
gcloud compute instances list --project=$PROJECT_ID
gcloud compute networks list --project=$PROJECT_ID
```

You should only see the `default` network (if it exists) and no VM instances.
