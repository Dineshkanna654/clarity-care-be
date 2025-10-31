# FastAPI Backend Deployment Guide
## Clarity Care - Medical Transcription Service

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Local Development Setup](#local-development-setup)
3. [AWS Configuration](#aws-configuration)
4. [Running the Server](#running-the-server)
5. [Testing](#testing)
6. [Production Deployment](#production-deployment)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software
- Python 3.10 or higher
- pip (Python package manager)
- AWS Account with appropriate permissions
- Git (optional, for version control)

### Required AWS Services
- AWS Transcribe Medical
- AWS Comprehend Medical
- IAM User with appropriate policies

---

## Local Development Setup

### 1. Create Project Directory
```bash
mkdir clarity-care-backend
cd clarity-care-backend
```

### 2. Create Virtual Environment
```bash
# Create virtual environment
python -m venv venv

# Activate virtual environment
# On macOS/Linux:
source venv/bin/activate

# On Windows:
venv\Scripts\activate
```

### 3. Install Dependencies
```bash
# Install required packages
pip install -r requirements.txt

# Verify installation
pip list
```

### 4. Set Up Environment Variables
```bash
# Copy example environment file
cp .env.example .env

# Edit .env with your credentials
nano .env  # or use your preferred editor
```

Edit `.env` file:
```env
AWS_ACCESS_KEY_ID=AKIAXXXXXXXXXXXXXXXX
AWS_SECRET_ACCESS_KEY=your_secret_key_here
AWS_REGION=us-east-1

API_HOST=0.0.0.0
API_PORT=8000
API_DEBUG=True

ALLOWED_ORIGINS=http://localhost:3000,http://127.0.0.1:3000
```

---

## AWS Configuration

### 1. Create IAM User

1. Go to AWS Console → IAM → Users
2. Click "Add users"
3. User name: `clarity-care-transcribe`
4. Select "Programmatic access"
5. Click "Next: Permissions"

### 2. Attach Policies

Attach the following managed policies:
- `TranscribeFullAccess`
- `ComprehendMedicalFullAccess`

Or create a custom policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "transcribe:StartMedicalStreamTranscription",
                "transcribe:StartMedicalTranscriptionJob",
                "transcribe:GetMedicalTranscriptionJob"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "comprehendmedical:DetectEntitiesV2",
                "comprehendmedical:DetectPHI"
            ],
            "Resource": "*"
        }
    ]
}
```

### 3. Save Credentials

Save the Access Key ID and Secret Access Key - you'll need these for the `.env` file.

### 4. Enable Services in Your Region

Ensure both services are available in your chosen region:
- AWS Transcribe Medical: Available in us-east-1, us-west-2, eu-west-1, ap-southeast-2
- AWS Comprehend Medical: Available in us-east-1, us-west-2, eu-west-1, ap-southeast-2

---

## Running the Server

### Development Mode

```bash
# Make sure virtual environment is activated
source venv/bin/activate  # or venv\Scripts\activate on Windows

# Run the server
python fastapi_backend.py
```

Or use uvicorn directly:
```bash
uvicorn fastapi_backend:app --reload --host 0.0.0.0 --port 8000
```

The server will start at: `http://localhost:8000`

### Check Server Status

Open your browser and navigate to:
- Health check: http://localhost:8000/health
- API docs: http://localhost:8000/docs
- Alternative docs: http://localhost:8000/redoc

---

## Testing

### 1. Test Health Endpoint

```bash
curl http://localhost:8000/health
```

Expected response:
```json
{
    "status": "healthy",
    "transcribe_region": "us-east-1",
    "services": {
        "transcribe": "available",
        "comprehend_medical": "available"
    }
}
```

### 2. Test NLP Extraction

```bash
curl http://localhost:8000/api/test-transcript
```

This will test the measurement extraction with a sample transcript.

### 3. Test WebSocket Connection (Python)

Create `test_websocket.py`:
```python
import asyncio
import websockets
import json

async def test_connection():
    uri = "ws://localhost:8000/ws/transcribe"
    
    async with websockets.connect(uri) as websocket:
        # Send configuration
        config = {
            "type": "config",
            "config": {
                "language_code": "en-US",
                "sample_rate": 16000,
                "media_encoding": "pcm",
                "specialty": "PRIMARYCARE",
                "type": "DICTATION"
            }
        }
        
        await websocket.send(json.dumps(config))
        
        # Receive response
        response = await websocket.recv()
        print(f"Received: {response}")
        
        # Send stop message
        await websocket.send(json.dumps({"type": "stop"}))

asyncio.run(test_connection())
```

Run the test:
```bash
python test_websocket.py
```

### 4. Test with iOS Client

Update the server URL in your iOS app:
```swift
let wsClient = WebSocketTranscriptionClient(
    serverURL: "ws://YOUR_SERVER_IP:8000/ws/transcribe"
)
```

For local testing on iPhone/iPad:
1. Get your computer's local IP address:
   - macOS: `ifconfig | grep "inet " | grep -v 127.0.0.1`
   - Windows: `ipconfig`
   - Linux: `ip addr show`

2. Use this IP in your iOS app:
   ```swift
   serverURL: "ws://192.168.1.100:8000/ws/transcribe"  // Replace with your IP
   ```

3. Ensure iPhone/iPad is on the same network

---

## Production Deployment

### Option 1: AWS EC2

#### 1. Launch EC2 Instance
```bash
# Ubuntu 22.04 LTS, t3.medium or larger
# Security Group: Allow ports 22 (SSH), 8000 (API), 443 (HTTPS)
```

#### 2. Connect and Setup
```bash
ssh -i your-key.pem ubuntu@your-ec2-ip

# Update system
sudo apt update && sudo apt upgrade -y

# Install Python 3.10
sudo apt install python3.10 python3.10-venv python3-pip -y

# Install supervisor (for process management)
sudo apt install supervisor -y
```

#### 3. Deploy Application
```bash
# Clone/upload your code
git clone your-repo-url
cd clarity-care-backend

# Create virtual environment
python3.10 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Setup environment variables
nano .env
```

#### 4. Configure Supervisor

Create `/etc/supervisor/conf.d/clarity-care.conf`:
```ini
[program:clarity-care]
directory=/home/ubuntu/clarity-care-backend
command=/home/ubuntu/clarity-care-backend/venv/bin/python -m uvicorn fastapi_backend:app --host 0.0.0.0 --port 8000
user=ubuntu
autostart=true
autorestart=true
stderr_logfile=/var/log/clarity-care.err.log
stdout_logfile=/var/log/clarity-care.out.log
environment=PATH="/home/ubuntu/clarity-care-backend/venv/bin"
```

Start the service:
```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start clarity-care
```

#### 5. Setup Nginx (Optional, for SSL)

```bash
sudo apt install nginx certbot python3-certbot-nginx -y
```

Create `/etc/nginx/sites-available/clarity-care`:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Enable and setup SSL:
```bash
sudo ln -s /etc/nginx/sites-available/clarity-care /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
sudo certbot --nginx -d your-domain.com
```

### Option 2: Docker Deployment

#### 1. Create Dockerfile

```dockerfile
FROM python:3.10-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY fastapi_backend.py .

# Expose port
EXPOSE 8000

# Run application
CMD ["uvicorn", "fastapi_backend:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### 2. Create docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - AWS_REGION=${AWS_REGION}
    restart: unless-stopped
```

#### 3. Deploy

```bash
# Build and run
docker-compose up -d

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

### Option 3: AWS Elastic Beanstalk

#### 1. Install EB CLI
```bash
pip install awsebcli
```

#### 2. Initialize EB Application
```bash
eb init -p python-3.10 clarity-care-api --region us-east-1
```

#### 3. Create Environment
```bash
eb create clarity-care-env
```

#### 4. Deploy
```bash
eb deploy
```

#### 5. Set Environment Variables
```bash
eb setenv AWS_ACCESS_KEY_ID=xxx AWS_SECRET_ACCESS_KEY=yyy AWS_REGION=us-east-1
```

### Option 4: AWS Lambda + API Gateway (Advanced)

For serverless deployment, you'll need to adapt the WebSocket handling or use AWS API Gateway WebSocket API.

---

## Monitoring and Logging

### 1. Application Logs

View logs in real-time:
```bash
# Supervisor logs
sudo tail -f /var/log/clarity-care.out.log
sudo tail -f /var/log/clarity-care.err.log

# Docker logs
docker-compose logs -f api
```

### 2. CloudWatch Integration (Optional)

Install CloudWatch agent:
```bash
pip install awscli-cwlogs
```

Configure logging to CloudWatch for production monitoring.

---

## Security Considerations

### 1. Environment Variables
- Never commit `.env` file to version control
- Use AWS Secrets Manager or Parameter Store for production
- Rotate credentials regularly

### 2. Network Security
- Use VPC security groups to restrict access
- Enable HTTPS/WSS in production
- Implement rate limiting

### 3. HIPAA Compliance (Future)
- Sign AWS BAA (Business Associate Agreement)
- Enable CloudTrail for audit logging
- Encrypt data in transit and at rest
- Implement proper authentication

### 4. API Authentication (Add to production)

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    token = credentials.credentials
    # Verify token (implement your logic)
    if not is_valid_token(token):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials"
        )
    return token

# Protect endpoints
@app.websocket("/ws/transcribe")
async def websocket_transcribe(
    websocket: WebSocket,
    token: str = Depends(verify_token)
):
    # ... rest of the code
```

---

## Troubleshooting

### Issue: WebSocket Connection Fails

**Solution:**
1. Check if server is running: `curl http://localhost:8000/health`
2. Verify firewall allows port 8000
3. Check AWS credentials in `.env`
4. Review logs for errors

### Issue: AWS Transcribe Returns 403 Error

**Solution:**
1. Verify IAM permissions include `transcribe:StartMedicalStreamTranscription`
2. Check if Transcribe Medical is available in your region
3. Ensure credentials are correct and not expired

### Issue: Audio Not Transcribing

**Solution:**
1. Verify audio format: 16kHz, mono, 16-bit PCM
2. Check audio chunk size (4096 samples recommended)
3. Test with sample audio file
4. Review AWS Transcribe console for job status

### Issue: NLP Extraction Not Working

**Solution:**
1. Test extraction endpoint: `curl http://localhost:8000/api/test-transcript`
2. Check if Comprehend Medical has proper permissions
3. Review transcript format and medical terminology
4. Adjust regex patterns in `extract_medical_data()` function

### Issue: High Latency

**Solution:**
1. Choose AWS region closest to your users
2. Increase buffer size for audio chunks
3. Optimize NLP processing frequency
4. Consider caching frequent patterns
5. Use EC2 instance with better network performance

---

## Performance Tuning

### 1. Audio Buffer Size
Adjust based on network latency:
```python
# In iOS client
bufferSize: 4096  # Larger = less frequent sends, more latency
bufferSize: 2048  # Smaller = more frequent sends, less latency
```

### 2. NLP Processing Frequency
```python
# In TranscriptionSession._receive_transcripts()
if len(self.transcript_buffer) >= 5:  # Adjust this threshold
    await self._process_nlp()
```

### 3. Connection Pool
```python
# In fastapi_backend.py
import boto3
from botocore.config import Config

config = Config(
    max_pool_connections=50,
    retries={'max_attempts': 3}
)

comprehend_medical_client = boto3.client(
    'comprehendmedical',
    region_name=COMPREHEND_REGION,
    config=config
)
```

---

## Backup and Recovery

### 1. Code Backup
```bash
# Git version control
git init
git add .
git commit -m "Initial commit"
git remote add origin your-repo-url
git push -u origin main
```

### 2. Configuration Backup
```bash
# Backup environment variables (encrypted)
tar -czf config-backup.tar.gz .env
openssl enc -aes-256-cbc -salt -in config-backup.tar.gz -out config-backup.tar.gz.enc
rm config-backup.tar.gz
```

### 3. Database (Future)
When adding database persistence, implement regular backups.

---

## Scaling Considerations

### Horizontal Scaling
- Use load balancer (AWS ALB or Nginx)
- Deploy multiple instances
- Implement sticky sessions for WebSocket connections

### Vertical Scaling
- Start with t3.medium (2 vCPU, 4 GB RAM)
- Scale up to t3.large or c5.xlarge for higher load
- Monitor CPU and memory usage

### Auto Scaling (AWS)
Configure auto-scaling group based on:
- CPU utilization > 70%
- Network throughput
- Active WebSocket connections

---

## Cost Optimization

### AWS Transcribe Medical Pricing (as of 2024)
- Streaming: $0.0300 per minute
- Standard: $0.0240 per minute

### AWS Comprehend Medical Pricing
- DetectEntities: $0.01 per 100 characters

### Estimated Costs (PoC)
For 100 hours of testing:
- Transcribe: $180
- Comprehend Medical: ~$50
- EC2 (t3.medium): ~$30/month
- **Total: ~$260 for PoC phase**

### Cost Reduction Tips
1. Use batch transcription when real-time not required
2. Process NLP only on final transcripts, not partial
3. Cache common medical terms and patterns
4. Use AWS Free Tier when available
5. Stop EC2 instances when not in use

---

## Next Steps

1. ✅ Deploy backend server
2. ✅ Test WebSocket connection
3. ✅ Integrate with iOS client
4. ⬜ Add authentication
5. ⬜ Implement database persistence
6. ⬜ Add comprehensive error handling
7. ⬜ Setup monitoring and alerts
8. ⬜ Conduct load testing
9. ⬜ Implement HIPAA compliance
10. ⬜ Production deployment

---

## Support Resources

- AWS Transcribe Medical: https://docs.aws.amazon.com/transcribe/latest/dg/medical-start-stream.html
- AWS Comprehend Medical: https://docs.aws.amazon.com/comprehend-medical/latest/dev/what-is.html
- FastAPI Documentation: https://fastapi.tiangolo.com/
- WebSocket Protocol: https://datatracker.ietf.org/doc/html/rfc6455

---

## Contact

For issues or questions:
- Check logs first
- Review AWS console for service errors
- Test with sample audio/transcript
- Verify all credentials and permissions

**Document Version**: 1.0  
**Last Updated**: November 2025  
**Project**: Clarity Care Backend Deployment
