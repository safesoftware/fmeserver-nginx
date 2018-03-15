# Configure FME Server for SSL using NGINX as a reverse proxy
Here is our blog post about the benefits of using NGINX as a revers proxy for FME Server:
https://blog.safe.com/2016/12/overview-fme-cloud-team-leveraging-nginx-fme-server-deliver-enhanced-performance/
## Ubuntu 16.04
### Install NGINX and create SSL certificate
#### Install NGINX
Run the following commands:
```
sudo apt-get update
sudo apt-get install nginx
```
#### Create SSL certificate
For the purpose of this demo we will use a self-signed certificate.
```
sudo mkdir /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
```
Fill out the prompts appropriately. The most important line is the one that requests the Common Name (e.g. server FQDN or YOUR name). You need to enter the domain name that you want to be associated with your server. You can enter the public IP address instead if you do not have a domain name.
### Install FME Server and configure for SSL
#### Installation
Install zip & gzip:
```
sudo apt-get install zip gzip
```
Download & install FME Server:
```
sudo wget https://downloads.safe.com/fme/2017/fme-server-2017.1.1-b17650-linux-x64~ubuntu.16.04.run
sudo chmod +x fme-server-2017.1.1-b17650-linux-x64~ubuntu.16.04.run
sudo ./fme-server-2017.1.1-b17650-linux-x64~ubuntu.16.04.run
```
Make sure that the host name is similar to the common name specified for the SSL certificate. The port needs to be 8080. Port 80 is already used by NGINX.
Start FME Server:
```
sudo /opt/fmeserver/Server/startServer.sh
```
#### File modifications
Modify the file `/opt/fmeserver/Utilities/tomcat/conf/server.xml` by adding these attributes to the connector element with the `port="8080"` attribute:
```
proxyPort="443"
Scheme="https"
address="127.0.0.1"
```
Modify the file `/opt/fmeserver/Utilities/tomcat/webapps/fmeserver/WEB-INF/conf/propertiesFile.properties` by adding the following line at the end: `WEB_SOCKET_SERVER_PORT=443`

Modify the file `opt/fmeserver/Server/fmeWebSocketConfig.txt` by changing `WEBSOCKET_REQUEST_PORT=7078` to `WEBSOCKET_REQUEST_PORT=8078`

Pre-gzip all fmeserver js and css files:
```
sudo find /opt/fmeserver/Utilities/tomcat/webapps/ -type f -name '*.js' -exec gzip -k -9 {} \;
sudo find /opt/fmeserver/Utilities/tomcat/webapps/ -type f -name '*.css' -exec gzip -k -9 {} \;
```
Import the certificate to the FME Server keystore:
```
sudo keytool -trustcacerts -keystore /opt/fmeserver/Utilities/jre/lib/security/cacerts -storepass changeit -noprompt -importcert -alias fmeserver-cert -file /etc/nginx/ssl/nginx.crt
```
Restart FME Server:
```
sudo /opt/fmeserver/Server/stopServer.sh
sudo /opt/fmeserver/Server/startServer.sh
```
### Configure NGINX as reverse proxy for FME Server
#### Diffie-Hellman for TLS (https://weakdh.org/sysadmin.html)
Run these comamnds:
```
sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
sudo chmod 400 /etc/nginx/ssl/dhparam.pem
```
#### Create & modify NGINX configuration files
Create a folder:
```
sudo mkdir /etc/nginx/fmeserver/
```
Place the following files in the folder:
```
/etc/nginx/fmeserver/502.html
/etc/nginx/fmeserver/ssl.conf
/etc/nginx/fmeserver/websocket.conf
/etc/nginx/fmeserver/nginx-proxy.conf
/etc/nginx/fmeserver/mail.conf
/etc/nginx/fmeserver/fmeserver.conf
```
Modify the file `/etc/nginx/fmeserver/fmeserver.conf` by changing all 3 occurrences of `server_name` to the host name that was used for the SSL certificate and the FME Server installation.
#### Enable the reverse proxy for FME Server
Remove the default site:
```
sudo rm /etc/nginx/sites-enabled/default
```
Copy the FME Server configuration:
```
sudo cp /etc/nginx/fmeserver/fmeserver.conf /etc/nginx/sites-enabled/fmeserver.conf
```
Restart NGINX
```
sudo service nginx restart
```
### Test configuration
Got to FME Server in your browser (for self-signed certificates the site will be show as not secure). Update the Service URLs to use https://. Now you can run Jobs and test the websocket server with the topic monitoring tool.
