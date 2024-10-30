# Apache Load Balancer Deployment for Tooling Website Solution

## Objective
In this project, we will enhance our Tooling Website solution by adding an **Apache Load Balancer** to distribute traffic between two Web Servers. This will allow users to access our website using a single URL.

## Task
Deploy and configure an Apache Load Balancer on a separate Ubuntu EC2 instance, ensuring that users can be served by both Web Servers through the Load Balancer.

For simplicity, this solution will implement load balancing with **2 Web Servers**, but the approach can be extended to accommodate more servers.

## Prerequisites

Ensure you have:
- Two RHEL8 Web Servers
- One MySQL DB Server (Ubuntu 20.04)
- One RHEL8 NFS server

## Step 1: Launch the Apache Load Balancer (Project-8-apache-lb)

1. Create an Ubuntu 20.04 EC2 instance and name it Project-8-apache-lb, and open TCP port 80 by creating an inbound rule in the security group for this instance.

![WhatsApp Image 2024-10-30 at 16 27 24_6f55c1b0](https://github.com/user-attachments/assets/3e54a7cf-9475-40d3-aed4-6146b4029a2a)

3. SSH into the Project-8-apache-lb instance:


## Step 2: Install and set up Apache on the Load Balancer

1. Install Apache:
```bash
sudo apt update
```
```bash
sudo apt install apache2 -y
```

```bash
sudo apt-get install libxml2-dev
```
![WhatsApp Image 2024-10-29 at 10 52 03_8bb6b1cf](https://github.com/user-attachments/assets/8445e9ad-8a31-4353-9434-4a34f7fa301e)



2.  Enable the necessary Apache modules for load balancing:
```bash
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic
```
![WhatsApp Image 2024-10-29 at 12 43 56_e1a548e2](https://github.com/user-attachments/assets/9b0e5049-8ed8-45a6-95a8-e6eb57b7af5a)


3. After enabling the modules, restart Apache to apply the changes:
```bash
sudo systemctl restart apache2
```

4. Ensure Apache is up and running:
```bash
sudo systemctl status apache2
```



## Step 3: Configure Load Balancing
1. Open the Apache configuration file:
```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```

2. Add the following configuration within the <VirtualHost *:80> section:
```apache
<VirtualHost *:80>
    <Proxy "balancer://mycluster">
        BalancerMember http://<WebServer1-Private-IP-Address>:80 loadfactor=5 timeout=1
        BalancerMember http://<WebServer2-Private-IP-Address>:80 loadfactor=5 timeout=1
        ProxySet lbmethod=bytraffic
        # ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
</VirtualHost>
```

![image](https://github.com/user-attachments/assets/15575ed7-0440-45c2-a7d2-77b21e3258d4)

> Replace <WebServer1-Private-IP-Address> and <WebServer2-Private-IP-Address> with the private IPs of your two RHEL8 web servers.

![](img/lbconfig.png)


> The lbmethod=bytraffic distributes incoming load based on traffic load.
> bytraffic balancing method will distribute incoming load between the  Web Servers according to current traffic load. We can control in which proportion the traffic must be distributed by loadfactor parameter. Other methods are bybusyness, byrequests, heartbeat

3. Restart Apache to Apply Changes:
```bash
sudo systemctl restart apache2
```


## Step 4: Test the Load Balancer and Verify Logs on the Web Servers
1. Access the public IP of your Project-8-apache-lb instance in a browser: 
```bash
http://<Load-Balancer-Public-IP>/index.php
```

- Note: If in the previous project, you mounted /var/log/httpd/ from your Web Servers to the NFS server - unmount them and make sure that each Web Server has its own log directory, do the following steps on the two web servers:

    - Check Existing NFS Mounts:
    ```bash
    df-h
    ```

    - Stop Apache to Prevent Logging Disruption:
    ```bash
    sudo systemctl stop httpd
    ```

    - To unmount the /var/log/httpd/ directory, run:
    ```bash
    sudo umount /var/log/httpd
    ```

    - Verify that the unmount was successful by running `df -h`.
    > This command should no longer display the /var/log/httpd mount.

    - Check /etc/fstab for Persistent NFS Mount:

        - Open the /etc/fstab file with a text editor:
        ```bash
        sudo vi /etc/fstab
        ```
        - Find the line that mounts the NFS directory (it will look similar to this):
        ```javascript
        <nfs-server-IP>:/mnt/logs    /var/log/httpd    nfs    defaults    0 0
        ```

        Comment out the line by adding a # at the beginning, or delete the line entirely:
        ```javascript
        #<nfs-server-IP>:/mnt/logs    /var/log/httpd    nfs    defaults    0 0
        ```

        - Save and close the file.
    - Recreate the Local Log Directory
        - If the /var/log/httpd/ directory was mounted from NFS, it may no longer exist locally on the web server. Recreate it:
        ```bash
        sudo mkdir -p /var/log/httpd
        ```

        - Set the appropriate ownership and permissions:
        ```bash
        sudo chown apache:apache /var/log/httpd
        sudo chmod 755 /var/log/httpd
        ```
    - Restart Apaches
    ```bash
    sudo systemctl start httpd
    ```



> If configured correctly, the requests will be balanced between the two web servers.

2. On both RHEL8 web servers, monitor incoming requests by running:

```bash
sudo tail -f /var/log/httpd/access_log
```

- Refresh the browser page several times, and you should see requests being distributed evenly across both servers



