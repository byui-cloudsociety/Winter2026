# Week 04

# 🛠 Host Setup: AWS EC2 Deployment

Follow these steps to deploy the game server on an AWS Academy Learner Lab account.

### **Step 1: Launch the EC2 Instance**

1. Log in to your **AWS Academy Learner Lab** and click **Start Lab**.
2. Once the dot turns green, click **AWS** to open the Management Console.
3. Navigate to **EC2** > **Instances** > **Launch Instance**.
4. **Name:** `Game-Server-Lab`
5. **AMI:** **Ubuntu 24.04 LTS** (64-bit x86).
6. **Instance Type:** Select **`t3.medium`** (2 vCPUs, 4GB RAM).

- _Note: If `t3.medium` is unavailable in your lab, `t3.small` will work but may lag with many players._

7. **Key Pair:** Create a new key pair or select an existing one to SSH into the machine.
8. **Network Settings:** Select **Create Security Group**. Add these **Inbound Rules**:

- **SSH (TCP 22):** Your IP (for access).
- **Custom UDP (30000):** Anywhere (Minetest).
- **Custom UDP (26000):** Anywhere (Xonotic).
- **Custom UDP (2757):** Anywhere (SuperTuxKart).
- **Custom UDP (2759):** Anywhere (SuperTuxKart).

9. **Advanced Details:**

- **IAM instance profile:** Select **`LabInstanceRole`**. This is critical for Academy accounts.

10. Click **Launch Instance**.

---

### **Step 2: Assign an Elastic IP (Static IP)**

By default, AWS changes your IP every time you stop/start the lab. An Elastic IP prevents this.

1. In the EC2 Sidebar, go to **Network & Security** > **Elastic IPs**.
2. Click **Allocate Elastic IP address** > **Allocate**.
3. Select the new IP, click **Actions** > **Associate Elastic IP address**.
4. Choose your `Game-Server-Lab` instance and click **Associate**.

- _Note: Update the student handout with this permanent IP._

---

### **Step 3: Server Configuration (Terminal)**

Once you SSH into your instance (`ssh -i your-key.pem ubuntu@your-elastic-ip`), run this "one-liner" script to install everything and launch the games.

```bash
# 1. Update and Install Docker
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose -y
sudo usermod -aG docker $USER

# 2. Create the project directory
mkdir ~/game-lab && cd ~/game-lab

# 3. Create the Docker Compose file with Pinned Versions
cat <<EOF > docker-compose.yml
services:
  luanti:
    image: linuxserver/luanti:5.15.0
    container_name: luanti_server
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - ./luanti_data:/config
    ports:
      - 30000:30000/udp
    restart: unless-stopped

  xonotic:
    image: umair101/xonotic-server:v1.1
    container_name: xonotic_server
    ports:
      - 26000:26000/udp
    restart: unless-stopped

  supertuxkart:
    image: timoschwarzer/supertuxkart-server:1.3
    container_name: stk_server
    ports:
      - 2757:2757/udp
      - 2759:2759/udp
    restart: unless-stopped
EOF

# 4. Launch the servers
sudo docker-compose up -d

```

---

### **Step 4: Monitoring**

To see if students are connecting or if a game is crashing, use:

- **Check status:** `sudo docker ps`
- **Live logs:** `sudo docker logs -f luanti_server` (Press Ctrl+C to exit logs).

---

### **A Quick "Learner Lab" Warning**

In AWS Academy, if you don't use the lab for a few hours, it will **auto-stop** your instance to save credits.

- **When you restart the lab:** You will need to log back into the EC2 console and **Start** the instance manually.
- **Because of the Elastic IP:** Your IP address will stay the same, so students won't need to change their connection settings!

---

# 🎮 University Game Lab: Connection Guide

Welcome to the lab! We are hosting three different open-source games on an AWS EC2 instance. To ensure you can connect, please download the **exact versions** listed below for your operating system.

**Server IP Address:** `[PASTE_YOUR_EC2_PUBLIC_IP_HERE]`

---

## 1. Luanti (Minetest)

- **Version:** 5.10.0
- **Port:** `30000`
- **Downloads:**
- [Windows (64-bit) & Mac](https://www.luanti.org/en/downloads/)

- **How to Join:** 1. Open Luanti.

2. Click the **Join Game** tab.
3. Enter the IP and Port above.
4. Choose any username and a password (this registers you on the server).

---

## 2. Xonotic (Fast-Paced Arena Shooter)

- **Version:** 0.8.6
- **Port:** `26000`
- **Downloads:**
- [Windows/macOS/Linux (Universal .zip)](https://www.google.com/search?q=https://dl.xonotic.org/xonotic-0.8.6.zip)

- **How to Join:**

1. Extract the `.zip` file.
2. **Windows:** Run `xonotic.exe`.
3. **macOS:** Right-click `Xonotic.app` and select **Open** (you may need to bypass the security warning in System Settings).
4. Go to **Multiplayer** → **Address** and enter `[IP]:26000`.

---

## 3. SuperTuxKart (Kart Racing)

- **Version:** 1.4
- **Port:** `2757, 2759`
- **Downloads:**
- [Game Installer](https://supertuxkart.net/Download)

- **How to Join:**

1. Open the game and go to **Online**.
2. Select **Enter Address**.
3. Enter the IP address. The port should be `2759`.

---

## 🛠 Troubleshooting

- **"Connection Timed Out":** Ensure you are on the university "Guest" Wi-Fi or a network that allows outgoing UDP traffic.
- **"Version Mismatch":** Double-check that you didn't download a "Nightly" or "Development" build. Use the links above.
- **Mac Users:** If an app won't open, go to **System Settings > Privacy & Security** and click **"Open Anyway"** at the bottom of the page.
