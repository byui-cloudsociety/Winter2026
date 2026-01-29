# Week 06

---

# 🚀 Lab: Build Your Private ChatGPT on AWS

**Cloud Solutions Society**

In this lab, we will launch an AWS EC2 instance, install an AI inference engine called **Ollama**, and build a web interface using **Streamlit** to chat with a local Large Language Model (Llama 3.2).

---

## 📋 Prerequisites

- Access to the **AWS Learner Lab** console.
- An SSH client (Terminal on Mac/Linux, or PowerShell/PuTTY on Windows).

---

## 🛠 Step 1: Launch Your EC2 Instance

1. **Launch Instance:** Click "Launch Instance" in the EC2 Dashboard.
2. **Name:** `My-Private-AI`
3. **OS:** Ubuntu 24.04 LTS (Amazon Machine Image).
4. **Instance Type:** Select `t3.large`.
5. **Key Pair:** Create a new key-pair named "private-ai-key" and then download the file.
6. **Storage (Critical):** Under "Configure Storage," change the size from **8 GiB** to **20 GiB**.
7. **IAM Role (Critical):** Under 'Advanced Details", select the "LabInstanceRole" in the IAM Role dropdown.
8. **Launch!**

---

## 🔓 Step 2: Open the Web Port

By default, AWS blocks external traffic to your web app. We need to open port **8501**.

1. Go to your instance **Security** tab.
2. Click on the **Security Groups** link.
3. Click **Edit inbound rules**.
4. Add a rule:

- **Type:** Custom TCP
- **Port Range:** `8501`
- **Source:** `0.0.0.0/0` (Anywhere)

5. Click **Save rules**.

---

## 💻 Step 3: Connect and Setup

SSH into your instance and follow these steps:

```bash
ssh -i /directory/of/private-key ubuntu@ec2-YOUR-DNS-URL-HERE

```

### 1. Create the Setup Script

Copy and paste this into your terminal:

```bash
nano setup.sh

```

Paste the following code inside:

```bash
#!/bin/bash
set -e

echo "--- Installing Dependencies ---"
sudo apt update
sudo apt install -y python3-pip python3-venv curl

echo "--- Installing Ollama ---"
curl -fsSL https://ollama.com/install.sh | sh
sleep 5

echo "--- Downloading Llama 3.2 (1B) ---"
ollama pull llama3.2:1b

echo "--- Setting up Python Environment ---"
python3 -m venv venv
source venv/bin/activate
pip install streamlit ollama

echo "✅ SETUP COMPLETE!"

```

_Press `Ctrl+O`, `Enter`, then `Ctrl+X` to save and exit._

### 2. Run the Setup

```bash
bash setup.sh

```

_(This will take 2-3 minutes to download the model and install libraries.)_

---

## 🐍 Step 4: Create the Chat App

Now, create the Python file that will serve as your ChatGPT-style interface.

```bash
nano app.py

```

Paste this code:

```python
import streamlit as st
import ollama

st.title("☁️ Cloud Society: My Private AI")

if "messages" not in st.session_state:
    st.session_state.messages = []

for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

if prompt := st.chat_input("Ask me anything..."):
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.markdown(prompt)

    with st.chat_message("assistant"):
        stream = ollama.chat(
            model='llama3.2:1b',
            messages=st.session_state.messages,
            stream=True,
        )
        response = st.write_stream(chunk['message']['content'] for chunk in stream)

    st.session_state.messages.append({"role": "assistant", "content": response})

```

_Press `Ctrl+O`, `Enter`, then `Ctrl+X` to save._

---

## 🚀 Step 5: Run the App!

Activate your environment and launch the server:

```bash
source venv/bin/activate
streamlit run app.py --server.address 0.0.0.0

```

### How to access:

Look for the **External URL** in the terminal output, or copy your **EC2 Public IP address** and paste it into your browser with `:8501` at the end:
`http://your-ec2-ip:8501`

---

## 🏁 Challenges for Early Finishers

1. **Change the Model:** Go into `app.py`, change `llama3.2:1b` to `phi3`, and run `ollama pull phi3` in the terminal. Which one is smarter?
2. **Personality:** Change the system prompt in the `ollama.chat` call to make the AI respond like a pirate 🏴‍☠️.
3. **Check Resources:** Open a new terminal tab, SSH in, and run `top`. Watch the CPU usage spike when the AI generates text!
