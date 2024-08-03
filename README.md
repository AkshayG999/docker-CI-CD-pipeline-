
# Docker CI/CD Pipeline for Node.js Application

This guide provides a comprehensive, step-by-step process to set up a CI/CD pipeline for a Node.js application using Docker, GitHub Actions, AWS EC2, and Nginx with SSL.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [AWS EC2 Setup](#aws-ec2-setup)
3. [Docker Setup on EC2](#docker-setup-on-ec2)
4. [GitHub Actions Setup](#github-actions-setup)
5. [Application Deployment](#application-deployment)
6. [Nginx and SSL Configuration](#nginx-and-ssl-configuration)
7. [Troubleshooting](#troubleshooting)

## Prerequisites

- AWS account
- GitHub account
- Docker Hub account
- Node.js application repository on GitHub

## AWS EC2 Setup

1. Launch an EC2 instance:
   - Use an Ubuntu Server AMI
   - Choose an instance type (e.g., t2.micro for free tier)
   - Configure security groups to allow SSH (port 22), HTTP (port 80), and HTTPS (port 443)
   - Create and download a key pair

2. Connect to your EC2 instance:
   ```bash
   ssh -i /path/to/your-key-pair.pem ubuntu@your-ec2-instance-public-ip
## Docker Setup on EC2

1.  Update package repository and install Docker:
       
     ```bash
    sudo  apt-get update sudo  apt-get  install -y docker.io
  ###
2.  Start Docker and enable it to start on boot:
	  ```bash
	  sudo systemctl start docker sudo systemctl enable  docker
    
3.  Add the ubuntu user to the docker group:
    ```bash
    sudo  usermod -aG docker ubuntu
  ###    
4.  Reboot the instance:
	  ```bash
    sudo  reboot
  ###    
5.  After reboot, reconnect and verify Docker access:
     ```bash
    docker --version docker  ps
  ##    

## GitHub Actions Setup

1.  In your GitHub repository, create a `.github/workflows` directory:
    ```bash
    mkdir -p .github/workflows
  ###    


2.  Create a workflow file named `main.yml` in the `.github/workflows` directory:

	2. Create a workflow file (e.g., `main.yml`) in the workflows directory:
```
		name: CI/CD Pipeline
		on:
		  push:
		    branches:
		      - main
		jobs:
		  build:
		    runs-on: ubuntu-latest
		    steps:
		      - name: Checkout code
		        uses: actions/checkout@v2
		      
		      - name: Set up Node.js
		        uses: actions/setup-node@v2
		        with:
		          node-version: '20'
		      
		      - name: Install dependencies
		        run: npm install
		      
		      - name: Build Docker image
		        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/best_bus_backend:latest .
		      
		      - name: Log in to DockerHub
		        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
		      
		      - name: Push Docker image to DockerHub
		        run: docker push ${{ secrets.DOCKER_USERNAME }}/best_bus_backend:latest
		      
		      - name: SSH to EC2 and Deploy
		        uses: appleboy/ssh-action@master
		        with:
		          host: ${{ secrets.EC2_HOST }}
		          username: ubuntu
		          key: ${{ secrets.EC2_KEY }}
		          script: |
		            docker pull ${{ secrets.DOCKER_USERNAME }}/best_bus_backend:latest
		            docker stop $(docker ps -a -q) || true
		            docker rm $(docker ps -a -q) || true
		            docker run -d -p 8080:8080 ${{ secrets.DOCKER_USERNAME }}/best_bus_backend:latest
```

 3.  Add the following secrets to your GitHub repository:
    -   `DOCKER_USERNAME`: Your Docker Hub username
    -   `DOCKERHUB_TOKEN`: Your Docker Hub access token
    -   `EC2_HOST`: Public IP of your EC2 instance
    -   `EC2_KEY`: Private key of your EC2 instance (the content of the .pem file)

## Application Deployment

1.  Create a `Dockerfile` in your project root:
    
```   
FROM node:20-alpine ENV NODE_ENV production WORKDIR /usr/src/app RUN npm install -g pm2 COPY package*.json ./ RUN npm ci --only=production COPY . . EXPOSE 8080 CMD ["pm2-runtime", "start", "server.js"]
```
    
2.  Create a `docker-compose.yml` file in your project root:

```   
version:  '3.8' services:   server: build: . env_file: - .env ports: -  "8080:8080" command:  ["pm2-runtime",  "start",  "server.js"]
```    
3.  Push your code to the main branch of your GitHub repository.
4.  GitHub Actions will automatically build, push, and deploy your application to the EC2 instance.

## Nginx and SSL Configuration

1.  Install Nginx and Certbot:
  ```  
sudo  apt update sudo  apt  install nginx certbot python3-certbot-nginx -y
```    

2.  Create Nginx configuration:
	  ```  
		sudo  nano /etc/nginx/sites-available/your-domain.com
	```   

   Add the following configuration:
```
server  {   listen  80; server_name your-domain.com www.your-domain.com; location /  { proxy_pass http://localhost:8080; proxy_set_header Host $host; proxy_set_header X-Real-IP $remote_addr; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_set_header X-Forwarded-Proto $scheme; } }
```    
3.  Enable the configuration:
   ``` 
    sudo  ln -s /etc/nginx/sites-available/your-domain.com /etc/nginx/sites-enabled/
  ```  
		  
###
4.  Test and reload Nginx:
 ```
sudo nginx -t sudo systemctl reload nginx
``` 


5.  Obtain SSL certificate:
 ```
sudo certbot --nginx -d your-domain.com -d www.your-domain.com`
```


6.  Verify SSL configuration by visiting `https://your-domain.com`

## Troubleshooting

-   Docker permission issues:
    ```
    sudo  groupadd  docker sudo  usermod -aG docker ubuntu sudo systemctl restart docker
	 ```   

-   View container logs:
    ```
    docker logs <container-id>
	  ```  


-   Inspect container status:
    ```
    docker inspect <container-id>
    ```
-   Check Nginx logs:
     ```
    `sudo  tail -n 20 /var/log/nginx/error.log`
    ```
-   Ensure firewall allows necessary ports:
   ```   
    `sudo ufw status sudo ufw allow 'Nginx Full'`
   ``` 
-   Check SSH server status:
    ``` 
    `sudo systemctl status ssh sudo  netstat -tuln |  grep :22`
    ```

Remember to replace placeholders like `your-domain.com` with your actual domain name and adjust any paths or configurations to match your specific project structure.
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTY5OTE3MjQwLC0yNjI3MDg1NTBdfQ==
-->