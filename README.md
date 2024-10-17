# Node.js CI/CD with GitHub Actions and EC2 - Complete Implementation Guide

## Table of Contents
- [Prerequisites](#prerequisites)
- [1. Local Development Setup](#1-local-development-setup)
- [2. GitHub Repository Setup](#2-github-repository-setup)
- [3. EC2 Instance Configuration](#3-ec2-instance-configuration)
- [4. SSH Key Setup](#4-ssh-key-setup)
- [5. GitHub Actions Configuration](#5-github-actions-configuration)
- [6. Environment Management](#6-environment-management)
- [7. Security Setup](#7-security-setup)
- [8. Monitoring & Maintenance](#8-monitoring--maintenance)
- [9. Troubleshooting Guide](#9-troubleshooting-guide)

## Prerequisites

Before starting, ensure you have:
- AWS Account with appropriate permissions
- GitHub Account
- Node.js and npm installed locally
- Basic Git knowledge
- SSH client installed

## 1. Local Development Setup

### Project Structure
```
your-nodejs-app/
├── .git/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── src/
├── .gitignore
├── package.json
├── ecosystem.config.js
└── README.md
```

### Create .gitignore
```
# Node.js
node_modules/
npm-debug.log
yarn-debug.log
yarn-error.log

# Environment variables
.env
.env.*
!.env.example

# Build output
dist/
build/
*.build/

# IDE specific files
.idea/
.vscode/
*.swp
*.swo

# OS specific files
.DS_Store
Thumbs.db

# Logs
logs
*.log

# Testing
coverage/

# PM2
ecosystem.config.js
```

## 2. GitHub Repository Setup

### Initialize Repository
```bash
# Initialize git
git init

# Add files
git add .

# Initial commit
git commit -m "Initial project setup"

# Add remote repository
git remote add origin git@github.com:username/your-repo.git

# Push to main branch
git push -u origin main
```

### Configure Repository Settings
1. Go to repository Settings
2. Enable branch protection rules for 'main'
3. Configure required status checks
4. Set up required reviews for pull requests

## 3. EC2 Instance Configuration

### Launch EC2 Instance
1. Navigate to AWS Console → EC2
2. Launch new instance:
   - Choose Ubuntu Server 22.04 LTS
   - Select instance type (t2.micro for testing)
   - Configure Security Group

### Security Group Configuration
```
Inbound Rules:
- SSH (22): Your IP
- HTTP (80): 0.0.0.0/0
- HTTPS (443): 0.0.0.0/0
- Application Port: 0.0.0.0/0
```

### Install Required Software
```bash
#!/bin/bash

# Update system packages
sudo apt update
sudo apt upgrade -y

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install npm and development tools
sudo apt install -y npm build-essential

# Install PM2
sudo npm install -g pm2

# Install Git
sudo apt install -y git

# Optional: Install Nginx for reverse proxy
sudo apt install -y nginx
```

### Setup Application Directory
```bash
# Create application directory
sudo mkdir -p /var/www/your-app-name
sudo chown -R ubuntu:ubuntu /var/www/your-app-name

# Initialize directory
cd /var/www/your-app-name
git init
```

### Configure PM2
Create ecosystem.config.js:
```javascript
module.exports = {
  apps: [{
    name: "your-app-name",
    script: "./src/index.js",
    instances: "max",
    exec_mode: "cluster",
    autorestart: true,
    watch: false,
    max_memory_restart: "1G",
    env: {
      NODE_ENV: "production",
      PORT: 3000
    }
  }]
}
```

## 4. SSH Key Setup

### Generate Deployment Keys
```bash
# On your local machine
ssh-keygen -t rsa -b 4096 -C "deployment-key" -f ./deploy_key

# This will create:
# - deploy_key (private key)
# - deploy_key.pub (public key)
```

### Configure EC2 SSH Access
```bash
# On EC2 instance
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Add deploy_key.pub content to authorized_keys
echo "your-public-key-content" >> ~/.ssh/authorized_keys
```

### Add GitHub Secrets
Add these secrets in GitHub repository:
- `SSH_PRIVATE_KEY`: Content of deploy_key
- `HOST`: EC2 public IP/DNS
- `USERNAME`: EC2 username (usually 'ubuntu')
- `PM2_APP_NAME`: Your application name

## 5. GitHub Actions Configuration

### Create Workflow File
Create `.github/workflows/deploy.yml`:
```yaml
name: Node.js CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  NODE_ENV: production

jobs:
  test-and-deploy:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18.x]
        
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install Dependencies
      run: npm ci
    
    - name: Run Tests
      run: npm test
      if: success()
    
    - name: Build Application
      run: npm run build --if-present
      if: success()
    
    - name: Setup SSH
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy-key
        chmod 600 ~/.ssh/deploy-key
        ssh-keyscan -H ${{ secrets.HOST }} >> ~/.ssh/known_hosts
    
    - name: Deploy to EC2
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: |
        ssh -i ~/.ssh/deploy-key ${{ secrets.USERNAME }}@${{ secrets.HOST }} '
          cd /var/www/${{ secrets.PM2_APP_NAME }} && \
          git pull origin main && \
          npm ci --production && \
          npm run build --if-present && \
          pm2 reload ecosystem.config.js --env production || \
          pm2 start ecosystem.config.js --env production'
    
    - name: Health Check
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: |
        sleep 10
        curl --fail http://${{ secrets.HOST }}:3000/health || exit 1
```

## 6. Environment Management

### Environment Variables
Create `.env.example`:
```
NODE_ENV=production
PORT=3000
DATABASE_URL=mongodb://localhost:27017/your-db
API_KEY=your-api-key
```

### Add Production Secrets
Add to GitHub repository secrets:
- `NODE_ENV`
- `DATABASE_URL`
- `API_KEY`
- Other application-specific variables

## 7. Security Setup

### EC2 Security
1. Update security groups
2. Enable AWS CloudWatch
3. Setup AWS CloudTrail
4. Configure Network ACLs

### Application Security
1. Install security packages:
```bash
npm install helmet
npm install express-rate-limit
npm install cors
```

2. Configure security middleware:
```javascript
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const cors = require('cors');

app.use(helmet());
app.use(cors());
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100
}));
```

## 8. Monitoring & Maintenance

### Setup PM2 Monitoring
```bash
# Install PM2 logrotate module
pm2 install pm2-logrotate

# Configure log rotation
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7

# Setup PM2 startup script
pm2 startup systemd
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

### Configure Application Logging
1. Install Winston:
```bash
npm install winston
```

2. Setup logging configuration:
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});
```

## 9. Troubleshooting Guide

### Common Issues and Solutions

#### Deployment Failures
```bash
# Check GitHub Actions logs
# View deployment status in Actions tab

# Check SSH connectivity
ssh -i deploy_key username@host

# Verify PM2 status
pm2 list
pm2 logs

# Check application logs
tail -f /var/log/nginx/error.log
journalctl -u nginx.service
```

#### Application Issues
```bash
# Check Node.js version
node --version

# Verify npm packages
npm list

# Check disk space
df -h

# Monitor memory usage
free -m
top

# View PM2 metrics
pm2 monit
```

### Maintenance Commands
```bash
# Update system packages
sudo apt update
sudo apt upgrade -y

# Clean npm cache
npm cache clean --force

# Update npm packages
npm update

# Rotate logs
pm2 flush
pm2 reloadLogs

# Backup database (if using MongoDB)
mongodump --out /backup/$(date +%Y%m%d)
```

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [PM2 Documentation](https://pm2.keymetrics.io/docs/usage/quick-start/)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)

## Contributors
- Your Name
- Your Email

## License
MIT License (or your chosen license)
