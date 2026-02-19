# PSC Lab Guide — Cross-Project SRT-over-PSC Simulation

## What Are We Building?

We're replicating the **real Globo ↔ LRS** Private Service Connect setup using **two separate GCP projects** — exactly how it works in production. One project simulates Globo (customer), the other simulates Harmonic LRS (producer).

```
  PROJECT 1: project-ea255408-241f-4c48-8e8                     PROJECT 2: lrs-harmonic
  (Globo — Customer / Consumer)              (Harmonic — LRS / Producer)

┌──────────────────────────┐              ┌──────────────────────────┐
│  Globo VPC               │     PSC      │  LRS VPC                 │
│  10.100.0.0/24           │◄────────────►│  10.200.0.0/24           │
│                          │              │                          │
│  ┌────────────────────┐  │              │  ┌────────────────────┐  │
│  │ globo-srt-source   │──┼── SRT ──────►│  │ lrs-srt-listener   │  │
│  │ (Caller :7086)     │  │              │  │ (Listener :7086)   │  │
│  └────────────────────┘  │              │  └────────────────────┘  │
│                          │              │                          │
│  Network Attachment      │              │  PSC Interface           │
│  (Consumer — controls    │              │  (Producer — gets IP     │
│   who can connect)       │              │   from Globo's subnet)   │
└──────────────────────────┘              └──────────────────────────┘
```

**Why two projects?** This is how PSC works in the real world — the customer and Harmonic are in **different GCP projects**. The customer controls access via the Network Attachment, and Harmonic connects via the PSC Interface. Using two projects validates the cross-project trust model (project ID allowlisting).

---

## Prerequisites

- GCP account with free trial credits
- Cloud Shell access
- Both projects under the same billing account

---

## Setup — Create Two Projects

> **What:** We need two separate GCP projects to simulate the two organisations. Both projects share your billing account so the free credits cover everything.

### Step 1: Set the Globo project

Your default project `project-ea255408-241f-4c48-8e8` ("My First Project") will be the Globo (customer) project.

```bash
gcloud config set project project-ea255408-241f-4c48-8e8
```

### Step 2: Create the LRS project (Harmonic)

> **What:** Create a second project to simulate Harmonic's LRS environment. This is a completely separate project with its own VPC, VMs, and firewall rules — just like in production.

```bash
gcloud projects create lrs-harmonic --name="LRS Harmonic"
```

### Step 3: Link the LRS project to your billing account

> **What:** The new project needs a billing account to create resources. Link it to the same billing account as `srt-sources` so your free credits cover both.

```bash
# Find your billing account ID
gcloud billing accounts list

# Link it (replace YOUR_BILLING_ACCOUNT_ID with the ID from above)
gcloud billing projects link lrs-harmonic \
  --billing-account=YOUR_BILLING_ACCOUNT_ID
```

### Step 4: Enable Compute Engine API in both projects

> **What:** The Compute Engine API must be enabled before creating VPCs, VMs, or firewall rules.

```bash
gcloud services enable compute.googleapis.com --project=project-ea255408-241f-4c48-8e8
gcloud services enable compute.googleapis.com --project=lrs-harmonic
```

### Step 5: Set environment variables

> **What:** Set shell variables for both projects. We'll reference `$GLOBO_PROJECT` and `$LRS_PROJECT` throughout the guide to make it clear which project each command targets.

```bash
export GLOBO_PROJECT="project-ea255408-241f-4c48-8e8"
export LRS_PROJECT="lrs-harmonic"
export REGION="us-west1"
export ZONE="us-west1-a"
```

---

## Part 1 — Create the Globo VPC (Customer / Consumer)

> **What we're achieving:** Building Globo's network environment in project `project-ea255408-241f-4c48-8e8`. In the real world, Globo already has a VPC with their SRT encoders connected via Google Direct Connect from their headend. Here we create a VPC and a VM to simulate that.

### Step 6: Create the Globo VPC and subnet

> **What:** A VPC is an isolated network. The subnet defines the IP range and region. This subnet is where:
> - The Globo SRT source VM will live
> - The **Network Attachment** will be created (PSC consumer side)
> - The PSC Interface will get its IP from (making LRS reachable from Globo's network)

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

### Step 7: Create firewall rules for Globo VPC

> **What:** GCP VPCs block all traffic by default. We need:
> - SSH via IAP (so we can log into the VM without a public IP)
> - Internal traffic (so VMs in the subnet can talk to each other and to the PSC Interface IP)

```bash
# Allow SSH via IAP (35.235.240.0/20 = Google's IAP tunnel IP range)
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

### Step 8: Create the Globo SRT Source VM

> **What:** This VM simulates Globo's SRT encoder. It runs in the Globo VPC with **no public IP** (`no-address`), just like a real encoder behind Google Direct Connect. We'll use `ffmpeg` to generate a test stream and send it as an **SRT Caller** to the LRS Listener.

```bash
gcloud compute instances create globo-srt-source \
  --project=$GLOBO_PROJECT \
  --zone=$ZONE \
  --machine-type=e2-medium \
  --network-interface=network=globo-vpc,subnet=globo-subnet,no-address \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB
```

### Step 9: Add Cloud NAT for Globo VPC

> **What:** The VM has no public IP, so it can't reach the internet to install packages. Cloud NAT provides outbound internet access (for `apt-get install`) without requiring a public IP on the VM.

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

### Step 10: Install SRT tools on the Globo VM

> **What:** Install `ffmpeg` (to generate a test video stream) and `srt-tools` (provides `srt-live-transmit` for SRT streaming).

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

## Part 2 — Create the LRS VPC (Harmonic / Producer)

> **What we're achieving:** Building Harmonic's LRS environment in a **separate project** (`lrs-harmonic`). This is a completely different GCP project — the two projects have no network connectivity by default. The only way they'll communicate is through PSC. This mirrors the real-world separation between Globo and Harmonic.

### Step 11: Create the LRS VPC and subnet

> **What:** The LRS VM's primary network interface sits in this VPC/subnet. This is Harmonic's internal network — the customer (Globo) has no visibility into it.

```bash
gcloud compute networks create lrs-vpc \
  --project=$LRS_PROJECT \
  --subnet-mode=custom

gcloud compute networks subnets create lrs-subnet \
  --project=$LRS_PROJECT \
  --network=lrs-vpc \
  --region=$REGION \
  --range=10.200.0.0/24
```

### Step 12: Create firewall rules for LRS VPC

> **What:** Same pattern — SSH via IAP, internal traffic, plus one critical rule: allow **SRT traffic (UDP 7086) arriving from the customer's subnet** via the PSC Interface. The PSC Interface gets an IP from Globo's subnet (10.100.0.0/24), so SRT packets arrive with source IPs from that range.

```bash
# Allow SSH via IAP
gcloud compute firewall-rules create lrs-allow-iap-ssh \
  --project=$LRS_PROJECT \
  --network=lrs-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --description="Allow SSH via IAP"

# Allow all internal traffic within LRS VPC
gcloud compute firewall-rules create lrs-allow-internal \
  --project=$LRS_PROJECT \
  --network=lrs-vpc \
  --allow=all \
  --source-ranges=10.200.0.0/24 \
  --description="Allow internal VPC traffic"

# Allow SRT traffic arriving via PSC from customer's subnet
gcloud compute firewall-rules create lrs-allow-srt-from-psc \
  --project=$LRS_PROJECT \
  --network=lrs-vpc \
  --allow=udp:7086 \
  --source-ranges=10.100.0.0/24 \
  --description="Allow SRT from customer VPC via PSC"
```

### Step 13: Create the LRS SRT Listener VM

> **What:** This VM simulates the **LRS Cloud Source**. It runs `srt-live-transmit` in **Listener mode** on port 7086 — accepting incoming SRT connections from Globo's encoders. It has no public IP.

```bash
gcloud compute instances create lrs-srt-listener \
  --project=$LRS_PROJECT \
  --zone=$ZONE \
  --machine-type=e2-medium \
  --network-interface=network=lrs-vpc,subnet=lrs-subnet,no-address \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB
```

### Step 14: Add Cloud NAT for LRS VPC

> **What:** Same as Globo — the LRS VM needs outbound internet to install packages.

```bash
gcloud compute routers create lrs-router \
  --project=$LRS_PROJECT \
  --network=lrs-vpc \
  --region=$REGION

gcloud compute routers nats create lrs-nat \
  --project=$LRS_PROJECT \
  --router=lrs-router \
  --region=$REGION \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

### Step 15: Install SRT tools on the LRS VM

```bash
gcloud compute ssh lrs-srt-listener \
  --project=$LRS_PROJECT \
  --zone=$ZONE \
  --tunnel-through-iap
```

Inside the VM:

```bash
sudo apt-get update && sudo apt-get install -y srt-tools
exit
```

---

## Part 3 — Set Up Private Service Connect

> **What we're achieving:** This is the core of the lab — connecting two separate GCP projects via PSC. Globo (Consumer) creates a Network Attachment in their VPC. Harmonic (Producer) adds a PSC Interface to the LRS VM, connecting it to Globo's Network Attachment. The LRS VM then gets an IP in Globo's subnet — making it reachable from Globo's SRT sources.

### Step 16: Create the Network Attachment (Globo — Consumer side)

> **What:** The Network Attachment is Globo's side of PSC. It says: "I allow project `lrs-harmonic` to create a PSC Interface into my subnet." The `ACCEPT_MANUAL` flag means only explicitly listed projects can connect — this is the access control that replaces VPC peering.
>
> **In the real setup:** Globo creates this and adds Harmonic's LRS project ID to the `--producer-accept-list`. Here we add `lrs-harmonic`.

```bash
gcloud compute network-attachments create globo-psc-attachment \
  --project=$GLOBO_PROJECT \
  --region=$REGION \
  --subnets=globo-subnet \
  --connection-preference=ACCEPT_MANUAL \
  --producer-accept-list=$LRS_PROJECT
```

### Step 17: Get the Network Attachment URL

> **What:** The Network Attachment has a unique URL that Harmonic needs to create the PSC Interface. In the real setup, Globo sends this URL to Harmonic. Here we retrieve it ourselves.

```bash
gcloud compute network-attachments describe globo-psc-attachment \
  --project=$GLOBO_PROJECT \
  --region=$REGION \
  --format="value(selfLink)"
```

Save the output — it looks like:
```
projects/project-ea255408-241f-4c48-8e8/regions/us-west1/networkAttachments/globo-psc-attachment
```

### Step 18: Stop the LRS VM and add the PSC Interface (Harmonic — Producer side)

> **What:** This is Harmonic's side of PSC. We add a **second network interface (PSC Interface)** to the LRS VM in the `lrs-harmonic` project. This interface connects to Globo's Network Attachment in the `srt-sources` project. GCP assigns it an IP from **Globo's subnet** (10.100.0.0/24).
>
> After this, the LRS VM has **two NICs**:
> - **NIC 1 (primary):** In the LRS VPC (10.200.0.x) — Harmonic's internal network
> - **NIC 2 (PSC):** In the Globo VPC (10.100.0.x) — reachable from Globo's SRT sources
>
> The VM must be stopped to add a network interface — GCP requirement.

```bash
# Stop the LRS VM
gcloud compute instances stop lrs-srt-listener \
  --project=$LRS_PROJECT \
  --zone=$ZONE

# Add PSC Interface — connects to Globo's Network Attachment (cross-project!)
gcloud compute instances add-network-interface lrs-srt-listener \
  --project=$LRS_PROJECT \
  --zone=$ZONE \
  --network-attachment=projects/$GLOBO_PROJECT/regions/$REGION/networkAttachments/globo-psc-attachment

# Start the VM
gcloud compute instances start lrs-srt-listener \
  --project=$LRS_PROJECT \
  --zone=$ZONE
```

> Note the cross-project reference: the LRS VM in `lrs-harmonic` is connecting to a Network Attachment in `srt-sources`. This only works because Globo added `lrs-harmonic` to the `--producer-accept-list`.

### Step 19: Get the PSC Interface IP

> **What:** Find the IP that GCP assigned to the PSC Interface. This IP comes from **Globo's subnet** (10.100.0.x), making the LRS VM reachable from Globo's VPC. This is the IP that Globo's SRT sources connect to.

```bash
gcloud compute instances describe lrs-srt-listener \
  --project=$LRS_PROJECT \
  --zone=$ZONE \
  --format="json(networkInterfaces)"
```

In the output, find the network interface with a `networkAttachment` field. The `networkIP` is the PSC IP (e.g. `10.100.0.2`).

```bash
export PSC_IP="10.100.0.2"   # ← replace with actual IP from output
```

---

## Part 4 — Configure SRT Firewall Rules

> **What we're achieving:** SRT uses **bidirectional UDP**. The Caller (Globo) initiates the connection, but SRT's handshake, ACKs, NAKs, and retransmitted packets flow in both directions. We need firewall rules in the **Globo VPC** allowing UDP 7086 both out to the PSC IP and back in from the PSC IP.

### Step 20: Allow SRT traffic in the Globo VPC

```bash
# Egress: allow Globo VM to send SRT packets to the PSC IP
gcloud compute firewall-rules create globo-allow-srt-egress \
  --project=$GLOBO_PROJECT \
  --network=globo-vpc \
  --direction=EGRESS \
  --allow=udp:7086 \
  --destination-ranges=$PSC_IP/32 \
  --description="Allow SRT egress to LRS via PSC"

# Ingress: allow SRT return traffic (ACKs, NAKs, retransmits) from the PSC IP
gcloud compute firewall-rules create globo-allow-srt-ingress \
  --project=$GLOBO_PROJECT \
  --network=globo-vpc \
  --direction=INGRESS \
  --allow=udp:7086 \
  --source-ranges=$PSC_IP/32 \
  --description="Allow SRT return traffic from LRS via PSC"
```

---

## Part 5 — Stream and Verify

> **What we're achieving:** The moment of truth — send an SRT stream from the Globo VM (Caller, project `project-ea255408-241f-4c48-8e8`) through PSC to the LRS VM (Listener, project `lrs-harmonic`). If this works, we've proven that **SRT over cross-project PSC works** and the same setup applies to the real Globo deployment.

### Step 21: Start the SRT Listener on the LRS VM

> **What:** Start an SRT Listener on the LRS VM. It must bind to `0.0.0.0` (all interfaces) so it accepts traffic on the PSC interface (NIC 2), not just the primary NIC.

SSH into the LRS VM (in the LRS project):

```bash
gcloud compute ssh lrs-srt-listener \
  --project=$LRS_PROJECT \
  --zone=$ZONE \
  --tunnel-through-iap
```

Inside the VM, start the listener:

```bash
srt-live-transmit "srt://0.0.0.0:7086?mode=listener" "file:///dev/null" -v
```

> You should see it waiting for a connection. **Leave this running.** Open a **second Cloud Shell tab** for the next step.

### Step 22: Start the SRT Caller on the Globo VM

> **What:** Send a test video stream from the Globo VM to the LRS Listener via PSC. `ffmpeg` generates a colour bar test pattern with a 1kHz tone, encodes it as H.264/AAC in MPEG-TS, and sends it over SRT in **Caller mode** to the PSC Interface IP.

In the **second Cloud Shell tab**, set variables (new shell session):

```bash
export GLOBO_PROJECT="project-ea255408-241f-4c48-8e8"
export ZONE="us-west1-a"
export PSC_IP="10.100.0.2"   # ← use your actual PSC IP
```

SSH into the Globo VM (in the Globo project):

```bash
gcloud compute ssh globo-srt-source \
  --project=$GLOBO_PROJECT \
  --zone=$ZONE \
  --tunnel-through-iap
```

Inside the VM, send the test stream (replace IP if different):

```bash
ffmpeg -re -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=48000 \
  -c:v libx264 -preset ultrafast -b:v 2M \
  -c:a aac -b:a 128k \
  -f mpegts "srt://10.100.0.2:7086?mode=caller"
```

### Step 23: Verify — What Success Looks Like

**Terminal 1 — LRS Listener** (project: `lrs-harmonic`) should show:
```
SRT connected
...receiving data bytes...
```

**Terminal 2 — Globo Caller / ffmpeg** (project: `project-ea255408-241f-4c48-8e8`) should show:
```
frame=  120 fps= 30 q=20.0 size=    1024kB time=00:00:04.00 bitrate=2048.0kbits/s
```

If both terminals show this — **cross-project PSC + SRT is validated!**

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
#   ens4: 10.200.0.x  ← NIC 1: primary, LRS VPC (lrs-harmonic project)
#   ens5: 10.100.0.x  ← NIC 2: PSC interface, Globo VPC (srt-sources project)
```

**Common issues:**

| Problem | Cause | Fix |
|---------|-------|-----|
| `add-network-interface` fails | Network Attachment not accepting LRS project | Check `--producer-accept-list` includes `lrs-harmonic` |
| Connection timeout | Firewall rules missing | Check Globo VPC (Part 4) and LRS VPC (Step 12) rules |
| Connection refused | Listener not running | Make sure `srt-live-transmit` is running on LRS VM |
| Listener doesn't see traffic | Bound to wrong interface | Must use `0.0.0.0`, not `10.200.0.x` |
| PSC Interface no IP | Network Attachment status | `gcloud compute network-attachments describe globo-psc-attachment --project=$GLOBO_PROJECT --region=$REGION` |

---

## What This Proves

| What | Validated | Maps to Real Setup |
|------|-----------|-------------------|
| Cross-project PSC works | ✓ | Globo project ↔ Harmonic LRS project |
| Network Attachment controls access | ✓ | Globo chooses which project can connect |
| PSC Interface gives LRS an IP in customer subnet | ✓ | LRS reachable from Globo's VPC |
| SRT Caller → PSC → SRT Listener | ✓ | Globo encoder → PSC → LRS Cloud Source |
| Bidirectional UDP (SRT handshake + data) | ✓ | SRT ARQ recovery works over PSC |
| Cross-project firewall rules | ✓ | Each side manages their own firewall |

**What's different in production:**
- Harmonic's LRS project is managed by Harmonic (you won't create it — it already exists)
- LRS Cloud Source handles the SRT Listener automatically (no manual `srt-live-transmit`)
- LRS connects to VOS 360 via **VPC Peering** (Harmonic internal — not part of PSC)
- The customer side (Network Attachment, firewall rules, SRT Caller config) is **identical** to what we tested

---

## Cleanup

> **Important:** Run this when done testing to stop charges. VMs + Cloud NAT cost ~$0.16/hr total across both projects.

### Cleanup Globo project

```bash
export GLOBO_PROJECT="project-ea255408-241f-4c48-8e8"
export REGION="us-west1"
export ZONE="us-west1-a"

# Delete VM
gcloud compute instances delete globo-srt-source \
  --project=$GLOBO_PROJECT --zone=$ZONE --quiet

# Delete firewall rules
gcloud compute firewall-rules delete \
  globo-allow-iap-ssh globo-allow-internal globo-allow-srt-egress globo-allow-srt-ingress \
  --project=$GLOBO_PROJECT --quiet

# Delete Cloud NAT and Router
gcloud compute routers nats delete globo-nat --router=globo-router \
  --project=$GLOBO_PROJECT --region=$REGION --quiet
gcloud compute routers delete globo-router \
  --project=$GLOBO_PROJECT --region=$REGION --quiet

# Delete Network Attachment
gcloud compute network-attachments delete globo-psc-attachment \
  --project=$GLOBO_PROJECT --region=$REGION --quiet

# Delete subnet and VPC
gcloud compute networks subnets delete globo-subnet \
  --project=$GLOBO_PROJECT --region=$REGION --quiet
gcloud compute networks delete globo-vpc \
  --project=$GLOBO_PROJECT --quiet
```

### Cleanup LRS project (lrs-harmonic)

```bash
export LRS_PROJECT="lrs-harmonic"

# Delete VM (PSC interface is removed when VM is deleted)
gcloud compute instances delete lrs-srt-listener \
  --project=$LRS_PROJECT --zone=$ZONE --quiet

# Delete firewall rules
gcloud compute firewall-rules delete \
  lrs-allow-iap-ssh lrs-allow-internal lrs-allow-srt-from-psc \
  --project=$LRS_PROJECT --quiet

# Delete Cloud NAT and Router
gcloud compute routers nats delete lrs-nat --router=lrs-router \
  --project=$LRS_PROJECT --region=$REGION --quiet
gcloud compute routers delete lrs-router \
  --project=$LRS_PROJECT --region=$REGION --quiet

# Delete subnet and VPC
gcloud compute networks subnets delete lrs-subnet \
  --project=$LRS_PROJECT --region=$REGION --quiet
gcloud compute networks delete lrs-vpc \
  --project=$LRS_PROJECT --quiet

# Optionally delete the entire LRS project
# gcloud projects delete lrs-harmonic --quiet
```

### Verify cleanup

```bash
gcloud compute instances list --project=project-ea255408-241f-4c48-8e8
gcloud compute instances list --project=lrs-harmonic
gcloud compute networks list --project=project-ea255408-241f-4c48-8e8
gcloud compute networks list --project=lrs-harmonic
```

No VMs, only `default` networks (if any) should remain.
