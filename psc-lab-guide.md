# PSC Lab Guide — Self-Contained SRT-over-PSC Test

## Objective

Validate Private Service Connect (PSC) + SRT end-to-end **without needing access to the real LRS project**. We create two VPCs in the same GCP project — one simulating the customer (Globo), one simulating LRS — connected via PSC. An SRT Caller on one side streams to an SRT Listener on the other.

```
┌──────────────────────────┐              ┌──────────────────────────┐
│  "Globo" VPC             │     PSC      │  "LRS" VPC               │
│                          │◄────────────►│                          │
│  ┌────────────────────┐  │              │  ┌────────────────────┐  │
│  │ globo-srt-source   │──┼── SRT ──────►│  │ lrs-srt-listener   │  │
│  │ (Caller :7086)     │  │              │  │ (Listener :7086)   │  │
│  └────────────────────┘  │              │  └────────────────────┘  │
│                          │              │                          │
│  Network Attachment      │              │  PSC Interface           │
│  (Consumer)              │              │  (Producer)              │
└──────────────────────────┘              └──────────────────────────┘

Both VPCs in project: saas-vos360-dev-harmonicseg
```

When this works, the same PSC pattern applies to the real Globo ↔ LRS setup — just swap the "LRS" VPC for the actual LRS VPC project.

---

## Prerequisites

- [x] Access to GCP Cloud Shell (or `gcloud` CLI locally)
- [x] Project `saas-vos360-dev-harmonicseg` (or any project you have Compute/Network Admin on)
- [ ] Compute Engine API enabled in the project

```bash
# Set these once — used in all commands below
export PROJECT_ID="saas-vos360-dev-harmonicseg"
export REGION="us-central1"
export ZONE="us-central1-a"
```

---

## Part 1 — Create the "Customer" VPC (Simulating Globo)

### Step 1: Create the Customer VPC and subnet

```bash
# Create VPC
gcloud compute networks create globo-vpc \
  --project=$PROJECT_ID \
  --subnet-mode=custom

# Create subnet
gcloud compute networks subnets create globo-subnet \
  --project=$PROJECT_ID \
  --network=globo-vpc \
  --region=$REGION \
  --range=10.100.0.0/24
```

### Step 2: Create firewall rules for Customer VPC

```bash
# Allow SSH via IAP
gcloud compute firewall-rules create globo-allow-iap-ssh \
  --project=$PROJECT_ID \
  --network=globo-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --description="Allow SSH via IAP"

# Allow all internal traffic
gcloud compute firewall-rules create globo-allow-internal \
  --project=$PROJECT_ID \
  --network=globo-vpc \
  --allow=all \
  --source-ranges=10.100.0.0/24 \
  --description="Allow internal VPC traffic"
```

> `35.235.240.0/20` is Google's IAP IP range — this allows SSH without a public IP on the VM.

---

## Part 2 — Create the "LRS" VPC (Simulating Harmonic LRS)

### Step 3: Create the LRS VPC and subnet

```bash
# Create VPC
gcloud compute networks create lrs-vpc \
  --project=$PROJECT_ID \
  --subnet-mode=custom

# Create subnet
gcloud compute networks subnets create lrs-subnet \
  --project=$PROJECT_ID \
  --network=lrs-vpc \
  --region=$REGION \
  --range=10.200.0.0/24
```

### Step 4: Create firewall rules for LRS VPC

```bash
# Allow SSH via IAP
gcloud compute firewall-rules create lrs-allow-iap-ssh \
  --project=$PROJECT_ID \
  --network=lrs-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --description="Allow SSH via IAP"

# Allow all internal traffic
gcloud compute firewall-rules create lrs-allow-internal \
  --project=$PROJECT_ID \
  --network=lrs-vpc \
  --allow=all \
  --source-ranges=10.200.0.0/24 \
  --description="Allow internal VPC traffic"

# Allow SRT ingress from PSC (the PSC interface will get an IP from globo-subnet 10.100.0.0/24)
gcloud compute firewall-rules create lrs-allow-srt-from-psc \
  --project=$PROJECT_ID \
  --network=lrs-vpc \
  --allow=udp:7086 \
  --source-ranges=10.100.0.0/24 \
  --description="Allow SRT traffic from customer VPC via PSC"
```

---

## Part 3 — Launch VMs

### Step 5: Create the SRT Listener VM (simulating LRS Cloud Source)

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

### Step 6: Create the SRT Caller VM (simulating Globo SRT Source)

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

### Step 7: Add Cloud NAT for both VPCs (for package installation)

Both VMs need outbound internet to install SRT tools:

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

### Step 8: Install SRT tools on both VMs

SSH into each VM and install:

```bash
# SSH into LRS Listener
gcloud compute ssh lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --tunnel-through-iap

# Inside the VM:
sudo apt-get update && sudo apt-get install -y srt-tools
exit
```

```bash
# SSH into Globo Caller
gcloud compute ssh globo-srt-source \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --tunnel-through-iap

# Inside the VM:
sudo apt-get update && sudo apt-get install -y srt-tools ffmpeg
exit
```

---

## Part 4 — Set Up Private Service Connect

### Step 9: Create the Network Attachment (Customer/Globo side)

This is the "consumer" side — Globo creates it and controls who can connect:

```bash
gcloud compute network-attachments create globo-psc-attachment \
  --project=$PROJECT_ID \
  --region=$REGION \
  --subnets=globo-subnet \
  --connection-preference=ACCEPT_MANUAL \
  --producer-accept-list=$PROJECT_ID
```

> Since both VPCs are in the same project, `--producer-accept-list` is our own project ID. In the real Globo setup, this would be the Harmonic LRS project ID.

### Step 10: Get the Network Attachment URL

```bash
gcloud compute network-attachments describe globo-psc-attachment \
  --project=$PROJECT_ID \
  --region=$REGION \
  --format="value(selfLink)"
```

Save the output — looks like:
```
projects/saas-vos360-dev-harmonicseg/regions/us-central1/networkAttachments/globo-psc-attachment
```

### Step 11: Stop the LRS VM and add PSC Interface

The PSC interface can only be added to a stopped VM:

```bash
# Stop the LRS Listener VM
gcloud compute instances stop lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE

# Add the PSC interface (this creates a second NIC connected to the customer's VPC)
gcloud compute instances add-network-interface lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --network-attachment=projects/$PROJECT_ID/regions/$REGION/networkAttachments/globo-psc-attachment

# Start the VM
gcloud compute instances start lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE
```

### Step 12: Get the PSC Interface IP

```bash
gcloud compute instances describe lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --format="json(networkInterfaces)"
```

Look for the network interface with `networkAttachment` — the `networkIP` is the PSC IP (from the 10.100.0.0/24 Globo subnet). Example: `10.100.0.2`

```bash
export PSC_IP="10.100.0.2"   # replace with actual IP from output
```

---

## Part 5 — Configure SRT Firewall Rules

### Step 13: Allow SRT traffic in the Customer VPC

```bash
# Egress: allow Globo VM to send SRT to PSC IP
gcloud compute firewall-rules create globo-allow-srt-egress \
  --project=$PROJECT_ID \
  --network=globo-vpc \
  --direction=EGRESS \
  --allow=udp:7086 \
  --destination-ranges=$PSC_IP/32 \
  --description="Allow SRT egress to LRS via PSC"

# Ingress: allow SRT return traffic from PSC IP
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

### Step 14: Start the SRT Listener on the LRS VM

SSH into the LRS VM and start listening:

```bash
gcloud compute ssh lrs-srt-listener \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --tunnel-through-iap
```

Inside the VM — start the SRT Listener. The listener must bind to `0.0.0.0` (all interfaces) so it accepts traffic on the PSC interface:

```bash
# Listen on all interfaces, port 7086, and dump received data to /dev/null
srt-live-transmit "srt://0.0.0.0:7086?mode=listener" "file:///dev/null" -v
```

> You should see it waiting for a connection. Leave this running.

### Step 15: Start the SRT Caller on the Globo VM

Open a **second Cloud Shell tab** (or terminal), SSH into the Globo VM:

```bash
gcloud compute ssh globo-srt-source \
  --project=$PROJECT_ID \
  --zone=$ZONE \
  --tunnel-through-iap
```

Inside the VM — send a test stream:

```bash
# Option A: ffmpeg test pattern → SRT Caller
ffmpeg -re -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=48000 \
  -c:v libx264 -preset ultrafast -b:v 2M \
  -c:a aac -b:a 128k \
  -f mpegts "srt://$PSC_IP:7086?mode=caller"

# Option B: simpler — just send a stream with srt-live-transmit
# First generate a small test file:
ffmpeg -f lavfi -i testsrc=size=320x240:rate=25 -t 60 -c:v libx264 -f mpegts /tmp/test.ts
# Then transmit it in a loop:
while true; do srt-live-transmit "file:///tmp/test.ts" "srt://$PSC_IP:7086?mode=caller" -v; done
```

### Step 16: Verify

On the **LRS Listener VM**, you should see:
- `SRT connected` message
- Data bytes being received
- No errors

If it works, **PSC + SRT is validated**. The exact same pattern applies to the real Globo setup — just replace the "LRS" VPC with the actual Harmonic LRS VPC.

### Troubleshooting

If the connection doesn't establish:

```bash
# From the Globo VM — check if PSC IP is reachable
ping -c 3 $PSC_IP

# Check UDP port
nc -zuv $PSC_IP 7086

# On the LRS VM — verify the PSC interface exists
ip addr show
# You should see two interfaces:
#   ens4 (10.200.0.x) — primary NIC on LRS VPC
#   ens5 (10.100.0.x) — PSC interface on Globo VPC
```

Common issues:
- **Firewall rules missing** — check both VPCs have the right rules
- **SRT Listener not bound to 0.0.0.0** — if bound to 10.200.0.x (primary NIC), it won't see PSC traffic
- **PSC Interface not accepted** — check Network Attachment status: `gcloud compute network-attachments describe globo-psc-attachment --region=$REGION`

---

## Summary — What This Proves

| What | Validated |
|------|-----------|
| Network Attachment creation (customer side) | ✓ |
| PSC Interface provisioning (producer side) | ✓ |
| PSC gives the LRS VM a reachable IP in customer's subnet | ✓ |
| SRT Caller → PSC → SRT Listener works | ✓ |
| Bidirectional UDP (SRT handshake + data) flows through PSC | ✓ |
| Firewall rules needed for SRT over PSC | ✓ |

**What's different in production:** The "LRS" side is the real LRS VPC (different project), and LRS Cloud Source handles the SRT Listener automatically. The customer side (Network Attachment, firewall rules, SRT Caller config) is identical.

---

## Cleanup

```bash
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

---

## Notes

- **Same project, two VPCs** — simulates cross-project PSC without needing access to the LRS project
- **Same region is mandatory** — PSC only works within the same GCP region
- **VM must be stopped** to add PSC interface — plan for this in production
- **SRT Listener must bind to 0.0.0.0** — not the primary NIC IP, otherwise PSC traffic is ignored
- **Port 7086** — matches real LRS Cloud Source SRT port
- **No VPC Peering needed for this test** — we're only validating the Customer ↔ LRS leg. The LRS ↔ VOS360 peering is Harmonic's internal setup
