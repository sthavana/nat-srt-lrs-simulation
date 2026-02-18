# PSC Lab Guide — Simulating Customer SRT-to-LRS via Private Service Connect

## Objective

Simulate the Globo customer environment by creating a separate GCP VPC, launching a VM with SRT, and connecting it to your existing LRS Cloud Source via Private Service Connect — before doing it with the real customer.

---

## Prerequisites

- [x] VOS360 cluster deployed in GCP (VOS360 VPC exists)
- [x] LRS Cloud Source created via Syndicate (sits in the **LRS VPC** — separate from VOS360 VPC, connected via VPC Peering)
- [ ] `gcloud` CLI authenticated with sufficient permissions
- [ ] Your **LRS VPC** project ID and region (you'll need these throughout — this is NOT the VOS360 project)

```bash
# Set these once — used in all commands below
export LRS_PROJECT_ID="your-lrs-project-id"       # LRS VPC project (NOT the VOS360 project)
export LRS_REGION="us-central1"           # region where LRS is deployed
export LRS_ZONE="us-central1-a"
export CUSTOMER_PROJECT_ID="your-test-project-id"   # can be same project or different
export CUSTOMER_REGION="us-central1"      # MUST be same region as LRS
export CUSTOMER_ZONE="us-central1-a"
```

> **Important:** The customer VPC and LRS VPC must be in the **same GCP region** for PSC to work.

---

## Part 1 — Create the "Customer" VPC (Simulating Globo)

### Step 1: Create the VPC

```bash
gcloud compute networks create globo-test-vpc \
  --project=$CUSTOMER_PROJECT_ID \
  --subnet-mode=custom
```

### Step 2: Create a subnet

```bash
gcloud compute networks subnets create globo-test-subnet \
  --project=$CUSTOMER_PROJECT_ID \
  --network=globo-test-vpc \
  --region=$CUSTOMER_REGION \
  --range=10.100.0.0/24
```

### Step 3: Create firewall rules

Allow SSH (for you to access the VM) and allow all internal traffic within the VPC:

```bash
# Allow SSH from your IP (or use IAP)
gcloud compute firewall-rules create globo-allow-ssh \
  --project=$CUSTOMER_PROJECT_ID \
  --network=globo-test-vpc \
  --allow=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --description="Allow SSH access"

# Allow all internal traffic within the VPC
gcloud compute firewall-rules create globo-allow-internal \
  --project=$CUSTOMER_PROJECT_ID \
  --network=globo-test-vpc \
  --allow=all \
  --source-ranges=10.100.0.0/24 \
  --description="Allow internal VPC traffic"
```

---

## Part 2 — Launch the SRT Source VM

### Step 4: Create the VM

```bash
gcloud compute instances create globo-srt-source \
  --project=$CUSTOMER_PROJECT_ID \
  --zone=$CUSTOMER_ZONE \
  --machine-type=e2-medium \
  --network-interface=network=globo-test-vpc,subnet=globo-test-subnet,no-address \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install -y ffmpeg srt-tools'
```

> `--no-address` means no external IP — this simulates the customer scenario where traffic goes via PSC, not the public internet. Use IAP for SSH access (see below).

### Step 5: SSH into the VM via IAP

Since the VM has no external IP:

```bash
gcloud compute ssh globo-srt-source \
  --project=$CUSTOMER_PROJECT_ID \
  --zone=$CUSTOMER_ZONE \
  --tunnel-through-iap
```

> If IAP SSH isn't set up, you can temporarily add an external IP, or add a Cloud NAT for outbound (needed for apt-get anyway — see Step 5b).

### Step 5b (if needed): Add Cloud NAT for outbound internet

The VM needs outbound internet to install packages (apt-get). If you used `--no-address`:

```bash
# Create a Cloud Router
gcloud compute routers create globo-router \
  --project=$CUSTOMER_PROJECT_ID \
  --network=globo-test-vpc \
  --region=$CUSTOMER_REGION

# Create Cloud NAT
gcloud compute routers nats create globo-nat \
  --project=$CUSTOMER_PROJECT_ID \
  --router=globo-router \
  --region=$CUSTOMER_REGION \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

### Step 6: Verify SRT tools are installed

SSH into the VM and check:

```bash
ffmpeg -version | head -1
srt-live-transmit --version
```

If `srt-tools` didn't install via startup script:

```bash
sudo apt-get update
sudo apt-get install -y ffmpeg srt-tools
```

### Step 7: Prepare a test stream

Generate a test pattern SRT stream (don't run yet — we need the PSC IP first):

```bash
# Option A: ffmpeg test pattern → SRT Caller
ffmpeg -re -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=48000 \
  -c:v libx264 -preset ultrafast -b:v 2M \
  -c:a aac -b:a 128k \
  -f mpegts "srt://PSC_IP:7086?mode=caller"

# Option B: srt-live-transmit with test file
srt-live-transmit "file://test.ts" "srt://PSC_IP:7086?mode=caller" -v
```

---

## Part 3 — Set Up Private Service Connect

### Step 8: Create the Network Attachment (Customer side)

This is done in the "customer" (Globo test) VPC:

```bash
gcloud compute network-attachments create globo-psc-attachment \
  --project=$CUSTOMER_PROJECT_ID \
  --region=$CUSTOMER_REGION \
  --subnets=globo-test-subnet \
  --connection-preference=ACCEPT_MANUAL \
  --producer-accept-list=$LRS_PROJECT_ID
```

> `--producer-accept-list` allows only the LRS project to connect to this Network Attachment.

### Step 9: Get the Network Attachment URL

```bash
gcloud compute network-attachments describe globo-psc-attachment \
  --project=$CUSTOMER_PROJECT_ID \
  --region=$CUSTOMER_REGION \
  --format="value(selfLink)"
```

Save the output — you'll need it for the next step. It looks like:

```
projects/CUSTOMER_PROJECT_ID/regions/us-central1/networkAttachments/globo-psc-attachment
```

### Step 10: Create the PSC Interface on the LRS VM (Harmonic/Producer side)

> **This step is done in the LRS project.** You need to know the LRS VM instance name.

First, find the LRS Cloud Source VM:

```bash
# List VMs in the LRS project to find the Cloud Source instance
gcloud compute instances list \
  --project=$LRS_PROJECT_ID \
  --filter="zone:$LRS_ZONE"
```

Identify the LRS Cloud Source VM name (e.g. `lrs-cloud-source-vm`), then add a PSC Interface to it:

```bash
export LRS_VM_NAME="lrs-cloud-source-vm"   # replace with actual name

# Stop the VM first (PSC interface can only be added when stopped)
gcloud compute instances stop $LRS_VM_NAME \
  --project=$LRS_PROJECT_ID \
  --zone=$LRS_ZONE

# Add the PSC network interface
gcloud compute instances add-network-interface $LRS_VM_NAME \
  --project=$LRS_PROJECT_ID \
  --zone=$LRS_ZONE \
  --network-attachment=projects/$CUSTOMER_PROJECT_ID/regions/$CUSTOMER_REGION/networkAttachments/globo-psc-attachment

# Start the VM
gcloud compute instances start $LRS_VM_NAME \
  --project=$LRS_PROJECT_ID \
  --zone=$LRS_ZONE
```

### Step 11: Get the PSC Interface IP

```bash
gcloud compute instances describe $LRS_VM_NAME \
  --project=$LRS_PROJECT_ID \
  --zone=$LRS_ZONE \
  --format="json(networkInterfaces)" | grep -A5 "networkAttachment"
```

Look for the `networkIP` field — this is the PSC IP in the customer's subnet (e.g. `10.100.0.2`). Save this as `PSC_IP`.

```bash
export PSC_IP="10.100.0.2"   # replace with actual IP from output
```

---

## Part 4 — Configure Firewall Rules for SRT

### Step 12: Allow SRT traffic through the customer VPC

SRT uses **bidirectional UDP** — the customer initiates (Caller) but SRT handshake and data flow both ways:

```bash
# Egress: allow VM to send SRT to PSC IP
gcloud compute firewall-rules create globo-allow-srt-egress \
  --project=$CUSTOMER_PROJECT_ID \
  --network=globo-test-vpc \
  --direction=EGRESS \
  --allow=udp:7086 \
  --destination-ranges=$PSC_IP/32 \
  --description="Allow SRT egress to LRS via PSC"

# Ingress: allow SRT return traffic from PSC IP
gcloud compute firewall-rules create globo-allow-srt-ingress \
  --project=$CUSTOMER_PROJECT_ID \
  --network=globo-test-vpc \
  --direction=INGRESS \
  --allow=udp:7086 \
  --source-ranges=$PSC_IP/32 \
  --description="Allow SRT return traffic from LRS via PSC"
```

---

## Part 5 — Stream and Verify

### Step 13: Start the SRT stream from the VM

SSH into the VM and run:

```bash
# Test pattern stream → LRS Cloud Source via PSC
ffmpeg -re -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=48000 \
  -c:v libx264 -preset ultrafast -b:v 2M \
  -c:a aac -b:a 128k \
  -f mpegts "srt://$PSC_IP:7086?mode=caller"
```

### Step 14: Verify in Syndicate / VOS360

1. Open Syndicate → check the LRS Cloud Source status — it should show **Connected**
2. Check VOS360 → the ingest should be receiving the test stream
3. Verify the video is playing (test pattern with tone)

### Step 15: Verify connectivity from VM

If the stream doesn't connect, debug from the VM:

```bash
# Check if the PSC IP is reachable
ping -c 3 $PSC_IP

# Check if UDP port 7086 is reachable (timeout expected for UDP, but no "connection refused")
nc -zuv $PSC_IP 7086

# Check SRT connection with verbose logging
srt-live-transmit "file:///dev/null" "srt://$PSC_IP:7086?mode=caller" -v -timeout 5000000
```

---

## Summary — What You've Built

```
┌───────────────────────────┐        ┌──────────────────────────┐        ┌──────────────┐
│  "Globo" Test VPC         │        │  LRS VPC (Harmonic)      │        │  VOS360 VPC  │
│  (10.100.0.0/24)          │  PSC   │                          │Peering │  (Harmonic)  │
│                           │◄──────►│                          │◄──────►│              │
│  ┌──────────────────┐     │        │  ┌──────────────────┐    │        │  ┌────────┐  │
│  │ globo-srt-source │─SRT─┼───────►│  │ LRS Cloud Source │────┼───────►│  │ VOS360 │  │
│  │  (e2-medium)     │7086 │        │  │  (Listener:7086) │    │        │  │Ingest  │  │
│  └──────────────────┘     │        │  └──────────────────┘    │        │  └────────┘  │
│                           │        │                          │        │              │
│  Network Attachment:      │        │  PSC Interface:          │        │              │
│  globo-psc-attachment     │        │  IP = 10.100.0.2         │        │              │
└───────────────────────────┘        └──────────────────────────┘        └──────────────┘
```

---

## Cleanup (when done testing)

```bash
# Delete the VM
gcloud compute instances delete globo-srt-source --project=$CUSTOMER_PROJECT_ID --zone=$CUSTOMER_ZONE --quiet

# Delete firewall rules
gcloud compute firewall-rules delete globo-allow-ssh globo-allow-internal globo-allow-srt-egress globo-allow-srt-ingress --project=$CUSTOMER_PROJECT_ID --quiet

# Delete Cloud NAT and Router (if created)
gcloud compute routers nats delete globo-nat --router=globo-router --project=$CUSTOMER_PROJECT_ID --region=$CUSTOMER_REGION --quiet
gcloud compute routers delete globo-router --project=$CUSTOMER_PROJECT_ID --region=$CUSTOMER_REGION --quiet

# Delete Network Attachment
gcloud compute network-attachments delete globo-psc-attachment --project=$CUSTOMER_PROJECT_ID --region=$CUSTOMER_REGION --quiet

# Remove PSC Interface from LRS VM (stop VM first, remove the interface, start VM)
# gcloud compute instances stop $LRS_VM_NAME --project=$LRS_PROJECT_ID --zone=$LRS_ZONE
# gcloud compute instances remove-network-interface $LRS_VM_NAME --project=$LRS_PROJECT_ID --zone=$LRS_ZONE --network-interface-index=<INDEX>
# gcloud compute instances start $LRS_VM_NAME --project=$LRS_PROJECT_ID --zone=$LRS_ZONE

# Delete subnet and VPC
gcloud compute networks subnets delete globo-test-subnet --project=$CUSTOMER_PROJECT_ID --region=$CUSTOMER_REGION --quiet
gcloud compute networks delete globo-test-vpc --project=$CUSTOMER_PROJECT_ID --quiet
```

---

## Notes

- **Three VPCs** — Customer VPC → (PSC) → LRS VPC → (VPC Peering) → VOS360 VPC. LRS has its own dedicated VPC, separate from VOS360
- **Same region is mandatory** — PSC only works within the same GCP region. The customer VPC and LRS VPC must be co-located
- **The LRS VM must be stopped** to add/remove PSC interfaces — plan for a maintenance window
- **This lab validates the network path only** — the actual Globo setup will use their existing VPC and Direct Connect, not a test VPC
- **Port 7086** — LRS Cloud Source always listens on this port; each Cloud Source needs its own PSC Interface IP
- **LRS project ≠ VOS360 project** — Make sure you use the LRS VPC project ID (not VOS360) for the `--producer-accept-list` and PSC Interface commands
