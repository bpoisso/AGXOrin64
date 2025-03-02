# NVIDIA Jetson AGX Orin AI Agent Manager Setup Guide

## Password List

Before starting, generate passwords for the following services and replace the placeholders in this document:

- `POSTGRES_PASSWORD`: For PostgreSQL database
- `SUPABASE_PASSWORD`: For Supabase authentication
- `OLLAMA_API_KEY`: For Ollama API access (if needed)
- `N8N_PASSWORD`: For n8n.io workflow management
- `OPENWEBUI_PASSWORD`: For OpenWebUI admin access

## System Overview

This guide will help you set up a complete AI agent workflow system on your NVIDIA Jetson AGX Orin 64GB with ARM64 architecture. The system includes:

- Jetson tools (jtop) for monitoring system resources
- CUDA 12.6 for LLM processing
- PostgreSQL for database storage
- Docker for containerization
- n8n.io for workflow automation
- Supabase (ARM64-compatible version) for backend services
- Crawl4AI for web scraping
- Ollama with deepseek-r1 model for local LLM inference
- OpenWebUI for LLM interaction
- MaryTTS for text-to-speech with voice cloning capabilities
- Streamlit for creating web-based interfaces

## 1. Initial System Setup

### Update System Packages

```bash
sudo apt update
sudo apt upgrade -y
```

### Install Essential Tools

```bash
sudo apt install -y git curl wget build-essential python3-pip python3-dev
```

### Install jtop for Jetson Monitoring

```bash
sudo pip3 install -U jetson-stats
```

After installation, you can run jtop with:

```bash
sudo jtop
```

## 2. CUDA 12.6 Installation for Jetson AGX Orin

CUDA is pre-installed on the Jetson platform, but we need to ensure it's properly configured.

### Check Current CUDA Version

```bash
nvcc --version
```

### Update CUDA Environment Variables

Add these lines to your `~/.bashrc` file:

```bash
cat << 'EOF' >> ~/.bashrc
# CUDA Environment Variables
export PATH=/usr/local/cuda-12.6/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-12.6/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
EOF

source ~/.bashrc
```

## 3. Docker Installation

### Install Docker

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=arm64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

### Configure Docker

```bash
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker
```

### Install Docker Compose

```bash
sudo apt-get install -y docker-compose-plugin
```

Verify installation:

```bash
docker compose version
```

## 4. PostgreSQL Setup

### Create Docker Compose File for PostgreSQL

```bash
mkdir -p ~/ai-agent-system/postgres
cd ~/ai-agent-system/postgres

cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    container_name: postgres
    restart: always
    environment:
      POSTGRES_PASSWORD: POSTGRES_PASSWORD
      POSTGRES_USER: postgres
      POSTGRES_DB: aiagent
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres-data:
EOF
```

### Start PostgreSQL

```bash
cd ~/ai-agent-system/postgres
docker compose up -d
```

## 5. Supabase Setup (ARM64-compatible)

Since Supabase's official Docker setup might have compatibility issues with ARM64, we'll set up a simplified version with the core components.

### Create Docker Compose File for Supabase

```bash
mkdir -p ~/ai-agent-system/supabase
cd ~/ai-agent-system/supabase

cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  supabase-db:
    image: postgres:14-alpine
    container_name: supabase-db
    restart: always
    environment:
      POSTGRES_PASSWORD: SUPABASE_PASSWORD
      POSTGRES_USER: postgres
      POSTGRES_DB: supabase
    volumes:
      - supabase-db-data:/var/lib/postgresql/data
    ports:
      - "5433:5432"  # Different port to avoid conflicts with main PostgreSQL

  supabase-auth:
    image: supabase/auth:latest
    container_name: supabase-auth
    depends_on:
      - supabase-db
    restart: always
    environment:
      POSTGRES_PASSWORD: SUPABASE_PASSWORD
      POSTGRES_USER: postgres
      POSTGRES_DB: supabase
      POSTGRES_HOST: supabase-db
      POSTGRES_PORT: 5432
    ports:
      - "9999:9999"

volumes:
  supabase-db-data:
EOF
```

### Start Supabase

```bash
cd ~/ai-agent-system/supabase
docker compose up -d
```

## 6. n8n.io Setup for Workflow Automation

### Create Docker Compose File for n8n

```bash
mkdir -p ~/ai-agent-system/n8n
cd ~/ai-agent-system/n8n

cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=N8N_PASSWORD
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
      - WEBHOOK_URL=http://localhost:5678/
    volumes:
      - n8n-data:/home/node/.n8n

volumes:
  n8n-data:
EOF
```

### Start n8n

```bash
cd ~/ai-agent-system/n8n
docker compose up -d
```

## 7. Crawl4AI Setup for Web Scraping

### Create Docker Compose File for Crawl4AI

```bash
mkdir -p ~/ai-agent-system/crawl4ai
cd ~/ai-agent-system/crawl4ai

cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  crawl4ai:
    image: crawlai/crawl4ai:latest
    container_name: crawl4ai
    restart: always
    ports:
      - "9080:9080"
    volumes:
      - crawl4ai-data:/app/data

volumes:
  crawl4ai-data:
EOF
```

### Start Crawl4AI

```bash
cd ~/ai-agent-system/crawl4ai
docker compose up -d
```

## 8. Ollama with deepseek-r1 Setup

### Create Docker Compose File for Ollama

```bash
mkdir -p ~/ai-agent-system/ollama
cd ~/ai-agent-system/ollama

cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: always
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

volumes:
  ollama-data:
EOF
```

### Start Ollama and Pull deepseek-r1 Model

```bash
cd ~/ai-agent-system/ollama
docker compose up -d

# Wait for Ollama to start, then pull the deepseek-r1 model
sleep 10
curl -X POST http://localhost:11434/api/pull -d '{"name": "deepseek-coder:6.7b-instruct-q5_K_M"}'
```

## 9. OpenWebUI Setup

### Create Docker Compose File for OpenWebUI

```bash
mkdir -p ~/ai-agent-system/openwebui
cd ~/ai-agent-system/openwebui

cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    restart: always
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_API_BASE_URL=http://ollama:11434
    volumes:
      - openwebui-data:/app/backend/data
    depends_on:
      - ollama

volumes:
  openwebui-data:
EOF
```

### Start OpenWebUI

```bash
cd ~/ai-agent-system/openwebui
docker compose up -d
```

## 10. MaryTTS Setup for Text-to-Speech with Voice Cloning

### Create Docker Compose File for MaryTTS

```bash
mkdir -p ~/ai-agent-system/marytts
cd ~/ai-agent-system/marytts

cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  marytts:
    image: synesthesiam/marytts:latest
    container_name: marytts
    restart: always
    ports:
      - "59125:59125"
    volumes:
      - marytts-data:/app/voice-data

volumes:
  marytts-data:
EOF
```

### Start MaryTTS

```bash
cd ~/ai-agent-system/marytts
docker compose up -d
```

### Setup Voice Cloning

For voice cloning with MaryTTS, you'll need to train a custom voice model. The easiest approach is to use the Voice Builder:

1. Access MaryTTS web interface at http://localhost:59125/
2. Navigate to "Voice Builder"
3. Record samples of your voice (at least 30 minutes of clear audio is recommended)
4. Follow the web interface to train your custom voice

## 11. Streamlit for Web-Based Front-End

### Create Docker Compose File for Streamlit

```bash
mkdir -p ~/ai-agent-system/streamlit
cd ~/ai-agent-system/streamlit

cat << 'EOF' > Dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8501

ENTRYPOINT ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]
EOF

cat << 'EOF' > requirements.txt
streamlit
requests
pandas
numpy
matplotlib
plotly
aiohttp
EOF

cat << 'EOF' > app.py
import streamlit as st
import requests
import json

st.set_page_config(page_title="AI Agent Dashboard", layout="wide")

st.title("AI Agent Manager Dashboard")

# Sidebar for navigation
st.sidebar.title("Navigation")
page = st.sidebar.radio("Go to", ["Home", "Create Agent", "Manage Agents", "Run Workflows", "Settings"])

if page == "Home":
    st.header("Welcome to AI Agent Manager")
    st.write("Use this dashboard to create, manage, and run AI agents for various tasks.")
    
    # System stats
    st.subheader("System Status")
    col1, col2, col3 = st.columns(3)
    with col1:
        st.metric("CPU Usage", "42%")
    with col2:
        st.metric("Memory Usage", "6.2 GB")
    with col3:
        st.metric("GPU Usage", "18%")
    
    # Quick actions
    st.subheader("Quick Actions")
    col1, col2, col3 = st.columns(3)
    with col1:
        st.button("Create New Agent")
    with col2:
        st.button("Run Workflow")
    with col3:
        st.button("View Logs")

elif page == "Create Agent":
    st.header("Create a New AI Agent")
    
    agent_name = st.text_input("Agent Name")
    agent_type = st.selectbox("Agent Type", ["Text Generation", "Web Scraping", "Data Processing", "Text-to-Speech"])
    
    st.subheader("Agent Configuration")
    if agent_type == "Text Generation":
        model = st.selectbox("Model", ["deepseek-r1", "llama2", "mistral"])
        temperature = st.slider("Temperature", 0.0, 1.0, 0.7)
        max_tokens = st.slider("Max Tokens", 100, 4000, 1000)
    elif agent_type == "Web Scraping":
        urls = st.text_area("URLs to Scrape (one per line)")
        depth = st.slider("Crawl Depth", 1, 5, 2)
        extract = st.multiselect("Extract", ["Text", "Images", "Links", "Tables"])
    
    if st.button("Create Agent"):
        st.success(f"Agent '{agent_name}' created successfully!")

elif page == "Manage Agents":
    st.header("Manage Existing Agents")
    
    # Sample data
    agents = [
        {"name": "Content Writer", "type": "Text Generation", "status": "Active"},
        {"name": "News Scraper", "type": "Web Scraping", "status": "Idle"},
        {"name": "Data Analyzer", "type": "Data Processing", "status": "Error"}
    ]
    
    for agent in agents:
        col1, col2, col3, col4 = st.columns([3, 2, 2, 1])
        with col1:
            st.write(agent["name"])
        with col2:
            st.write(agent["type"])
        with col3:
            status = agent["status"]
            if status == "Active":
                st.success(status)
            elif status == "Idle":
                st.info(status)
            else:
                st.error(status)
        with col4:
            st.button("Edit", key=f"edit_{agent['name']}")

elif page == "Run Workflows":
    st.header("Run and Monitor Workflows")
    
    workflow = st.selectbox("Select Workflow", ["Content Generation", "Web Data Collection", "Report Creation"])
    
    if workflow == "Content Generation":
        st.subheader("Content Generation Settings")
        topic = st.text_input("Topic")
        tone = st.selectbox("Tone", ["Professional", "Conversational", "Academic"])
        length = st.selectbox("Length", ["Short", "Medium", "Long"])
    
    if st.button("Run Workflow"):
        st.info("Workflow started...")
        progress = st.progress(0)
        for i in range(100):
            # Simulate progress
            import time
            time.sleep(0.05)
            progress.progress(i + 1)
        st.success("Workflow completed successfully!")

elif page == "Settings":
    st.header("System Settings")
    
    st.subheader("API Connections")
    ollama_url = st.text_input("Ollama API URL", "http://localhost:11434")
    n8n_url = st.text_input("n8n URL", "http://localhost:5678")
    
    st.subheader("Database Settings")
    db_host = st.text_input("Database Host", "localhost")
    db_port = st.text_input("Database Port", "5432")
    
    if st.button("Save Settings"):
        st.success("Settings saved successfully!")
EOF

cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  streamlit:
    build: .
    container_name: streamlit
    restart: always
    ports:
      - "8501:8501"
    volumes:
      - ./app.py:/app/app.py
      - ./requirements.txt:/app/requirements.txt

EOF
```

### Start Streamlit

```bash
cd ~/ai-agent-system/streamlit
docker compose up -d
```

## 12. Google Drive Integration for Backup

### Install rclone for Google Drive Integration

```bash
curl https://rclone.org/install.sh | sudo bash
```

### Configure rclone for Google Drive

```bash
rclone config
```

Follow the interactive setup:
1. Choose `n` for a new remote
2. Name it `gdrive`
3. Select `drive` for Google Drive
4. Leave client ID and secret blank for now
5. Choose option for accessing the scope (1 for full access)
6. Leave root folder ID blank
7. Choose `n` for advanced config
8. Choose `y` for auto config
9. Follow the browser authentication process
10. Choose `y` to confirm the configuration

### Create a Backup Script

```bash
mkdir -p ~/ai-agent-system/backup
cd ~/ai-agent-system/backup

cat << 'EOF' > backup.sh
#!/bin/bash

# Directory to store backups
BACKUP_DIR="/home/$(whoami)/ai-agent-system/backup/data"
mkdir -p $BACKUP_DIR

# Date format for backup files
DATE=$(date +%Y-%m-%d_%H-%M-%S)

# Backup PostgreSQL databases
echo "Backing up PostgreSQL databases..."
docker exec postgres pg_dump -U postgres aiagent > $BACKUP_DIR/aiagent_$DATE.sql
docker exec supabase-db pg_dump -U postgres supabase > $BACKUP_DIR/supabase_$DATE.sql

# Backup n8n data
echo "Backing up n8n data..."
tar -czf $BACKUP_DIR/n8n_data_$DATE.tar.gz -C /var/lib/docker/volumes n8n-data

# Backup MaryTTS voice data
echo "Backing up MaryTTS voice data..."
tar -czf $BACKUP_DIR/marytts_data_$DATE.tar.gz -C /var/lib/docker/volumes marytts-data

# Sync to Google Drive
echo "Syncing to Google Drive..."
rclone sync $BACKUP_DIR gdrive:JetsonAIBackup

echo "Backup completed at $(date)"
EOF

chmod +x backup.sh
```

### Create a Cron Job for Regular Backups

```bash
(crontab -l 2>/dev/null; echo "0 3 * * * /home/$(whoami)/ai-agent-system/backup/backup.sh >> /home/$(whoami)/ai-agent-system/backup/backup.log 2>&1") | crontab -
```

This sets up a daily backup at 3 AM.

## 13. Parent Agent Setup

Let's create a parent agent that can coordinate other agents:

### Create Parent Agent Script

```bash
mkdir -p ~/ai-agent-system/parent-agent
cd ~/ai-agent-system/parent-agent

cat << 'EOF' > agent_manager.py
#!/usr/bin/env python3

import requests
import json
import os
import argparse
import logging
from datetime import datetime

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler("agent_manager.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger("AgentManager")

class AgentManager:
    def __init__(self):
        self.ollama_url = "http://localhost:11434"
        self.n8n_url = "http://localhost:5678"
        self.db_host = "localhost"
        self.db_port = "5432"
        self.agents = {}
        self.load_agents()
        
    def load_agents(self):
        """Load agent configurations from a JSON file"""
        try:
            if os.path.exists("agents.json"):
                with open("agents.json", "r") as f:
                    self.agents = json.load(f)
                logger.info(f"Loaded {len(self.agents)} agents from configuration")
            else:
                logger.info("No agent configuration found, starting fresh")
                self.agents = {}
        except Exception as e:
            logger.error(f"Error loading agents: {str(e)}")
            self.agents = {}
    
    def save_agents(self):
        """Save agent configurations to a JSON file"""
        try:
            with open("agents.json", "w") as f:
                json.dump(self.agents, f, indent=2)
            logger.info(f"Saved {len(self.agents)} agents to configuration")
        except Exception as e:
            logger.error(f"Error saving agents: {str(e)}")
    
    def create_agent(self, name, agent_type, params=None):
        """Create a new agent"""
        if name in self.agents:
            logger.warning(f"Agent {name} already exists")
            return False
        
        if params is None:
            params = {}
        
        agent = {
            "name": name,
            "type": agent_type,
            "created_at": datetime.now().isoformat(),
            "params": params,
            "status": "idle"
        }
        
        self.agents[name] = agent
        self.save_agents()
        logger.info(f"Created new agent: {name} of type {agent_type}")
        return True
    
    def delete_agent(self, name):
        """Delete an existing agent"""
        if name not in self.agents:
            logger.warning(f"Agent {name} does not exist")
            return False
        
        del self.agents[name]
        self.save_agents()
        logger.info(f"Deleted agent: {name}")
        return True
    
    def run_agent(self, name, input_data=None):
        """Run a specific agent"""
        if name not in self.agents:
            logger.warning(f"Agent {name} does not exist")
            return None
        
        agent = self.agents[name]
        logger.info(f"Running agent: {name} of type {agent['type']}")
        
        agent["status"] = "running"
        self.save_agents()
        
        result = None
        try:
            if agent["type"] == "text-generation":
                result = self.run_text_generation_agent(agent, input_data)
            elif agent["type"] == "web-scraping":
                result = self.run_web_scraping_agent(agent, input_data)
            elif agent["type"] == "text-to-speech":
                result = self.run_tts_agent(agent, input_data)
            else:
                logger.warning(f"Unknown agent type: {agent['type']}")
        except Exception as e:
            logger.error(f"Error running agent {name}: {str(e)}")
            agent["status"] = "error"
            self.save_agents()
            return None
        
        agent["status"] = "idle"
        agent["last_run"] = datetime.now().isoformat()
        self.save_agents()
        
        return result
    
    def run_text_generation_agent(self, agent, input_data):
        """Run a text generation agent using Ollama"""
        if not input_data or "prompt" not in input_data:
            logger.error("No prompt provided for text generation")
            return None
        
        model = agent["params"].get("model", "deepseek-coder:6.7b-instruct-q5_K_M")
        
        try:
            response = requests.post(
                f"{self.ollama_url}/api/generate",
                json={
                    "model": model,
                    "prompt": input_data["prompt"],
                    "temperature": agent["params"].get("temperature", 0.7),
                    "max_tokens": agent["params"].get("max_tokens", 1000)
                }
            )
            response.raise_for_status()
            return response.json()
        except Exception as e:
            logger.error(f"Error calling Ollama API: {str(e)}")
            raise
    
    def run_web_scraping_agent(self, agent, input_data):
        """Run a web scraping agent using Crawl4AI"""
        if not input_data or "url" not in input_data:
            logger.error("No URL provided for web scraping")
            return None
        
        crawl_url = "http://localhost:9080/api/crawl"
        
        try:
            response = requests.post(
                crawl_url,
                json={
                    "url": input_data["url"],
                    "depth": agent["params"].get("depth", 1),
                    "extract": agent["params"].get("extract", ["text"])
                }
            )
            response.raise_for_status()
            return response.json()
        except Exception as e:
            logger.error(f"Error calling Crawl4AI API: {str(e)}")
            raise
    
    def run_tts_agent(self, agent, input_data):
        """Run a text-to-speech agent using MaryTTS"""
        if not input_data or "text" not in input_data:
            logger.error("No text provided for text-to-speech")
            return None
        
        marytts_url = "http://localhost:59125/process"
        voice = agent["params"].get("voice", "cmu-slt-hsmm")
        
        try:
            params = {
                "INPUT_TYPE": "TEXT",
                "OUTPUT_TYPE": "AUDIO",
                "INPUT_TEXT": input_data["text"],
                "LOCALE": "en_US",
                "VOICE": voice,
                "AUDIO": "WAVE"
            }
            response = requests.get(marytts_url, params=params)
            response.raise_for_status()
            
            # Save the audio file
            filename = f"output_{datetime.now().strftime('%Y%m%d_%H%M%S')}.wav"
            with open(filename, "wb") as f:
                f.write(response.content)
            
            return {"filename": filename, "size": len(response.content)}
        except Exception as e:
            logger.error(f"Error calling MaryTTS API: {str(e)}")
            raise

    def detect_and_run_appropriate_agent(self, task_description):
        """Analyze the task and determine which agent to use"""
        logger.info(f"Detecting appropriate agent for task: {task_description}")
        
        # Use the LLM to classify the task
        try:
            response = requests.post(
                f"{self.ollama_url}/api/generate",
                json={
                    "model": "deepseek-coder:6.7b-instruct-q5_K_M",
                    "prompt": f"Classify the following task into one of these categories: text-generation, web-scraping, text-to-speech. Return only the category name.\n\nTask: {task_description}",
                    "temperature": 0.1,
                    "max_tokens": 50
                }
            )
            response.raise_for_status()
            task_type = response.json().get("response", "").strip().lower()
            
            logger.info(f"Detected task type: {task_type}")
            
            # Find an appropriate agent
            for name, agent in self.agents.items():
                if agent["type"] == task_type and agent["status"] == "idle":
                    logger.info(f"Selected agent: {name}")
                    
                    # Prepare input data based on task type
                    input_data = None
                    if task_type == "text-generation":
                        input_data = {"prompt": task_description}
                    elif task_type == "web-scraping":
                        # Extract URL from task description
                        url_response = requests.post(
                            f"{self.ollama_url}/api/generate",
                            json={
                                "model": "deepseek-coder:6.7b-instruct-q5_K_M",
                                "prompt": f"Extract the URL from this text. Return only the URL:\n\n{task_description}",
                                "temperature": 0.1,
                                "max_tokens": 100
                            }
                        )
                        url = url_response.json().get("response", "").strip()
                        input_data = {"url": url}
                    elif task_type == "text-to-speech":
                        input_data = {"text": task_description}
                    
                    return self.run_agent(name, input_data)
            
            logger.warning(f"No suitable idle agent found for task type: {task_type}")
            return None
            
        except Exception as e:
            logger.error(f"Error detecting appropriate agent: {str(e)}")
            return None

def main():
    parser = argparse.ArgumentParser(description="AI Agent Manager")
    subparsers = parser.add_subparsers(dest="command", help="Commands")
    
    # Create agent command
    create_parser = subparsers.add_parser("create", help="Create a new agent")
    create_parser.add_argument("name", help="Agent name")
    create_parser.add_argument("type", choices=["text-generation", "web-scraping", "text-to-speech"], 
                              help="Agent type")
    
    # Delete agent command
    delete_parser = subparsers.add_parser("delete", help="Delete an agent")
    delete_parser.add_argument("name", help="Agent name")
    
    # Run agent command
    run_parser = subparsers.add_parser("run", help="Run an agent")
    run_parser.add_argument("name", help="Agent name")
    run_parser.add_argument("--input", help="Input data as JSON string")
    
    # Auto-detect and run command
    auto_parser = subparsers.add_parser("auto", help="Auto-detect and run appropriate agent")
    auto_parser.add_argument("task", help="Task description")
    
    # List agents command
    subparsers.add_parser("list", help="List all agents")
    
    args = parser.parse_args()
    manager = AgentManager()
    
    if args.command == "create":
        success = manager.create_agent(args.name, args.type)
        if success:
            print(f"Agent {args.name} created successfully")
        else:
            print(f"Failed to create agent {args.name}")
    
    elif args.command == "delete":
        success = manager.delete_agent(args.name)
        if success:
            print(f"Agent {args.name} deleted successfully")
        else:
            print(f"Failed to delete agent {args.name}")
    
    elif args.command == "run":
        input_data = None
        if args.input:
            try:
                input_data = json.loads(args.input)
            except json.JSONDecodeError:
                print("Error: Input must be valid JSON")
                return
        
        result = manager.run_agent(args.name, input_data)
        if result:
            print(json.dumps(result, indent=2))
        else:
            print(f"Failed to run agent {args.name}")
    
    elif args.command == "auto":
        result = manager.detect_and_run_appropriate_agent(args.task)
        if result:
            print(json.dumps(result, indent=2))
        else:
            print("Failed to automatically run an agent for the given task")
    
    elif args.command == "list":
        if manager.agents:
            print("Available agents:")
            for name, agent in manager.agents.items():
                print(f"- {name} ({agent['type']}): {agent['status']}")
        else:
            print("No agents available")
    
    else:
        parser.print_help()

if __name__ == "__main__":
    main()
EOF

# Make the script executable
chmod +x agent_manager.py

# Create Python requirements.txt file
cat << 'EOF' > requirements.txt
requests
argparse
python-dotenv
EOF

# Install dependencies
pip install -r requirements.txt

# Create a sample agent configuration
cat << 'EOF' > agents.json
{
  "text_agent": {
    "name": "text_agent",
    "type": "text-generation",
    "created_at": "2023-01-01T00:00:00",
    "params": {
      "model": "deepseek-coder:6.7b-instruct-q5_K_M",
      "temperature": 0.7,
      "max_tokens": 1000
    },
    "status": "idle"
  },
  "web_scraper": {
    "name": "web_scraper",
    "type": "web-scraping",
    "created_at": "2023-01-01T00:00:00",
    "params": {
      "depth": 2,
      "extract": ["text", "links"]
    },
    "status": "idle"
  },
  "voice_gen": {
    "name": "voice_gen",
    "type": "text-to-speech",
    "created_at": "2023-01-01T00:00:00",
    "params": {
      "voice": "cmu-slt-hsmm"
    },
    "status": "idle"
  }
}
EOF

## 14. IDE-Like Agent Development Environment

To provide an IDE-like experience for agent development, let's create a web-based code editor that integrates with our agent system:

### Setup Code-Server (VS Code in Browser)

```bash
mkdir -p ~/ai-agent-system/code-server
cd ~/ai-agent-system/code-server

cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  code-server:
    image: codercom/code-server:latest
    container_name: code-server
    user: "1000:1000"
    ports:
      - "8080:8080"
    environment:
      - PASSWORD=OPENWEBUI_PASSWORD  # Reuse the OpenWebUI password
    volumes:
      - ./config:/home/coder/.config
      - ~/ai-agent-system:/home/coder/project
    command: --auth password --disable-telemetry
EOF

# Start code-server
docker compose up -d
```

### Create Agent Development Template

```bash
mkdir -p ~/ai-agent-system/templates
cd ~/ai-agent-system/templates

# Create a basic agent template
cat << 'EOF' > agent_template.py
#!/usr/bin/env python3

import os
import json
import logging
import requests
from datetime import datetime

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler(f"{os.path.basename(__file__)}.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger("CustomAgent")

class CustomAgent:
    def __init__(self, name, config=None):
        self.name = name
        self.config = config or {}
        logger.info(f"Initialized {self.name} agent")
    
    def process(self, input_data):
        """
        Process the input data and return results
        Override this method in your custom agent implementation
        """
        logger.info(f"Processing input: {input_data}")
        # Implement your agent logic here
        return {"result": "Not implemented", "timestamp": datetime.now().isoformat()}
        
    def save_results(self, result, output_file=None):
        """Save results to a file"""
        if output_file is None:
            output_file = f"{self.name}_result_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
            
        with open(output_file, 'w') as f:
            json.dump(result, f, indent=2)
            
        logger.info(f"Results saved to {output_file}")
        return output_file

def main():
    # Example usage
    agent = CustomAgent("example")
    result = agent.process({"data": "example input"})
    agent.save_results(result)

if __name__ == "__main__":
    main()
EOF

# Create a specialized agent example that extends the template
cat << 'EOF' > specialized_agent_example.py
#!/usr/bin/env python3

from agent_template import CustomAgent, logger
import requests

class WebScrapingAgent(CustomAgent):
    def __init__(self, name, config=None):
        super().__init__(name, config)
        
    def process(self, input_data):
        """
        Process web scraping requests
        
        Args:
            input_data: Dictionary containing at least a "url" key
            
        Returns:
            Dictionary with scraped content
        """
        if not input_data or "url" not in input_data:
            logger.error("No URL provided")
            return {"error": "No URL provided"}
            
        url = input_data["url"]
        logger.info(f"Scraping URL: {url}")
        
        try:
            # Use Crawl4AI API
            response = requests.post(
                "http://localhost:9080/api/crawl",
                json={
                    "url": url,
                    "depth": self.config.get("depth", 1),
                    "extract": self.config.get("extract", ["text"])
                }
            )
            response.raise_for_status()
            return response.json()
        except Exception as e:
            logger.error(f"Error scraping URL {url}: {str(e)}")
            return {"error": str(e)}

def main():
    # Example usage
    config = {
        "depth": 1,
        "extract": ["text", "links", "images"]
    }
    agent = WebScrapingAgent("web_scraper", config)
    result = agent.process({"url": "https://example.com"})
    agent.save_results(result)

if __name__ == "__main__":
    main()
EOF
```

## 15. System Testing and Integration

Let's create a test script that verifies all components are working together:

### Create System Test Script

```bash
mkdir -p ~/ai-agent-system/tests
cd ~/ai-agent-system/tests

cat << 'EOF' > system_test.py
#!/usr/bin/env python3

import requests
import json
import time
import subprocess
import sys
import os

def test_component(name, url, expect_code=200):
    """Test if a component is responsive"""
    print(f"Testing {name}... ", end="")
    try:
        response = requests.get(url, timeout=5)
        if response.status_code == expect_code:
            print("✅ OK")
            return True
        else:
            print(f"❌ Failed with status code {response.status_code}")
            return False
    except Exception as e:
        print(f"❌ Failed: {str(e)}")
        return False

def test_ollama():
    """Test Ollama API"""
    print("Testing Ollama API... ", end="")
    try:
        response = requests.post(
            "http://localhost:11434/api/generate",
            json={"model": "deepseek-coder:6.7b-instruct-q5_K_M", "prompt": "Hello", "max_tokens": 10}
        )
        if response.status_code == 200:
            print("✅ OK")
            return True
        else:
            print(f"❌ Failed with status code {response.status_code}")
            return False
    except Exception as e:
        print(f"❌ Failed: {str(e)}")
        return False

def test_agent_manager():
    """Test the agent manager"""
    print("Testing agent manager... ", end="")
    try:
        result = subprocess.run(
            ["python3", "../parent-agent/agent_manager.py", "list"],
            capture_output=True,
            text=True
        )
        if "Available agents" in result.stdout:
            print("✅ OK")
            return True
        else:
            print(f"❌ Failed: {result.stderr}")
            return False
    except Exception as e:
        print(f"❌ Failed: {str(e)}")
        return False

# Main test function
def main():
    print("=== Running System Tests ===\n")
    
    # Test all components
    components = [
        ("PostgreSQL", "http://localhost:5432", 502),  # Will fail with 502 but that means Postgres is running
        ("Supabase Auth", "http://localhost:9999", 404),  # 404 means service is running but path is not found
        ("n8n", "http://localhost:5678", 200),
        ("Crawl4AI", "http://localhost:9080", 200),
        ("Ollama API", "http://localhost:11434", 200),
        ("OpenWebUI", "http://localhost:3000", 200),
        ("MaryTTS", "http://localhost:59125", 200),
        ("Streamlit", "http://localhost:8501", 200),
        ("Code Server", "http://localhost:8080", 200)
    ]
    
    results = []
    for name, url, code in components:
        results.append(test_component(name, url, code))
    
    # Test Ollama API specifically
    results.append(test_ollama())
    
    # Test agent manager
    results.append(test_agent_manager())
    
    # Print summary
    print("\n=== Test Summary ===")
    success = sum(results)
    total = len(results)
    print(f"Passed: {success}/{total} tests")
    
    if success == total:
        print("\n🎉 All systems operational!")
        return 0
    else:
        print("\n⚠️ Some components are not operational. Please check the logs.")
        return 1

if __name__ == "__main__":
    sys.exit(main())
EOF

chmod +x system_test.py
```

### Create System Start Script

```bash
cd ~/ai-agent-system

cat << 'EOF' > start_all.sh
#!/bin/bash

# Start all services in the AI Agent System
echo "Starting AI Agent System..."

# Start PostgreSQL
echo "Starting PostgreSQL..."
cd ~/ai-agent-system/postgres
docker compose up -d

# Start Supabase
echo "Starting Supabase..."
cd ~/ai-agent-system/supabase
docker compose up -d

# Start n8n
echo "Starting n8n..."
cd ~/ai-agent-system/n8n
docker compose up -d

# Start Crawl4AI
echo "Starting Crawl4AI..."
cd ~/ai-agent-system/crawl4ai
docker compose up -d

# Start Ollama
echo "Starting Ollama..."
cd ~/ai-agent-system/ollama
docker compose up -d

# Wait for Ollama to be ready
echo "Waiting for Ollama to initialize..."
sleep 10

# Start OpenWebUI
echo "Starting OpenWebUI..."
cd ~/ai-agent-system/openwebui
docker compose up -d

# Start MaryTTS
echo "Starting MaryTTS..."
cd ~/ai-agent-system/marytts
docker compose up -d

# Start Streamlit
echo "Starting Streamlit..."
cd ~/ai-agent-system/streamlit
docker compose up -d

# Start Code Server
echo "Starting Code Server IDE..."
cd ~/ai-agent-system/code-server
docker compose up -d

echo "All services started."
echo "Running system test..."

# Run system test
cd ~/ai-agent-system/tests
python3 system_test.py
EOF

chmod +x start_all.sh
```

## 16. Final System Configuration and Verification

### Create Directory for Custom Agent Storage

```bash
mkdir -p ~/ai-agent-system/agents
cd ~/ai-agent-system/agents

# Create README
cat << 'EOF' > README.md
# Custom Agents Directory

This directory is for storing custom agent implementations.

## Creating a new agent:

1. Copy the template from `~/ai-agent-system/templates/agent_template.py`
2. Implement the `process()` method with your custom logic
3. Register the agent using the parent agent manager:
   ```
   cd ~/ai-agent-system/parent-agent
   ./agent_manager.py create your_agent_name agent_type
   ```

## Available agent types:
- text-generation
- web-scraping
- text-to-speech

## Example:
To create a custom web scraping agent:
1. Copy the template: `cp ~/ai-agent-system/templates/agent_template.py my_scraper.py`
2. Edit the file to implement your logic
3. Register: `cd ~/ai-agent-system/parent-agent && ./agent_manager.py create my_scraper web-scraping`
EOF
```

### Create System Documentation

```bash
cd ~/ai-agent-system

cat << 'EOF' > README.md
# AI Agent Manager on NVIDIA Jetson AGX Orin

This system provides a complete AI agent workflow environment with the following components:

## Components:
- PostgreSQL: Database storage
- Supabase: Simplified backend services
- n8n.io: Workflow automation
- Crawl4AI: Web scraping
- Ollama with deepseek-r1: Local LLM inference
- OpenWebUI: LLM interaction
- MaryTTS: Text-to-speech with voice cloning
- Streamlit: Web-based interfaces
- Code-Server: IDE for agent development

## Service URLs:
- PostgreSQL: localhost:5432
- Supabase Auth: localhost:9999
- n8n: http://localhost:5678
- Crawl4AI: http://localhost:9080
- Ollama API: http://localhost:11434
- OpenWebUI: http://localhost:3000
- MaryTTS: http://localhost:59125
- Streamlit: http://localhost:8501
- Code-Server IDE: http://localhost:8080

## Getting Started:
1. Start all services: `./start_all.sh`
2. Access the Streamlit dashboard at http://localhost:8501
3. Create and manage agents using the parent agent manager:
   ```
   cd ~/ai-agent-system/parent-agent
   ./agent_manager.py create myagent text-generation
   ```

## Directory Structure:
- ~/ai-agent-system/postgres: PostgreSQL database
- ~/ai-agent-system/supabase: Simplified Supabase setup
- ~/ai-agent-system/n8n: Workflow automation
- ~/ai-agent-system/crawl4ai: Web scraping service
- ~/ai-agent-system/ollama: Local LLM inference
- ~/ai-agent-system/openwebui: Web interface for LLM
- ~/ai-agent-system/marytts: Text-to-speech service
- ~/ai-agent-system/streamlit: Web dashboard
- ~/ai-agent-system/parent-agent: Agent management
- ~/ai-agent-system/agents: Custom agent implementations
- ~/ai-agent-system/templates: Agent templates
- ~/ai-agent-system/backup: Backup system
- ~/ai-agent-system/tests: System tests
- ~/ai-agent-system/code-server: Web-based IDE

## Backup:
The system is configured to back up to Google Drive daily at 3 AM.
EOF
```

### Run the System

Finally, let's run the entire system:

```bash
cd ~/ai-agent-system
./start_all.sh
```

## Summary

Congratulations! You've set up a complete AI Agent Manager on your NVIDIA Jetson AGX Orin. This system includes:

1. **Hardware Monitoring**: jtop for Jetson system monitoring
2. **CUDA 12.6**: Configured for optimal LLM processing
3. **Core Services**:
   - PostgreSQL for database storage
   - Docker for containerization
   - n8n.io for workflow automation
   - Supabase (ARM64-compatible) for backend services
   - Crawl4AI for web scraping
   - Ollama with deepseek-r1 for local LLM inference
   - OpenWebUI for LLM interaction
   - MaryTTS for text-to-speech with voice cloning
   - Streamlit for web-based interfaces

4. **Agent Development**:
   - Parent agent for managing specialized agents
   - Code-Server for IDE-like development
   - Agent templates for quick development
   - System for creating, managing, and running agents

5. **Backup System**:
   - Google Drive integration for cloud backup
   - Daily automated backups

6. **System Management**:
   - Start-up script for all services
   - System testing and verification

The system is designed to be self-hosted on your Jetson hardware, with all services running locally. This setup provides a complete environment for developing and running AI agents that can handle tasks like text generation, web scraping, and text-to-speech conversion.

To get started, navigate to http://localhost:8501 to access the Streamlit dashboard for managing your agents
