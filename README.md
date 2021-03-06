### devopsplayground8-docker-windows

# Hands-on with Microsoft Windows and Docker containers

## Overview
Windows can now natively run containers, without the help of any virtual machines. 
During this session, we'll explore together the capabilities of Docker on Windows, and ways to containerize Windows. 

## Requirements 
1. The IP of your Windows Server 2016 instance  
2. An RDP client (Remote Desktop Protocol)

## Step 1:  Setup Environment

RDP in to your AWS instance.  
User: `Administrator`  
Pass: `Playground123`  

Open a powershell window and run:  
`docker --version`  

I have setup the Docker daemon to listen on both pipe and TCP, but if you have an error about a pipe then run this:  
`Stop-Service docker; dockerd --unregister-service; dockerd -H npipe:// -H 0.0.0.0:2375 --register-service; Start-Service docker;`  
If you do the above but Docker fails to restart, a temporary fix is to restart the vm: `Restart-Computer -Force`  
Wait a few minutes before logging in via RDP and trying `docker --version` again. 

### Step 2: Start an example Docker container  
See what Docker images are avaliable:  
`docker images`  
Try to run the Linux hello world image:  
`docker run hello-world`   
Run the Windows hello world image:  
`docker run microsoft/sample-dotnet`  
List the running containers:  
`docker ps`  
List all containers:  
`docker ps -a`  

## Step 3: Build an IIS Docker image using a DockerFile
Open notepad and edit the sample index.html page:  
`notepad.exe C:\docker\iis\content\index.html`  
Open notepad and create a file called 'DockerFile':  
`notepad.exe C:\docker\iis\DockerFile`  

Copy the following contents to the DockerFile:  
```
FROM microsoft/iis:nanoserver

RUN mkdir C:\site

RUN powershell -NoProfile -Command Import-module IISAdministration; New-IISSite -Name "Site" -PhysicalPath C:\site -BindingInformation "*:8000:"

EXPOSE 8000

ADD content/ /site
```

Build the Docker image:  
`docker build -t test-iis-image C:\docker\iis\`  
Check to see our new Docker Image:  
`docker images`

## Step 4: Run your IIS Docker image as a container
Run the IIS image as a container:  
`docker run -d -p 8000:8000 --name test-iis-container-1 test-iis-image`  
Check to see if the Docker container running:  
`docker ps`  
Inspect the container:  
`docker inspect test-iis-container-1`  
Open Internet Explorer and navigate to:  
`http://<your-container-ip>:8000`  

## Step 5: Start a second IIS container to handle the extra load or simulate a production release
Open notepad and edit the sample index.html page again:  
`notepad.exe C:\docker\iis\content\index.html`  
Re-build the Docker image:  
`docker build -t test-iis-image C:\docker\iis\`  
Run the IIS image as a container:  
`docker run -d -p 8001:8000 --name test-iis-container-2 test-iis-image`  
Check to see if the Docker container running:  
`docker ps`  
Inspect the container:  
`docker inspect -f "{{ .NetworkSettings.Networks.nat.IPAddress }}" test-iis-container-2`  
Open Internet Explorer and navigate to:  
`http://<your-container-ip>:8000`  

## Step 6: Create a Nginx load balancer Docker image
Get the IP addresses for both of your IIS containers:  
`docker inspect -f "{{ .NetworkSettings.Networks.nat.IPAddress }}" test-iis-container-1`  
`docker inspect -f "{{ .NetworkSettings.Networks.nat.IPAddress }}" test-iis-container-2`  

Open notepad and edit the Nginx config file to use the IIS container IP's:  
`notepad.exe C:\docker\nginx\content\lb.conf`  

Open notepad and create a file called 'DockerFile':  
`notepad.exe C:\docker\nginx\DockerFile`  
Copy the following contents to the DockerFile:
```
FROM microsoft/windowsservercore
RUN mkdir C:\nginx
EXPOSE 80
ADD content/ /nginx
```

Build the Docker image:  
`docker build -t test-nginx-lb-image C:\docker\nginx\`  
Check to see our new Docker image:  
`docker images`


## Step 7: Start the load balancer container and start the Nginx process
Run the Nginx image as a container:  
`docker run -itd -p 80:80  --name test-nginx-lb-container test-nginx-lb-image`  
Check to see if the Docker container running:  
`docker ps` 

Run a Docker exec to execute commands on the container:  
`docker exec -i test-nginx-lb-container powershell`

Confirm you are in the container:  
`hostname`  
Check the load balancer config file copied correctly:  
`cd c:\nginx\`  
`ls`  
Run Nginx with the load balancer config:  
`.\nginx.exe -c .\lb.conf`  

Open a new powershell window and inspect the Nginx container:  
`docker inspect -f "{{ .NetworkSettings.Networks.nat.IPAddress }}" test-nginx-lb-container`  
Open Internet Explorer and navigate to:  
`http://<your-container-ip>`  
Refresh the page a few times to see both IIS server pages.

# Help

### Don't be afraid to ask!

### If you need to delete a container
`docker stop container-id-or-name`  
`docker rm container-id-or-name`

### If you need to delete a image
`docker rmi image-id-or-name`  

### If yo need to kill a running nginx process
`tasklist /fi "imagename eq nginx.exe"`  
`taskkill /F /T /PID <pid>`  
