**Hosting:**
Hoting is the process of deploying an application or website on a server that is connected to the internet so that users can access it remotely.

During my DevOps journey I **hosted** a **static html project**, a **python django** project and a **php application** in a demo server

**STEPS FOR STATIC HOSTING PRACTICE**

  I used ubuntu server and nginx service for hosting through my virtual machine.

First of all, download the service;
  **sudo apt update
  sudo apt install -y nginx**

the next step is to start and enable the servie using command:
  **sudo systemctl start nginx
  sudo systemctl enable nginx**

Confirm if the service has started using command:
  **sudo systemctl status nginx.service**

Then open the directory /var/www/html
All of your application data is stored there

I created a "default" directory, created a index.html file inside the directory and wrote some content using **vim index.html**
Remember having an index.html file is a must while hosting an website because the nginx scans for this file to be loaded first and if the file is not found. It displays error.

The next step is storing a domain name under your virtual machies ip address as:
  STEP 1: Open cmd as an administrator in your windows machine:
  STEP 2: then browse through **/drivers/etc/notepad hosts**
The notepad host is opened. Then in that file write domain name as:

ip    domain_name
For example:
192.168.0.0    www.newstatichtml.com

Why this step is necessary:
This step maps the domain name to the IP address locally. When the browser tries to access www.newstatichtml.com, the system knows which IP address to send the request to. 
Without this mapping, the domain name would not resolve and the site would not load.

The next step is writing a configuration file:

inside /etc/nginx/conf.d I created a new file named new.conf
The configuration code goes in this file as:

server {
        listen 80;
        server_name www.newstatichtml.com;
        root /var/www/html/default;
        index index.html index.html index.nginx-debian.html;
}

Why this step is necessary:
This configuration tells Nginx how to handle incoming HTTP requests:
**listen 80** tells Nginx to accept web traffic on port 80
**server_name** specifies which domain this configuration applies to
**root** defines the directory where the website files are stored
**index** specifies the default file to load when the site is accessed
Without this configuration, Nginx would not know which website to serve for the given domain.

Once the configuration is written check of there is any error using:

nginx -t

Why this step is necessary:
This command verifies the Nginx configuration syntax before applying it. It helps catch mistakes early and prevents Nginx from failing to start due to configuration errors.

Than restart the service using the command:
systemctl restart nginx.service

Why this step is necessary:
Restarting the Nginx service applies the new configuration changes. Until the service is restarted (or reloaded), Nginx will continue using the old configuration and the new site will not be served.

Now you can accees the application using your domain name (in this case www.newstatichtml.com) in your web browser.

 



