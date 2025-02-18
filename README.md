# adf-2-mdb

===
---  
   
**Important Note:**  
   
- **Licensing**: Oracle WebLogic Server and Oracle ADF are proprietary software. You'll need to accept Oracle's license agreements and ensure you have the appropriate licenses to use these products.  
    
- **Downloads**: Some Oracle software requires you to download installers manually after logging into your Oracle account.  
    
---  
   
## Overview  
   
We'll create:  
   
- A Docker image with Oracle WebLogic Server and ADF Runtime installed.  
- A simple "Hello World" ADF application packaged as a WAR file.  
- A `docker-compose.yml` file to build and run the application.  
   
---  
   
## Prerequisites  
   
- **Docker** and **Docker Compose** installed on your machine.  
- **Oracle WebLogic Server** and **ADF Runtime** installers.  
- **Git** (optional, for cloning repositories).  
    
---  
   
## Step 1: Download Necessary Software  
   
### 1.1. Oracle WebLogic Server Installer  
   
- Download the **Generic Installer** for WebLogic Server 12c (12.2.1.4.0) from [Oracle's website](https://www.oracle.com/middleware/technologies/weblogic-server-installers-downloads.html).  
  
  - Filename: `fmw_12.2.1.4.0_wls_Disk1_1of1.zip`  
   
### 1.2. Oracle Fusion Middleware Infrastructure Installer (ADF Runtime)  
   
- Download the **Fusion Middleware Infrastructure** installer for 12c (12.2.1.4.0) from [Oracle's website](https://www.oracle.com/middleware/technologies/fusionmiddleware-downloads.html).  
  
  - Filename: `fmw_12.2.1.4.0_infrastructure_Disk1_1of1.zip`  
   
---  
   
## Step 2: Clone Oracle's Docker Images Repository  
   
Oracle provides Dockerfiles and scripts for building WebLogic Server images.  
   
```bash  
git clone https://github.com/oracle/docker-images.git  
```  
   
---  
   
## Step 3: Build the WebLogic Server Base Image  
   
### 3.1. Copy Installer to Dockerfiles Directory  
   
```bash  
cp fmw_12.2.1.4.0_wls_Disk1_1of1.zip docker-images/OracleWebLogic/dockerfiles/12.2.1.4/  
```  
   
### 3.2. Build the Image  
   
Navigate to the dockerfiles directory and build the image:  
   
```bash  
cd docker-images/OracleWebLogic/dockerfiles  
./buildDockerImage.sh -v 12.2.1.4 -d  
```  
   
This will create a Docker image tagged `oracle/weblogic:12.2.1.4-developer`.  
   
---  
   
## Step 4: Build the ADF Runtime Image  
   
### 4.1. Create a Directory for ADF Image  
   
```bash  
mkdir ~/weblogic-adf  
cd ~/weblogic-adf  
```  
   
### 4.2. Create a Dockerfile  
   
Create a file named `Dockerfile` with the following content:  
   
```dockerfile  
FROM oracle/weblogic:12.2.1.4-developer  
   
USER oracle  
   
# Set environment variables  
ENV ORACLE_HOME=/u01/oracle  
ENV PATH=$ORACLE_HOME/bin:$PATH  
   
# Copy the Fusion Middleware Infrastructure installer  
COPY fmw_12.2.1.4.0_infrastructure_Disk1_1of1.zip /u01/  
   
# Install ADF Runtime  
RUN cd /u01 && \  
    unzip fmw_12.2.1.4.0_infrastructure_Disk1_1of1.zip && \  
    java -jar fmw_12.2.1.4.0_infrastructure.jar -silent -responseFile /u01/install.rsp -invPtrLoc /u01/oraInst.loc && \  
    rm -rf fmw_12.2.1.4.0_infrastructure*  
   
# Copy response files  
COPY oraInst.loc /u01/oraInst.loc  
COPY install.rsp /u01/install.rsp  
```  
   
### 4.3. Create Response Files  
   
#### 4.3.1. `oraInst.loc`  
   
Create a file named `oraInst.loc`:  
   
```plain  
inventory_loc=/u01/app/oraInventory  
inst_group=oracle  
```  
   
#### 4.3.2. `install.rsp`  
   
Create a file named `install.rsp`:  
   
```plain  
[ENGINE]  
   
[GENERIC]  
ORACLE_HOME=/u01/oracle  
INSTALL_TYPE=Fusion Middleware Infrastructure  
DECLINE_AUTO_UPDATES=true  
```  
   
### 4.4. Copy the Installer and Response Files  
   
- Copy `fmw_12.2.1.4.0_infrastructure_Disk1_1of1.zip` to the current directory (`~/weblogic-adf`).  
- Ensure `Dockerfile`, `oraInst.loc`, and `install.rsp` are in the same directory.  
   
### 4.5. Build the ADF Image  
   
```bash  
docker build -t oracle/weblogic:12.2.1.4-adf .  
```  
   
---  
   
## Step 5: Create a Simple "Hello World" ADF Application  
   
You can create a simple ADF application in JDeveloper and package it as a WAR file named `HelloWorldADF.war`. For this example, we'll assume you have this WAR file ready.  
   
---  
   
## Step 6: Set Up Docker Compose  
   
### 6.1. Create a Directory for Docker Compose Setup  
   
```bash  
mkdir ~/adf-docker-compose  
cd ~/adf-docker-compose  
```  
   
### 6.2. Copy the ADF Application WAR  
   
Copy `HelloWorldADF.war` into the `~/adf-docker-compose` directory.  
   
### 6.3. Create the Domain Creation Script  
   
Create a file named `createDomain.sh`:  
   
```bash  
#!/bin/bash  
source /u01/oracle/wlserver/server/bin/setWLSEnv.sh  
java -Xmx1024m -Djava.security.egd=file:/dev/./urandom weblogic.WLST /u01/oracle/createDomain.py  
```  
   
Make it executable:  
   
```bash  
chmod +x createDomain.sh  
```  
   
### 6.4. Create the WLST Script  
   
Create a file named `createDomain.py`:  
   
```python  
readTemplate("/u01/oracle/wlserver/common/templates/wls/wls.jar")  
   
cd('Servers/AdminServer')  
set('ListenAddress', '')  
set('ListenPort', 7001)  
   
cd('/')  
cd('Security/base_domain/User/weblogic')  
cmo.setPassword('welcome1')  # Replace with a secure password  
   
setOption('OverwriteDomain', 'true')  
writeDomain('/u01/oracle/user_projects/domains/base_domain')  
closeTemplate()  
exit()  
```  
   
### 6.5. Create a Dockerfile  
   
Create a file named `Dockerfile`:  
   
```dockerfile  
FROM oracle/weblogic:12.2.1.4-adf  
   
USER oracle  
   
ENV DOMAIN_NAME="base_domain"  
ENV DOMAIN_HOME="/u01/oracle/user_projects/domains/${DOMAIN_NAME}"  
ENV ADMIN_PORT="7001"  
ENV ADMIN_PASSWORD="welcome1"  # Replace with a secure password  
   
# Copy files  
COPY createDomain.sh /u01/oracle/  
COPY createDomain.py /u01/oracle/  
COPY HelloWorldADF.war /u01/oracle/  
   
# Create domain and deploy application  
RUN /u01/oracle/createDomain.sh && \  
    mkdir -p ${DOMAIN_HOME}/autodeploy && \  
    mv /u01/oracle/HelloWorldADF.war ${DOMAIN_HOME}/autodeploy/  
   
EXPOSE ${ADMIN_PORT}  
   
# Start WebLogic Server  
CMD ["${DOMAIN_HOME}/startWebLogic.sh"]  
```  
   
### 6.6. Create `docker-compose.yml`  
   
Create a file named `docker-compose.yml`:  
   
```yaml  
version: '3'  
services:  
  adfserver:  
    build: .  
    ports:  
      - "7001:7001"  
    environment:  
      ADMIN_PASSWORD: "welcome1"  # Replace with a secure password  
    image: helloworld-adf:latest  
```  
   
---  
   
## Step 7: Build and Run Using Docker Compose  
   
### 7.1. Build the Docker Image  
   
```bash  
docker-compose build  
```  
   
### 7.2. Start the Container  
   
```bash  
docker-compose up  
```  
   
This command will build the image (if not already built) and start the container. The WebLogic Server might take a few minutes to fully start up.  
   
---  
   
## Step 8: Access the "Hello World" ADF Application  
   
Once the server is running, access your application by navigating to:  
   
```plaintext  
http://localhost:7001/HelloWorldADF  
```  
   
Or, if your application uses a specific context path:  
   
```plaintext  
http://localhost:7001/HelloWorldADF/faces/helloWorld.jspx  
```  
   
You should see your "Hello World" message rendered by the ADF application.  
   
---  
   
## Clean Up  
   
To stop the Docker Compose setup, press `Ctrl+C` in the terminal where it's running, then execute:  
   
```bash  
docker-compose down  
```  
   
---  
   
## Additional Information  
   
- **Adjusting Ports**: If port `7001` is already in use or you want to map it to a different host port, adjust the `ports` section in `docker-compose.yml`:  
  
  ```yaml  
  ports:  
    - "8001:7001"  
  ```  
   
- **Using Secure Passwords**: Always replace default passwords with secure ones, especially in production environments.  
   
- **Logs and Troubleshooting**: To view logs from the container, run:  
  
  ```bash  
  docker logs -f <container_id_or_name>  
  ```  
   
- **Rebuilding the Image**: If you make changes to the Dockerfile or other configuration files, rebuild the image:  
  
  ```bash  
  docker-compose build --no-cache  
  ```  
   
---  
