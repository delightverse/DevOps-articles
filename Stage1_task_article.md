# From Zero to Hero: Building a Production-Grade Automated Deployment Script

*A beginner-friendly guide to automating Docker deployments with Bash*

**Github repo of the task solution:** https://github.com/delightverse/hng13-stage1-devops

---

## 🎯 What You'll Learn

By the end of this article, you'll understand:
- What deployment automation really means
- How to write production-ready Bash scripts
- Docker containerization in practice
- Nginx reverse proxy configuration
- Real-world DevOps workflows

**Difficulty Level:** Intermediate  
**Time to Read:** 15 minutes  
**Practical Application:** Immediate - use this at work/internship!

---

## 🌍 The Real-World Problem

Imagine you're working at a startup. Your team deploys new features multiple times a day. Currently, the deployment process looks like this:

```
1. SSH into server (manually)
2. Pull latest code (manually)  
3. Stop old container (manually)
4. Build new Docker image (manually, takes 5 minutes)
5. Start new container (manually)
6. Configure Nginx (manually, prone to typos)
7. Test if it works (manually)
8. If it fails, debug and repeat (wasted time)
```

**Each deployment takes 20-30 minutes. You do this 5 times a day.**

That's **2 hours of repetitive work every single day!**

---

## 💡 The Solution: Automation

What if you could run ONE command and everything happens automatically?

```bash
./deploy.sh
```

That's exactly what we built!

---

## 🏗️ What We're Building

### The Big Picture

```
┌─────────────────────────────────────────────────┐
│  YOUR LAPTOP                                    │
│                                                 │
│  Run: ./deploy.sh                               │
│    ↓                                            │
│  Script asks questions                          │
│    ↓                                            │
│  You answer (repo URL, server IP, etc.)         │
└─────────────────┬───────────────────────────────┘
                  │
                  │ (Script executes automatically)
                  ↓
┌─────────────────────────────────────────────────┐
│  GITHUB                                         │
│    • Clone repository with your code            │
│    • Authenticate using Personal Access Token   │
└─────────────────┬───────────────────────────────┘
                  │
                  ↓
┌────────────────────────────────────────────────┐
│  REMOTE SERVER (AWS/DigitalOcean/etc)          │
│                                                │
│  1. Install Docker (if not present)            │
│  2. Install Nginx (if not present)             │
│  3. Build Docker container                     │
│  4. Run your application                       │
│  5. Configure Nginx proxy                      │
│  6. Verify everything works                    │
│                                                │
│  Result: Your app is LIVE!                     │
└────────────────────────────────────────────────┘
```

**All of this happens in ONE command!**

---

## 📚 Core Concepts Explained

### 1. Bash Scripting

**What is it?**  
Bash is a command language that lets you automate tasks in Linux/Unix systems.

**Why use it?**  
- Available on every Linux server
- No installation required
- Perfect for automation
- Industry standard

**Simple Example:**
```bash
#!/bin/bash
echo "Hello, DevOps!"
```

### 2. Docker Containerization

**What is it?**  
Docker packages your application with everything it needs to run (code, dependencies, system libraries) into a "container."

**Why use it?**  
- **"Works on my machine" problem solved!** If it works in Docker locally, it works everywhere.
- Consistent environments across dev, testing, production
- Easy to deploy and scale

**Real-World Analogy:**  
Think of Docker like shipping containers. Just as a shipping container can hold anything and be moved anywhere (ship, truck, train), a Docker container holds your app and runs anywhere (your laptop, AWS, Google Cloud).

**Dockerfile Example:**
```dockerfile
FROM node:18-alpine      # Start with Node.js base
WORKDIR /app            # Set working directory
COPY app.js .           # Copy your code
EXPOSE 3000             # Expose port
CMD ["node", "app.js"]  # Run your app
```

### 3. Nginx Reverse Proxy

**What is it?**  
Nginx sits between the internet and your application, forwarding requests.

**Why use it?**  
- Handle SSL/HTTPS
- Load balancing
- Serve static files
- Better security

**Simple Analogy:**  
Nginx is like a receptionist at a company. Visitors (web requests) come to the front desk (Nginx), and the receptionist directs them to the right person (your application).

**Configuration Example:**
```nginx
server {
    listen 80;
    location / {
        proxy_pass http://localhost:3000;  # Forward to your app
    }
}
```

---

## 🛠️ Building the Script: Step by Step

### Phase 1: Collecting User Input

Our script needs information before it can deploy. Let's ask the user!

```bash
#!/bin/bash

# Ask for Git repository
read -p "Enter Git Repository URL: " GIT_REPO_URL

# Ask for authentication
read -sp "Enter Personal Access Token: " GIT_PAT
echo ""

# Ask for server details
read -p "Enter SSH username [default: ubuntu]: " SSH_USER
SSH_USER=${SSH_USER:-ubuntu}  # Use 'ubuntu' if user presses Enter

read -p "Enter server IP address: " SERVER_IP

read -p "Enter SSH key path: " SSH_KEY_PATH
```

**What's happening?**  
- `-p` shows a prompt message
- `-sp` hides password input (secure!)
- `${VAR:-default}` uses default value if variable is empty

### Phase 2: Input Validation

**Never trust user input!** Always validate.

```bash
# Validate IP address format
validate_ip() {
    local ip=$1
    if [[ $ip =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
        return 0  # Valid
    fi
    return 1  # Invalid
}

# Use it
if validate_ip "$SERVER_IP"; then
    echo "✓ IP address is valid"
else
    echo "✗ Invalid IP address!"
    exit 1
fi
```

**Why validate?**  
- Catch errors early
- Better user experience
- Prevent security issues
- Professional quality

### Phase 3: Git Operations

Clone the repository using authentication:

```bash
# Inject PAT into URL for authentication
auth_url=$(echo "$GIT_REPO_URL" | sed "s|https://|https://${GIT_PAT}@|")

# Clone repository
if git clone -b main "$auth_url" app-directory; then
    echo "✓ Repository cloned successfully"
else
    echo "✗ Git clone failed"
    exit 2
fi

# Verify Dockerfile exists
if [ ! -f "app-directory/Dockerfile" ]; then
    echo "✗ No Dockerfile found!"
    exit 2
fi
```

**Security Note:** The PAT is hidden in the URL and never logged!

### Phase 4: SSH Connection Test

Before doing anything, verify we can connect:

```bash
# Test SSH connection
if ssh -i "$SSH_KEY_PATH" \
    -o BatchMode=yes \
    -o ConnectTimeout=10 \
    "${SSH_USER}@${SERVER_IP}" \
    "echo 'Connected'" &>/dev/null; then
    echo "✓ SSH connection successful"
else
    echo "✗ Cannot connect to server"
    exit 3
fi
```

**What are those flags?**  
- `-o BatchMode=yes` - No interactive prompts
- `-o ConnectTimeout=10` - Timeout after 10 seconds
- `&>/dev/null` - Hide output (we just want success/fail)

### Phase 5: Install Docker on Remote Server

```bash
ssh -i "$SSH_KEY_PATH" "${SSH_USER}@${SERVER_IP}" << 'ENDSSH'
    # Check if Docker is installed
    if ! command -v docker &> /dev/null; then
        echo "Installing Docker..."
        curl -fsSL https://get.docker.com | sh
        sudo usermod -aG docker $USER
        echo "Docker installed successfully"
    else
        echo "Docker already installed"
    fi
ENDSSH
```

**Heredoc Syntax (`<< 'ENDSSH'`):**  
Everything between `<< 'ENDSSH'` and `ENDSSH` runs on the remote server!

### Phase 6: Deploy the Application

```bash
# Transfer files to server
rsync -avz -e "ssh -i ${SSH_KEY_PATH}" \
    --exclude '.git' \
    --exclude 'node_modules' \
    app-directory/ "${SSH_USER}@${SERVER_IP}:/home/ubuntu/app/"

# Build and run Docker container
ssh -i "$SSH_KEY_PATH" "${SSH_USER}@${SERVER_IP}" << ENDSSH
    cd /home/ubuntu/app
    
    # Stop old container (if exists)
    docker stop my-app || true
    docker rm my-app || true
    
    # Build new image
    docker build -t my-app:latest .
    
    # Run container
    docker run -d \
        --name my-app \
        --restart unless-stopped \
        -p 3000:3000 \
        my-app:latest
    
    echo "Container started successfully"
ENDSSH
```

**Key Docker Commands:**  
- `docker build -t name .` - Build image from Dockerfile
- `docker run -d` - Run container in background (detached)
- `--restart unless-stopped` - Auto-restart if it crashes
- `-p 3000:3000` - Map port 3000 to host

### Phase 7: Configure Nginx

```bash
ssh -i "$SSH_KEY_PATH" "${SSH_USER}@${SERVER_IP}" << 'ENDSSH'
    # Create Nginx config
    sudo tee /etc/nginx/sites-available/my-app > /dev/null <<'EOF'
server {
    listen 80;
    server_name _;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

    # Enable site
    sudo ln -sf /etc/nginx/sites-available/my-app \
                /etc/nginx/sites-enabled/my-app
    
    # Remove default site
    sudo rm -f /etc/nginx/sites-enabled/default
    
    # Test and reload
    sudo nginx -t && sudo systemctl reload nginx
ENDSSH
```

### Phase 8: Validation

```bash
# Check if container is running
ssh -i "$SSH_KEY_PATH" "${SSH_USER}@${SERVER_IP}" \
    "docker ps | grep my-app" || {
    echo "✗ Container not running"
    exit 6
}

# Test application endpoint
if curl -sf "http://${SERVER_IP}" > /dev/null; then
    echo "✓ Application is accessible!"
else
    echo "⚠ Application not responding yet (check security groups)"
fi
```

---

## 🎨 Production-Grade Features

### 1. Comprehensive Logging

```bash
# Setup logging
LOG_FILE="deploy_$(date +%Y%m%d_%H%M%S).log"
exec 1> >(tee -a "$LOG_FILE")  # Log to file AND screen
exec 2>&1                       # Log errors too

echo "[INFO] Starting deployment..."
echo "[SUCCESS] Deployment complete!"
```

**Why log everything?**  
When something breaks at 3 AM, you'll thank yourself for having logs!

### 2. Error Handling

```bash
set -euo pipefail  # Exit on any error

# Custom error handler
trap 'echo "Error on line $LINENO"; exit 1' ERR

# Cleanup on exit
cleanup() {
    echo "Cleaning up..."
    rm -rf temp-files/
}
trap cleanup EXIT
```

**What this does:**  
- Script stops immediately if ANY command fails
- You know exactly where it failed
- Cleanup happens even if script crashes

### 3. Idempotency

**Idempotent** = Can run multiple times safely

```bash
# Stop old container BEFORE starting new one
docker stop my-app 2>/dev/null || true   # Don't fail if doesn't exist
docker rm my-app 2>/dev/null || true

# Now safe to start fresh
docker run -d --name my-app ...
```

**Why important?**  
You can run the script again to update your app. It won't create duplicates or break existing setup.

---

## 🧪 Testing the Script

### Local Testing

```bash
# Make script executable
chmod +x deploy.sh

# Test with dry-run (comment out destructive operations)
./deploy.sh
```

### What to Test

1. **Input Validation:**  
   - Try invalid IP addresses
   - Try wrong SSH key paths
   - Try empty inputs

2. **Error Handling:**  
   - Enter wrong PAT (should fail gracefully)
   - Use unreachable server (should timeout cleanly)
   - Remove Dockerfile (should detect and stop)

3. **Idempotency:**  
   - Run script twice in a row
   - Both should succeed without errors

---

## 📈 Real-World Impact

### Before Automation
```
Manual deployment: 25 minutes × 5 times/day = 125 minutes/day
Per week: 125 × 5 = 625 minutes (10+ hours!)
Per month: 40+ hours of repetitive work
```

### After Automation
```
Automated deployment: 1 command, 5 minutes
Script runs unattended
You can deploy while making coffee ☕
Monthly time saved: 35+ hours!
```

**That's almost a full work week saved every month!**

---

## 💡 Key Takeaways

1. **Automation saves time and reduces errors**  
   Humans make mistakes. Scripts don't (if written correctly).

2. **Validate everything**  
   Never trust user input. Check early, fail fast.

3. **Error handling is NOT optional**  
   Production systems need graceful failure handling.

4. **Idempotency matters**  
   Scripts should be safe to run multiple times.

5. **Logging is your friend**  
   When debugging at 2 AM, logs are invaluable.

---

## 🚀 What's Next?

Now that you understand automated deployment, you can:

1. **Add more features:**
   - SSL certificate automation (Let's Encrypt)
   - Health checks with retries
   - Rollback capability
   - Slack notifications

2. **Scale up:**
   - Deploy to multiple servers
   - Implement blue/green deployments
   - Add monitoring and alerts

3. **Integrate with CI/CD:**
   - GitHub Actions
   - GitLab CI
   - Jenkins

---

## 🎓 Learning Resources

- **Docker Documentation:** https://docs.docker.com/
- **Nginx Documentation:** https://nginx.org/en/docs/
- **Bash Scripting Guide:** https://www.gnu.org/software/bash/manual/
- **DevOps Roadmap:** https://roadmap.sh/devops
- **Github repo of the task:** https://github.com/delightverse/hng13-stage1-devops

---

## 🤝 Conclusion

Deployment automation isn't just about saving time—it's about **consistency, reliability, and enabling rapid iteration.**

Companies hire DevOps engineers specifically to build systems like this. You've now built something that mirrors production workflows at companies like Netflix, Amazon, and Google.

**Congratulations! You're thinking like a DevOps engineer!** 🎉

---

## 📝 About This Project

This article documents my learning journey during the HNG Internship DevOps track. The complete code is available on GitHub.

**Technologies Used:**
- Bash scripting
- Docker & Docker Compose
- Nginx reverse proxy
- Git automation
- SSH automation

**Connect with me:**
- GitHub: [@delightverse](https://github.com/delightverse)
- HNG Internship: [https://hng.tech](https://hng.tech)

---

*Built and written with ❤️ by: Ubah Delight Okechukwu - DevOps Engineer*