# PSC Real LRS Test — SRT from Personal GCP to Harmonic LRS

## What Are We Doing?

Sending an SRT test stream from a VM in your **personal GCP account** through **Private Service Connect** to the **real Harmonic LRS** environment. This proves PSC works end-to-end with real LRS infrastructure and serves as a template for the Globo deployment.

```
  YOUR ACCOUNT: globo-customer              HARMONIC: saas-vos360-dev-vos360admin

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
│  (accepts Harmonic       │              │  (gets IP from your      │
│   project)               │              │   subnet: 10.100.0.x)   │
└──────────────────────────┘              └──────────────────────────┘
```

**Key difference from the lab:** Part 2 (LRS side) is the **real Harmonic LRS** — no simulated listener. You only control the left side.

---

## Environment Details

| | Your Side (Customer) | Harmonic Side (LRS) |
|---|---|---|
| **Project ID** | `globo-customer` | `saas-vos360-dev-vos360admin` |
| **VPC** | `globo-vpc` | `lrs-us-west1-0` |
| **Subnet** | `globo-subnet` (10.100.0.0/24) | (managed by Harmonic) |
| **Region** | `us-west1` | `us-west1` (must match) |
| **Role** | Consumer / Caller | Producer / Listener |

---

## What Already Exists (from yesterday's lab)

From yesterday, you still have in `globo-customer`:
- ✅ `globo-vpc` with `globo-subnet` (10.100.0.0/24)
- ✅ Firewall rules (`globo-allow-iap-ssh`, `globo-allow-internal`)
- ✅ Cloud Router (`globo-router`)
- ✅ Network Attachment (`globo-psc-attachment`) — **needs updating** (currently accepts `lrs-harmonic`, needs to accept `saas-vos360-dev-vos360admin`)

What was deleted:
- ❌ VMs (deleted)
- ❌ Cloud NAT (deleted)

---

## Step 1: Set Environment Variables

```bash
export GLOBO_PROJECT="globo-customer"
export LRS_PROJECT="saas-vos360-dev-vos360admin"
export REGION="us-west1"
export ZONE="us-west1-a"
```

---

## Step 2: Verify Existing Resources

> **What:** Confirm the VPC, subnet, and firewall rules from yesterday are still in place.

```bash
# Check VPC and subnet exist
gcloud compute networks subnets list \
  --project=$GLOBO_PROJECT \
  --filter="region:$REGION"

# Check firewall rules exist
gcloud compute firewall-rules list \
  --project=$GLOBO_PROJECT \
  --filter="network:globo-vpc"
```

You should see `globo-subnet` (10.100.0.0/24) and the firewall rules.

---

## Step 3: Recreate Cloud NAT

> **What:** Cloud NAT was deleted yesterday to save costs. Recreate it so the VM can install packages.

```bash
gcloud compute routers nats create globo-nat \
  --project=$GLOBO_PROJECT \
  --router=globo-router \
  --region=$REGION \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

---

## Step 4: Recreate the Globo SRT Source VM

> **What:** Recreate the test VM that will send the SRT stream. Same config as yesterday — no public IP, in `globo-subnet`.

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

---

## Step 5: Install SRT Tools on the VM

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

## Step 6: Update Network Attachment to Accept the Real LRS Project

> **What:** Yesterday's Network Attachment accepts `lrs-harmonic` (the simulated project). We need to update it to accept the **real LRS project** (`saas-vos360-dev-vos360admin`). Since `gcloud` doesn't support updating the accept list directly, we delete and recreate it.

```bash
# Delete the old Network Attachment
gcloud compute network-attachments delete globo-psc-attachment \
  --project=$GLOBO_PROJECT \
  --region=$REGION \
  --quiet

# Recreate with the real LRS project in the accept list
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

---

## Step 7: Get the Network Attachment URL

> **What:** The LRS team needs this URL to create a PSC Interface on their LRS VM. This is the handoff point — you provide the URL, they create the connection.

```bash
gcloud compute network-attachments describe globo-psc-attachment \
  --project=$GLOBO_PROJECT \
  --region=$REGION \
  --format="value(selfLink)"
```

This will output something like:

```
https://www.googleapis.com/compute/v1/projects/globo-customer/regions/us-west1/networkAttachments/globo-psc-attachment
```

**Save this URL — you'll share it with the LRS team.**

---

## Step 8: LRS Team Creates PSC Interface (Harmonic Side)

> **What:** This step is performed by someone with access to the LRS project (`saas-vos360-dev-vos360admin`). They add a **PSC Interface** (additional network interface) to the LRS Cloud Source VM, pointing to your Network Attachment. This gives the LRS VM an IP address in your subnet (10.100.0.x).

**Share the following with the LRS team:**

> I've created a Network Attachment in my GCP project for PSC testing.
>
> **Network Attachment URL:**
> `projects/globo-customer/regions/us-west1/networkAttachments/globo-psc-attachment`
>
> **What's needed:** Add a PSC Interface to the LRS Cloud Source VM in project `saas-vos360-dev-vos360admin`, VPC `lrs-us-west1-0`, using this Network Attachment URL.
>
> The PSC Interface will get an IP from my subnet (10.100.0.0/24). Please share the assigned IP once created.

If you have access to the LRS project yourself, the command would be:

```bash
# NOTE: The LRS VM must be stopped to add a network interface,
# OR the VM must be recreated with the additional interface.
# The exact command depends on how LRS Cloud Source VMs are managed.
# Coordinate with the LRS team for the correct approach.

# Example (if creating a new VM with both interfaces):
gcloud compute instances create <LRS_VM_NAME> \
  --project=$LRS_PROJECT \
  --zone=$ZONE \
  --machine-type=<MACHINE_TYPE> \
  --network-interface=network=projects/$LRS_PROJECT/global/networks/lrs-us-west1-0,subnet=<LRS_SUBNET>,no-address \
  --network-interface=network-attachment=projects/$GLOBO_PROJECT/regions/$REGION/networkAttachments/globo-psc-attachment
```

---

## Step 9: Get the PSC Interface IP

> **What:** Once the LRS team creates the PSC Interface, they'll provide the IP address assigned from your subnet. This is the IP your SRT source will connect to.

If you have access to describe the LRS VM:

```bash
gcloud compute instances describe <LRS_VM_NAME> \
  --project=$LRS_PROJECT \
  --zone=$ZONE \
  --format="yaml(networkInterfaces)"
```

Look for the interface with `networkAttachment` pointing to `globo-psc-attachment`. The `networkIP` is the PSC IP.

```bash
# Save the PSC IP for the next steps
export PSC_IP="<IP_FROM_LRS_TEAM>"   # e.g. 10.100.0.2
```

---

## Step 10: Configure SRT Firewall Rules

> **What:** Allow SRT traffic (UDP 7086) between your VM and the PSC Interface IP. SRT is bidirectional — the caller sends video data, the listener sends back NAKs, ACKs, and retransmit requests.

```bash
# Egress: allow Globo VM to send SRT to LRS via PSC
gcloud compute firewall-rules create globo-allow-srt-egress \
  --project=$GLOBO_PROJECT \
  --network=globo-vpc \
  --direction=EGRESS \
  --allow=udp:7086 \
  --destination-ranges=$PSC_IP/32 \
  --description="Allow SRT egress to real LRS via PSC"

# Ingress: allow SRT return traffic from LRS
gcloud compute firewall-rules create globo-allow-srt-ingress \
  --project=$GLOBO_PROJECT \
  --network=globo-vpc \
  --direction=INGRESS \
  --allow=udp:7086 \
  --source-ranges=$PSC_IP/32 \
  --description="Allow SRT return traffic from LRS via PSC"
```

---

## Step 11: Send SRT Test Stream to Real LRS

> **What:** Send a test video stream from your VM to the real LRS Cloud Source via PSC. This is the moment of truth — if LRS shows the stream as active, PSC is working end-to-end with real infrastructure.

SSH into the Globo VM:

```bash
gcloud compute ssh globo-srt-source \
  --project=$GLOBO_PROJECT \
  --zone=$ZONE \
  --tunnel-through-iap
```

Send the test stream:

```bash
ffmpeg -re -f lavfi -i "testsrc=size=1280x720:rate=30" \
  -f lavfi -i "sine=frequency=1000:sample_rate=48000" \
  -c:v libx264 -b:v 2M -c:a aac \
  -f mpegts "srt://$PSC_IP:7086?mode=caller"
```

> Replace `$PSC_IP` with the actual IP if the variable isn't set inside the VM.

---

## Step 12: Verify on LRS Side

> **What:** Confirm the SRT stream is received by the real LRS Cloud Source.

Check with the LRS team or LRS UI:
- LRS Cloud Source should show the stream as **Active**
- SRT statistics should show **0% packet loss** (private backbone)
- Connection mode should show **Caller** from your PSC IP

---

## Cleanup

When done testing, delete the billable resources:

```bash
# Delete VM
gcloud compute instances delete globo-srt-source \
  --project=$GLOBO_PROJECT --zone=$ZONE --quiet

# Delete Cloud NAT
gcloud compute routers nats delete globo-nat \
  --router=globo-router --project=$GLOBO_PROJECT --region=$REGION --quiet

# Delete SRT firewall rules (keep IAP and internal rules for future tests)
gcloud compute firewall-rules delete globo-allow-srt-egress \
  --project=$GLOBO_PROJECT --quiet
gcloud compute firewall-rules delete globo-allow-srt-ingress \
  --project=$GLOBO_PROJECT --quiet
```

Keep the VPC, subnet, router, and Network Attachment — they're free and can be reused for the real Globo deployment.

---

## What This Proves

If the SRT stream reaches the real LRS Cloud Source:

1. **PSC works cross-project** — your personal GCP project connected to Harmonic's LRS project via PSC
2. **SRT over PSC works** — real LRS accepted the SRT caller connection through the PSC Interface
3. **The template is validated** — the same steps apply to the real Globo deployment, just replace `globo-customer` with Globo's actual GCP project ID
4. **No public internet, no NAT, no firewall holes** — the entire path is private
