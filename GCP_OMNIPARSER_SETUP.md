# Setting Up OmniParser on Google Cloud Platform

## Prerequisites
- Google account
- Credit card (for verification, won't be charged during $300 free trial)

---

## Part 1: Create GCP Account & Get Free Credits

### Step 1: Sign Up
1. Go to: https://cloud.google.com/free
2. Click **"Get started for free"**
3. Sign in with Google account
4. Enter billing info (required but won't charge during trial)
5. **Activate $300 free credit** (valid for 90 days)

### Step 2: Create Project
1. Go to: https://console.cloud.google.com
2. Click project dropdown → **"New Project"**
3. Name: `omniparser-ufo`
4. Click **"Create"**

---

## Part 2: Create GPU Instance

### Step 1: Enable Compute Engine API
1. Navigate to: **Compute Engine** → **VM instances**
2. Click **"Enable"** if prompted (takes 2-3 minutes)

### Step 2: Request GPU Quota (Important!)
**By default, GPU quota is 0. You MUST request increase:**

1. Go to: **IAM & Admin** → **Quotas**
2. Filter: `GPUs (all regions)`
3. Select: **"GPUs (all regions)"**
4. Click **"Edit Quotas"**
5. Request: **1** (for T4 GPU)
6. Reason: "Running OmniParser for Windows automation research"
7. Submit → Wait for approval (usually 2-24 hours)

### Step 3: Create VM Instance
Once quota is approved:

1. Go to: **Compute Engine** → **VM instances**
2. Click **"Create Instance"**

**Configuration:**
```
Name: omniparser-server
Region: us-central1 (or closest to you)
Zone: us-central1-a

Machine configuration:
  - Series: N1
  - Machine type: n1-standard-4 (4 vCPUs, 15 GB)

GPUs:
  - Click "Add GPU"
  - GPU type: NVIDIA Tesla T4
  - Number of GPUs: 1

Boot disk:
  - Operating system: Deep Learning on Linux
  - Version: Deep Learning VM for PyTorch 2.0 with CUDA 11.8
  - Boot disk type: Standard persistent disk
  - Size: 50 GB

Firewall:
  ✅ Allow HTTP traffic
  ✅ Allow HTTPS traffic
```

3. Click **"Create"** (takes 2-3 minutes)

---

## Part 3: Configure Firewall Rules

### Allow OmniParser Port (7861)

1. Go to: **VPC Network** → **Firewall**
2. Click **"Create Firewall Rule"**

**Configuration:**
```
Name: allow-omniparser
Direction: Ingress
Action on match: Allow
Targets: All instances in the network
Source IP ranges: 0.0.0.0/0
Protocols and ports: TCP → 7861
```

3. Click **"Create"**

---

## Part 4: Install OmniParser on VM

### Step 1: Connect to VM
1. In **VM instances**, click **"SSH"** next to your instance
2. Browser SSH terminal will open

### Step 2: Setup Script
Run this complete setup:

```bash
# Update system
sudo apt-get update

# Clone OmniParser
cd ~
git clone https://github.com/microsoft/OmniParser.git
cd OmniParser

# Install dependencies
pip install -r requirements.txt

# Download model weights (V2)
mkdir -p weights
cd weights
huggingface-cli download microsoft/OmniParser-v2.0 "icon_detect/train_args.yaml" --local-dir .
huggingface-cli download microsoft/OmniParser-v2.0 "icon_detect/model.pt" --local-dir .
huggingface-cli download microsoft/OmniParser-v2.0 "icon_detect/model.yaml" --local-dir .
huggingface-cli download microsoft/OmniParser-v2.0 "icon_caption/config.json" --local-dir .
huggingface-cli download microsoft/OmniParser-v2.0 "icon_caption/generation_config.json" --local-dir .
huggingface-cli download microsoft/OmniParser-v2.0 "icon_caption/model.safetensors" --local-dir .

# Rename caption folder
mv icon_caption icon_caption_florence

cd ~/OmniParser
```

### Step 3: Verify GPU
```bash
nvidia-smi
# Should show Tesla T4 GPU
```

### Step 4: Start OmniParser Server
```bash
# Start with public access
python gradio_demo.py
```

**Server will start on:** `http://0.0.0.0:7861`

### Step 5: Get External IP
```bash
# In new SSH tab, run:
curl -s http://checkip.amazonaws.com
```

Copy the IP address (e.g., `34.123.45.67`)

---

## Part 5: Test OmniParser

### From Your Browser:
```
http://YOUR_EXTERNAL_IP:7861
```

You should see the Gradio interface!

---

## Part 6: Configure UFO to Use Cloud OmniParser

### On Your Windows Machine:

1. **Edit UFO config:**
```powershell
# Open config file
notepad D:\dev\UFO\ufo\config\config.yaml
```

2. **Update OMNIPARSER section:**
```yaml
OMNIPARSER: {
  ENDPOINT: "http://YOUR_EXTERNAL_IP:7861",
  BOX_THRESHOLD: 0.05,
  IOU_THRESHOLD: 0.1,
  USE_PADDLEOCR: True,
  IMGSZ: 640
}
```

3. **Update CONTROL_BACKEND in config_dev.yaml:**
```yaml
CONTROL_BACKEND: ["uia", "win32", "omniparser"]
```

4. **Test UFO:**
```powershell
cd D:\dev\UFO
source .venv\Scripts\activate
python -m ufo --task test_omniparser -r "open notepad"
```

---

## Part 7: Keep Server Running (Optional)

### Option A: Run in Background (Simple)
```bash
# Install screen
sudo apt-get install -y screen

# Create named screen session
screen -S omniparser

# Start server
cd ~/OmniParser
python gradio_demo.py

# Detach: Press Ctrl+A, then D
# Reattach later: screen -r omniparser
```

### Option B: Systemd Service (Production)
Create service file:

```bash
sudo nano /etc/systemd/system/omniparser.service
```

Content:
```ini
[Unit]
Description=OmniParser Gradio Server
After=network.target

[Service]
Type=simple
User=YOUR_USERNAME
WorkingDirectory=/home/YOUR_USERNAME/OmniParser
ExecStart=/usr/bin/python3 gradio_demo.py
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable omniparser
sudo systemctl start omniparser
sudo systemctl status omniparser
```

---

## Cost Management

### Stop Instance When Not in Use:
```bash
# From GCP Console:
VM instances → Select instance → STOP
```

**You're only charged when instance is RUNNING**

### Monitor Usage:
1. Go to: **Billing** → **Reports**
2. Track your $300 credit usage

### Expected Costs (with $300 credit):
- **T4 GPU (preemptible):** ~$0.15/hour = ~2000 hours total
- **T4 GPU (standard):** ~$0.51/hour = ~588 hours total
- **Network egress:** Free tier (1GB/month)

---

## Troubleshooting

### GPU Not Available
```bash
# Check CUDA
nvcc --version

# Check PyTorch GPU access
python -c "import torch; print(torch.cuda.is_available())"
# Should print: True
```

### Port 7861 Not Accessible
```bash
# Check firewall rule exists
gcloud compute firewall-rules list | grep 7861

# Verify server is running
netstat -tuln | grep 7861
```

### Out of Memory
```bash
# Check GPU memory
nvidia-smi

# Restart server
pkill -f gradio_demo.py
python gradio_demo.py
```

---

## Security Considerations

### Production Setup (Optional):
1. **Add authentication** to Gradio:
```python
# In gradio_demo.py, modify launch():
demo.launch(share=True, server_port=7861, server_name='0.0.0.0',
           auth=("username", "password"))
```

2. **Restrict IP access** in firewall rule:
   - Instead of `0.0.0.0/0`, use your Windows PC's public IP
   - Find at: https://whatismyipaddress.com/

---

## Cleanup (When Done)

### Delete Resources:
1. **Stop VM:** Compute Engine → VM instances → STOP
2. **Delete VM:** Select instance → DELETE
3. **Delete firewall rule:** VPC Network → Firewall → DELETE rule
4. **Delete project:** IAM & Admin → Settings → SHUT DOWN

---

## Summary

**Total Setup Time:** ~30-60 minutes (including quota approval wait)

**Architecture:**
```
┌─────────────────────┐         ┌──────────────────────────┐
│  Your Windows PC    │────────▶│  GCP VM (us-central1)    │
│  Running UFO        │  HTTP   │  Running OmniParser      │
│  D:\dev\UFO         │◀────────│  + NVIDIA T4 GPU         │
└─────────────────────┘         └──────────────────────────┘
```

**Cost with Free Trial:** $0 (using $300 credit)
**Post-Trial Cost:** ~$0.15/hour (preemptible) or ~$0.51/hour (standard)

---

## Next Steps

1. ✅ Create GCP account
2. ✅ Request GPU quota
3. ✅ Create VM instance
4. ✅ Install OmniParser
5. ✅ Configure UFO
6. ✅ Test end-to-end

**Questions?** Check GCP documentation: https://cloud.google.com/compute/docs
