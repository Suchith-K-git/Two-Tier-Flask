# What is Amazon EKS?

Amazon Elastic Kubernetes Service (EKS) is a managed Kubernetes service provided by AWS that allows you to run Kubernetes clusters without managing the Kubernetes control plane.

## IAM Role Requirement

Before creating an EKS cluster, create an IAM User or IAM Role with the required permissions and configure AWS credentials using `aws configure`.

---

# Install AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
```

---

# Install kubectl

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

---

# Install eksctl

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Docker Application with Route 53 and Nginx Reverse Proxy

## Architecture

```text
                         Internet
                            |
                            v
                    www.suchith.site
                            |
                            v
                        Route 53
                            |
                            v
                     18.179.84.79
                            |
                         Port 80
                            |
                            v
                          Nginx
                            |
                     Proxy to Port 5000
                            |
                            v
                    Docker Container
                            |
                            v
                    Application :5000
```

---

# 1. Prerequisites

Make sure you have:

* AWS EC2 instance
* Docker installed
* Application running inside a Docker container
* Route 53 hosted zone
* Domain name: `suchith.site`

The application should be accessible using:

```bash
http://18.179.84.79:5000
```

---

# 2. Docker Application

Check whether your container is running:

```bash
docker ps
```

You should see a port mapping similar to:

```text
0.0.0.0:5000->5000/tcp
```

This means:

```text
EC2 Port 5000
      |
      v
Container Port 5000
```

Test the application:

```bash
curl http://localhost:5000
```

You can also test from your browser:

```text
http://18.179.84.79:5000
```

---

# 3. Configure Route 53

Go to:

```text
AWS Console
    ↓
Route 53
    ↓
Hosted Zones
    ↓
suchith.site
```

Create an **A Record**.

Use:

```text
Record name: www
Record type: A
Value: 18.179.84.79
```

### Important

Do NOT enter:

```text
http://18.179.84.79
```

Do NOT enter:

```text
18.179.84.79:5000
```

The correct value is only:

```text
18.179.84.79
```

The DNS flow is:

```text
www.suchith.site
        |
        v
18.179.84.79
```

Verify DNS:

```bash
nslookup www.suchith.site
```

Expected result:

```text
Name:    www.suchith.site
Address: 18.179.84.79
```

---

# 4. Install Nginx

Nginx will act as a **Reverse Proxy**.

Install Nginx:

```bash
sudo apt update
sudo apt install nginx -y
```

Check the status:

```bash
sudo systemctl status nginx
```

If required, start Nginx:

```bash
sudo systemctl start nginx
```

Enable Nginx to start automatically after reboot:

```bash
sudo systemctl enable nginx
```

---

# 5. Configure Nginx

Create a new Nginx configuration:

```bash
sudo nano /etc/nginx/sites-available/suchith
```

Add:

```nginx
server {
    listen 80;
    server_name www.suchith.site;

    location / {
        proxy_pass http://127.0.0.1:5000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Save the file:

```text
CTRL + O
Enter
CTRL + X
```

---

# 6. Enable Nginx Configuration

Create a symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/suchith /etc/nginx/sites-enabled/
```

Remove the default Nginx configuration:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

# 7. Test Nginx Configuration

Before restarting Nginx, test the configuration:

```bash
sudo nginx -t
```

Expected output:

```text
syntax is ok
test is successful
```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

---

# 8. Configure EC2 Security Group

Go to:

```text
AWS Console
    ↓
EC2
    ↓
Instances
    ↓
Your Instance
    ↓
Security
    ↓
Security Groups
    ↓
Inbound Rules
```

Add:

```text
Type: HTTP
Protocol: TCP
Port: 80
Source: 0.0.0.0/0
```

During testing, you can keep port `5000` open:

```text
Type: Custom TCP
Protocol: TCP
Port: 5000
Source: 0.0.0.0/0
```

---

# 9. How the Request Works

When the user enters:

```text
http://www.suchith.site
```

the request follows this path:

```text
Browser
   |
   v
www.suchith.site
   |
   v
Route 53
   |
   v
18.179.84.79
   |
   v
Port 80
   |
   v
Nginx
   |
   v
127.0.0.1:5000
   |
   v
Docker Container
   |
   v
Application
```

The user does NOT need to specify:

```text
:5000
```

---

# 10. Final URLs

### Before Nginx

You access the application using:

```text
http://18.179.84.79:5000
```

or:

```text
http://www.suchith.site:5000
```

### After Nginx

You can access it using:

```text
http://www.suchith.site
```

Nginx receives the request on port `80` and forwards it to your Docker application on port `5000`.

---

# 11. Docker + Nginx Port Flow

```text
                  EC2 INSTANCE
              18.179.84.79
                     |
                     |
                  Port 80
                     |
                     v
                  NGINX
                     |
                     |
              Proxy to 5000
                     |
                     v
                  Port 5000
                     |
                     v
              Docker Container
                     |
                     v
             Application :5000
```

Docker does NOT need to run Nginx.

Nginx is running directly on the **EC2 host**, while your application is running inside the **Docker container**.

---

# 12. Verify Everything

### Check Docker

```bash
docker ps
```

Expected:

```text
0.0.0.0:5000->5000/tcp
```

### Check application

```bash
curl http://localhost:5000
```

### Check Nginx

```bash
sudo systemctl status nginx
```

### Check Nginx configuration

```bash
sudo nginx -t
```

### Check DNS

```bash
nslookup www.suchith.site
```

Expected:

```text
18.179.84.79
```

### Test from browser

Open:

```text
http://www.suchith.site
```

---

# 13. After Everything Works

Once Nginx is working, you can remove public access to port `5000` from the EC2 Security Group.

Users will access:

```text
http://www.suchith.site
```

instead of:

```text
http://www.suchith.site:5000
```

The application can continue running on:

```text
localhost:5000
```

and Nginx will act as the entry point.

---

# Summary

```text
Route 53
   |
   | www.suchith.site
   v
18.179.84.79
   |
   | Port 80
   v
Nginx
   |
   | Proxy
   v
Port 5000
   |
   v
Docker Container
   |
   v
Application
```

### Main Commands

```bash
# Install Nginx
sudo apt update
sudo apt install nginx -y

# Create configuration
sudo nano /etc/nginx/sites-available/suchith

# Enable configuration
sudo ln -s /etc/nginx/sites-available/suchith /etc/nginx/sites-enabled/

# Remove default configuration
sudo rm /etc/nginx/sites-enabled/default

# Test configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx

# Check Nginx
sudo systemctl status nginx

# Check Docker
docker ps

# Test application
curl http://localhost:5000

# Check DNS
nslookup www.suchith.site
```


# Create EKS Cluster

```bash
eksctl create cluster --name three-tier-cluster --region us-west-2 --node-type t2.medium --nodes-min 2 --nodes-max 2
```
