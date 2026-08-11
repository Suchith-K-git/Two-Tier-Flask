 
# Flask App with MySQL Docker Setup

This is a simple Flask app that interacts with a MySQL database. The app allows users to submit messages, which are then stored in the database and displayed on the frontend.

## Prerequisites

Before you begin, make sure you have the following installed:

- Docker
- Git (optional, for cloning the repository)

## Setup

1. Clone this repository (if you haven't already):

   ```bash
   git clone https://github.com/your-username/your-repo-name.git
   ```

2. Navigate to the project directory:

   ```bash
   cd your-repo-name
   ```

3. Create a `.env` file in the project directory to store your MySQL environment variables:

   ```bash
   touch .env
   ```

4. Open the `.env` file and add your MySQL configuration:

   ```
   MYSQL_HOST=mysql
   MYSQL_USER=your_username
   MYSQL_PASSWORD=your_password
   MYSQL_DB=your_database
   ```

## Usage

1. Start the containers using Docker Compose:

   ```bash
   docker-compose up --build
   ```

2. Access the Flask app in your web browser:

   - Frontend: http://localhost
   - Backend: http://localhost:5000

3. Create the `messages` table in your MySQL database:

   - Use a MySQL client or tool (e.g., phpMyAdmin) to execute the following SQL commands:
   
     ```sql
     CREATE TABLE messages (
         id INT AUTO_INCREMENT PRIMARY KEY,
         message TEXT
     );
     ```

4. Interact with the app:

   - Visit http://localhost to see the frontend. You can submit new messages using the form.
   - Visit http://localhost:5000/insert_sql to insert a message directly into the `messages` table via an SQL query.

## Cleaning Up

To stop and remove the Docker containers, press `Ctrl+C` in the terminal where the containers are running, or use the following command:

```bash
docker-compose down
```

## To run this two-tier application using  without docker-compose

- First create a docker image from Dockerfile
```bash
docker build -t flaskapp .
```

- Now, make sure that you have created a network using following command
```bash
docker network create twotier
```

- Attach both the containers in the same network, so that they can communicate with each other

i) MySQL container 
```bash
docker run -d \
    --name mysql \
    -v mysql-data:/var/lib/mysql \
    --network=twotier \
    -e MYSQL_DATABASE=mydb \
    -e MYSQL_ROOT_PASSWORD=admin \
    -p 3306:3306 \
    mysql:5.7

```
ii) Backend container
```bash
docker run -d \
    --name flaskapp \
    --network=twotier \
    -e MYSQL_HOST=mysql \
    -e MYSQL_USER=root \
    -e MYSQL_PASSWORD=admin \
    -e MYSQL_DB=mydb \
    -p 5000:5000 \
    flaskapp:latest

```

## Notes

- Make sure to replace placeholders (e.g., `your_username`, `your_password`, `your_database`) with your actual MySQL configuration.

- This is a basic setup for demonstration purposes. In a production environment, you should follow best practices for security and performance.

- Be cautious when executing SQL queries directly. Validate and sanitize user inputs to prevent vulnerabilities like SQL injection.

- If you encounter issues, check Docker logs and error messages for troubleshooting.

```

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
