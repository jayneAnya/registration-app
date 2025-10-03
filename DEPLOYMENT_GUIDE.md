# Registration App - Complete Deployment Guide

This comprehensive guide covers building and deploying the Registration App both locally and through GitHub Actions to AWS EC2.

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Local Development Setup](#local-development-setup)
3. [GitHub Actions Deployment](#github-actions-deployment)
4. [Common Issues & Solutions](#common-issues--solutions)
5. [AWS Configuration](#aws-configuration)
6. [Troubleshooting](#troubleshooting)

## 🏗️ Project Overview

A full-stack user registration and authentication application with:
- **Frontend**: React 18 with Material-UI
- **Backend**: Python FastAPI with JWT authentication
- **Database**: PostgreSQL (configured but using in-memory storage)
- **Deployment**: Docker containers on AWS EC2
- **CI/CD**: GitHub Actions with Amazon ECR

### Architecture
- **Frontend**: Port 9090 (Nginx serving React build)
- **Backend**: Port 9091 (FastAPI with CORS enabled)
- **Database**: Port 9092 (PostgreSQL - not currently used)

## 🛠️ Local Development Setup

### Prerequisites
- Docker and Docker Compose
- Node.js 18+ (for local development)
- Python 3.9+ (for local development)
- Git

### 1. Clone Repository
```bash
git clone https://github.com/emanet1/registration-app.git
cd registration-app
```

### 2. Environment Configuration

Create a `.env` file in the root directory:
```env
# JWT Secret Key for token signing
JWT_SECRET_KEY=your-secret-key-change-in-production

# Optional: Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=magax
```

### 3. Docker Development Setup

#### Option A: Full Stack with Docker Compose (Recommended)
```bash
# Build and start all services
docker-compose up --build

# Access the application
# Frontend: http://localhost:9090
# Backend API: http://localhost:9091
# Database: localhost:9092
```

#### Option B: Individual Container Development
```bash
# Backend only
cd backend
docker build -t reg-backend .
docker run -d --name reg-backend -p 8000:8000 reg-backend

# Frontend only
cd frontend
docker build --build-arg REACT_APP_API_URL=http://localhost:8000 -t reg-frontend .
docker run -d --name reg-frontend -p 3000:3000 reg-frontend
```

### 4. Local Development (Without Docker)

#### Backend Setup
```bash
cd backend
pip install -r requirements.txt
python -m uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

#### Frontend Setup
```bash
cd frontend
npm install
# Set environment variable
export REACT_APP_API_URL=http://localhost:8000
npm start
```

### 5. Testing Local Setup

#### Backend API Tests
```bash
# Health check
curl http://localhost:9091/api/health

# Register user
curl -X POST http://localhost:9091/api/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","email":"test@example.com","password":"testpass"}'

# Login
curl -X POST http://localhost:9091/api/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=testuser&password=testpass"
```

#### Frontend Access
- Open http://localhost:9090 in your browser
- Test registration and login functionality
- Check browser developer tools for any console errors

## 🚀 GitHub Actions Deployment

### Prerequisites for GitHub Actions
1. AWS Account with ECR repository
2. EC2 instance with Docker installed
3. GitHub repository with secrets configured

### 1. AWS Setup

#### Create ECR Repository
```bash
# Create ECR repositories
aws ecr create-repository --repository-name registration-frontend
aws ecr create-repository --repository-name registration-backend
```

#### Launch EC2 Instance
- **Instance Type**: t3.medium (minimum)
- **OS**: Ubuntu 20.04 LTS
- **Storage**: 20GB minimum
- **Security Groups**: Ports 22 (SSH), 9090 (Frontend), 9091 (Backend)

#### Install Docker on EC2
```bash
# Update system
sudo apt update

# Install Docker
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt install -y docker-compose-plugin
# OR install legacy version
sudo apt install -y docker-compose

# Logout and login to apply group changes
```

### 2. GitHub Secrets Configuration

Configure the following secrets in your GitHub repository:

#### Required Secrets
```
AWS_ACCOUNT_ID=123456789012
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
EC2_HOST=3.89.71.89
EC2_USER=ubuntu
EC2_SSH_KEY=-----BEGIN OPENSSH PRIVATE KEY-----...
REACT_APP_API_URL=http://3.89.71.89:9091
BACKEND_SECRET_KEY=your-production-secret-key
```

#### How to Set GitHub Secrets
1. Go to your GitHub repository
2. Click Settings → Secrets and variables → Actions
3. Click "New repository secret"
4. Add each secret with its corresponding value

### 3. GitHub Actions Workflow

The deployment uses `.github/workflows/demo-secrets-good.yml` which:

1. **Builds and pushes Docker images** to Amazon ECR
2. **Copies docker-compose.ecr.yml** to EC2 instance
3. **Deploys application** using Docker Compose
4. **Sets up environment variables** on EC2

#### Workflow Triggers
- Push to main branch
- Manual workflow dispatch

#### Key Workflow Steps
```yaml
# Build and push frontend
- name: Build and push frontend
  uses: docker/build-push-action@v6
  with:
    context: frontend
    push: true
    tags: ${{ env.ECR_REGISTRY }}/registration-frontend:v1
    build-args: REACT_APP_API_URL=${{ secrets.REACT_APP_API_URL }}

# Build and push backend
- name: Build and push backend
  uses: docker/build-push-action@v6
  with:
    context: backend
    push: true
    tags: ${{ env.ECR_REGISTRY }}/registration-backend:v1

# Deploy to EC2
- name: Deploy to EC2
  uses: appleboy/ssh-action@v1.0.3
  with:
    host: ${{ secrets.EC2_HOST }}
    username: ${{ secrets.EC2_USER }}
    key: ${{ secrets.EC2_SSH_KEY }}
    script: |
      cd ~/app
      sudo docker compose -f docker-compose.ecr.yml pull
      sudo docker compose -f docker-compose.ecr.yml up -d
```

### 4. Deployment Process

#### Automatic Deployment
1. Push code to main branch
2. GitHub Actions automatically triggers
3. Images are built and pushed to ECR
4. Application is deployed to EC2

#### Manual Deployment
1. Go to GitHub Actions tab
2. Select "Demo - Using Secrets (GOOD PRACTICE)" workflow
3. Click "Run workflow"
4. Select branch and click "Run workflow"

### 5. Post-Deployment Verification

#### Check EC2 Services
```bash
# SSH into EC2 instance
ssh -i your-key.pem ubuntu@3.89.71.89

# Check running containers
docker ps

# Check logs
docker logs registration-app-frontend-1
docker logs registration-app-backend-1

# Test API
curl http://localhost:9091/api/health
```

#### Access Application
- **Frontend**: http://3.89.71.89:9090
- **Backend API**: http://3.89.71.89:9091
- **API Documentation**: http://3.89.71.89:9091/docs

## ⚠️ Common Issues & Solutions

### Issue 1: Frontend Can't Connect to Backend

#### Symptoms
```
POST http://3.89.71.89:9091/api/register net::ERR_CONNECTION_TIMED_OUT
```

#### Root Causes & Solutions

**1. Missing HTTP Protocol**
```javascript
// ❌ Wrong
const API_URL = "3.89.71.89:9091"

// ✅ Correct
const API_URL = "http://3.89.71.89:9091"
```

**2. CORS Configuration Mismatch**
```python
# backend/main.py
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://3.89.71.89:9090", "http://localhost:9090"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**3. Security Group Port Access**
- Ensure EC2 security group allows:
  - Port 9090 (Frontend) - Source: 0.0.0.0/0
  - Port 9091 (Backend) - Source: 0.0.0.0/0

### Issue 2: Environment Variable Problems

#### Symptoms
```
Error: REACT_APP_API_URL is not set
```

#### Solutions

**1. Docker Compose Environment**
```yaml
# docker-compose.yml
environment:
  - REACT_APP_API_URL=http://3.89.71.89:9091  # Include http://
```

**2. Build Arguments**
```yaml
# docker-compose.yml
build:
  context: ./frontend
  args:
    - REACT_APP_API_URL=http://3.89.71.89:9091  # Include http://
```

### Issue 3: GitHub Actions Deployment Failures

#### Common Error: SSH Connection Failed
```bash
Error: ssh: connect to host 3.89.71.89 port 22: Connection timed out
```

#### Solutions
1. **Check EC2 Security Group**: Ensure port 22 is open
2. **Verify SSH Key**: Ensure EC2_SSH_KEY secret is correctly formatted
3. **Check EC2 Status**: Ensure instance is running
4. **Verify IP Address**: Ensure EC2_HOST secret has correct IP

#### Common Error: ECR Login Failed
```bash
Error: An error occurred (AccessDenied) when calling the GetAuthorizationToken operation
```

#### Solutions
1. **Check AWS Credentials**: Verify AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY
2. **Check IAM Permissions**: Ensure user has ECR permissions
3. **Check AWS Region**: Ensure AWS_REGION matches ECR repository region

### Issue 4: Docker Build Failures

#### Frontend Build Issues
```bash
# Check build arguments
docker build --build-arg REACT_APP_API_URL=http://3.89.71.89:9091 -t frontend .
```

#### Backend Build Issues
```bash
# Check Python dependencies
docker build -t backend .
```

### Issue 5: Container Startup Issues

#### Check Container Logs
```bash
# View all logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f frontend
docker-compose logs -f backend
```

#### Common Container Issues
1. **Port Conflicts**: Ensure ports 9090 and 9091 are not in use
2. **Memory Issues**: Increase EC2 instance size if needed
3. **Network Issues**: Check Docker network configuration

## 🔧 AWS Configuration

### EC2 Security Group Rules

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| SSH | TCP | 22 | Your IP | SSH access |
| HTTP | TCP | 9090 | 0.0.0.0/0 | Frontend access |
| HTTP | TCP | 9091 | 0.0.0.0/0 | Backend API access |

### ECR Repository Configuration

```bash
# Create repositories
aws ecr create-repository --repository-name registration-frontend --region us-east-1
aws ecr create-repository --repository-name registration-backend --region us-east-1

# Get login token
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

### IAM User Permissions

Required permissions for GitHub Actions:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:PutImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload"
            ],
            "Resource": "*"
        }
    ]
}
```

## 🔍 Troubleshooting

### 1. Check Application Status

#### EC2 Instance
```bash
# SSH into instance
ssh -i your-key.pem ubuntu@3.89.71.89

# Check Docker services
sudo docker ps
sudo docker-compose ps

# Check application logs
sudo docker-compose logs -f
```

#### Network Connectivity
```bash
# Test frontend
curl -I http://3.89.71.89:9090

# Test backend
curl -I http://3.89.71.89:9091/api/health

# Test from EC2 instance
curl -I http://localhost:9090
curl -I http://localhost:9091/api/health
```

### 2. Debug GitHub Actions

#### Check Workflow Logs
1. Go to GitHub Actions tab
2. Click on failed workflow run
3. Expand each step to see detailed logs
4. Look for specific error messages

#### Common Debug Commands
```bash
# Check ECR repositories
aws ecr describe-repositories --region us-east-1

# Check EC2 instance status
aws ec2 describe-instances --instance-ids i-1234567890abcdef0

# Test SSH connection
ssh -i your-key.pem ubuntu@3.89.71.89 "echo 'SSH working'"
```

### 3. Application Debugging

#### Frontend Issues
```javascript
// Check browser console for errors
// Verify API_URL is correct
console.log('API_URL:', process.env.REACT_APP_API_URL);

// Test API connectivity
fetch('http://3.89.71.89:9091/api/health')
  .then(response => response.json())
  .then(data => console.log('Backend response:', data));
```

#### Backend Issues
```python
# Add debug logging in main.py
import logging
logging.basicConfig(level=logging.DEBUG)

# Test CORS
print("CORS origins:", ["http://3.89.71.89:9090", "http://localhost:9090"])
```

## 📚 Additional Resources

### Documentation
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [React Documentation](https://reactjs.org/docs)
- [Docker Documentation](https://docs.docker.com/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

### Useful Commands

#### Docker Management
```bash
# Clean up containers and images
docker system prune -a

# View container resource usage
docker stats

# Execute commands in running container
docker exec -it container_name sh
```

#### AWS Management
```bash
# List ECR repositories
aws ecr describe-repositories

# List EC2 instances
aws ec2 describe-instances

# View CloudWatch logs
aws logs describe-log-groups
```

## 🎯 Best Practices

### Security
1. **Use Environment Variables**: Never hardcode sensitive data
2. **Rotate Secrets**: Regularly update JWT secrets and API keys
3. **Limit Permissions**: Use least privilege principle for AWS IAM
4. **Monitor Access**: Enable CloudTrail for AWS API monitoring

### Development
1. **Version Control**: Use semantic versioning for releases
2. **Testing**: Implement unit and integration tests
3. **Code Review**: Require pull request reviews
4. **Documentation**: Keep deployment guides updated

### Deployment
1. **Blue-Green Deployment**: Consider zero-downtime deployments
2. **Health Checks**: Implement proper health check endpoints
3. **Monitoring**: Set up application monitoring and alerting
4. **Backup Strategy**: Implement database and configuration backups

## 📞 Support

For issues and questions:
1. Check this documentation first
2. Review GitHub Issues
3. Check application logs
4. Contact the development team

---

**Last Updated**: December 2024
**Version**: 1.0.0
