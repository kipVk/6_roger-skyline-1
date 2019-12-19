# 7_roger-skyline-1
Project roger-skyline-1 done at Hive Helsinki  


# Installation of Debian #  

- Download Debian from https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-10.2.0-amd64-netinst.iso  
- Use English language and Finnish locale  
- Select a root password  
- Create a non-root user  
- Follow the installation. In software selection we deselect the desktop environment to have enough space. Also this will be a web server, we don't need the GUI.  


# Non-root user #  

Login with the non-root user  
If we want to create one from the command line we can use this command:  


    adduser kip  


# Use sudo, with this user #  

Use su command to login as root  
Install sudo command:  


    apt-get install sudo  


Edit /etc/sudoers with nano or vi, add this line:  


    kip    ALL(ALL:ALL) ALL  


There is a workaround to do this process. If we install Debian, without a password in root, the system will install sudo in non-root users.  


# Static IP and a Netmask in \30. #  

We modify, in virtualbox, the network configuration to use Bridge configuration, in this mode we will be able to use a webserver in the virtual machine.  

We open the file /etc/network/interfaces with nano, or vi, and see the following lines:  


    allow-hotplug enp0s3  
    iface enp0s3 inet dhcp    


- **allow-hotplug**, allow to use the interface enp0s3 in hotplug mode  
- **iface enp0s3 inet dhcp**, use the interface enp0s3 in dhcp mode  

We will use this configuration  


    auto enp0s3  
    allow-hotplug enp0s3  
    iface enp0s3 inet static  
        address 10.11.200.108  
        netmask 255.255.255.252 #netmask /30  
        gateway 10.11.254.254  


Restart the service with:  
    sudo service networking restart  

Now we can test the connection sending a ping to google.com. We receive a response.  

# Modify SSH #  

We check if SSH is running with:  


    ps aux | grep ssh  


SSH is running in /sr/sbin/sshd -D  

Edit the configuration file in /etc/ssh/sshd_config  

- Uncomment Port 22, and write Port 5555  
- Uncomment PermitRootLogin, change to no  
- Uncomment PubKeyAuthentication yes  

Restart SSH to apply the changes  


    sudo service SSH restart  


In the client, we will create a public/private key using the following command:  


    ssh-keygen  


Login using ssh and copy the public key  


      ssh kip@10.11.200.108 -p 5555  
      sudo mkdir /.ssh  
      cd /.ssh  
      sudo nano authorized_kets  

Paste the pub key and save


      sudo nano /etc/ssh/sshd_config  


Uncomment Password Authentication and put no    
Save file   


      sudo service ssh restart  


Another option is to use this command  


      ssh-copy-id -i id_rsa.pub kip@10.11.200.108 -p 5555  


Login using the following command:  


      ssh kip@10.11.200.108 -p 5555  



# Firewall #  

Check the actual list of IPtables with the attribute -L  


      sudo iptables -L
      sudo iptables -t nat -L
      sudo iptables -t mangle -L


IPTables manages three types of tables  

- MANGLE tables. These tables modify the packets. The TOS, the TTL, and the MARK parameters.  
- NAT tables. These tables allow to modify the ip headers of the packets. We use this for SNAT (Ip masquerading) and DNAT (port forwarding).  
- Filter tables. These are the tables used to DROP and ACCEPT packets.  

We will create and modify the file /etc/network/if-pre-up.d/iptables, this file is loaded during startup.  


      sudo nano etc/network/if-pre-up.d/iptables  


Write this file  


        #!/bin/bash
        #Reset the rules of the three tables
        iptables -F
        iptables -X
        iptables -t nat -F
        iptables -t nat -X
        iptables -t mangle -F
        iptables -t mangle -X

        #DROP all the packages
        iptables -P INPUT DROP
        iptables -P OUTPUT DROP
        iptables -P FORWARD DROP

        #OPEN PORTS
        iptables -A INPUT -p tcp --dport 5555 -j ACCEPT #SSH
        iptables -A INPUT -p tcp --dport 80 -j ACCEPT #http
        iptables -A OUTPUT -p tcp --dport 80 -j ACCEPT #http, to update packages
        iptables -A INPUT -p tcp --dport 443 -j ACCEPT #https
        iptables -A OUTPUT -p tcp --dport 443 -j ACCEPT #https, to update packages
        iptables -A OUTPUT -p udp --dport 53 -j ACCEPT #dns normal
        iptables -A OUTPUT -p tcp --dport 53 -j ACCEPT #edns
        iptables -A OUTPUT -p tcp --dport 25 -j ACCEPT #sendmail
        iptables -A OUTPUT -p tcp --dport 587 -j ACCEPT #smtp
        iptables -A OUTPUT -p icmp --icmp-type echo-request -j ACCEPT #ping output
        iptables -A INPUT -p icmp --icmp-type echo-reply -j ACCEPT #ping reply

        #ACCEPT established or related connections
        iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
        iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT


We change the permissions to executable   


      sudo chmod +x /etc/network/if-pre-up.d/iptables  


If there is no net you have to use these permissions  


      sudo chmod 0777 /etc/network/if-pre-up.d/iptables  


Execute the script  


      sudo /etc/network/if-pre-up.d/iptables   


# DDOS PROTECTION #  

We can use several tools to have DOS Protection. IPTables, Fail2Ban  
IPTables requires a lot of rules, so we will use Fail2Ban to create these rules  

Install Fail2Ban  

We will use this guide to protect ssh, http and https:  

https://www.digitalocean.com/community/tutorials/how-to-protect-an-apache-server-with-fail2ban-on-ubuntu-14-04  


      sudo apt-get update  
      sudo apt-get install fail2ban  


copy the jail.conf file as jail.local  


      sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local  


We set the destination mail, and the sender  


      destemail = rcenamor@student.hive.fi  
      sender = root@<fq-hostname>  


The DOS protection is set. We can modify in the sshd section the maxrety values or the bantime.  


# PORT SCANNING #  

We will use Port Sentry  



    sudo apt-get install portsentry  


Edit the configuration in /etc/portsentry/portsentry.conf  

Change BLOCK_UDP and BLOCK_TCP to 1  

Edit the configuration in /etc/default/portsentry.conf   

We can try if the ports are being detected with  


      nmap -p 1-65535 -T4 -A -v -PE -PS22,25,80 -PA21,23,80 -Pn 10.11.200.108  


And detect attacks with  


      grep "attackalert" /var/log/syslog  



# SERVICES #  

Check the services enabled  


      sudo systemctl list-unit-files --state=enabled  


This is a list of the services  

cups.path
anacron.service  
apache2.service  
apparmor.service  
autovt@.service            
avahi-daemon.service        
bluetooth.service            
console-setup.service         
cron.service                   
cups-browsed.service            
cups.service                     
dbus-fi.w1.wpa_supplicant1.service
dbus-org.bluez.service             
dbus-org.freedesktop.Avahi.service  
dbus-org.freedesktop.timesync1.service  
fail2ban.service                        
getty@.service                          
keyboard-setup.service                  
networking.service                      
rsyslog.service                         
ssh.service                             
sshd.service                            

We only need these services  

apache2.service  
autovt@.service  
console-setup.service  
cron.service  
fail2ban.service  
getty@.service  
keyboard-setup.service  
networking.service  
ssh.service  
sshd.service  

We can disable them by using  


      sudo systemctl disable cups.path  
      sudo systemctl disable anacron.service  
      sudo systemctl disable apparmor.service  
      sudo systemctl disable avahi-daemon.service  
      sudo systemctl disable bluetooth.service  
      sudo systemctl disable cups-browsed.service  
      sudo systemctl disable cups.service   
      sudo systemctl disable dbus-fi.w1.wpa_supplicant1.service  
      sudo systemctl disable dbus-org.bluez.service   
      sudo systemctl disable dbus-org.freedesktop.Avahi.service  
      sudo systemctl disable dbus-org.freedesktop.timesync1.service  
      sudo systemctl disable rsyslog.service   


# UPDATE SCRIPT #   

At /etc/cron.d/ create a file

        sudo nano /etc/cron.d/update_script.sh  

On that script write  

        #!/bin/bash
        echo >> /var/log/update_script.log
        echo "----------------------------" >> /var/log/update_script.log
        echo "- Scheduled Upgrade system -" >> /var/log/update_script.log
        echo "----------------------------" >> /var/log/update_script.log
        echo >> /var/log/update_script.log
        echo "Started at: " >> /var/log/update_script.log
        date >> /var/log/update_script.log
        echo >> /var/log/update_script.log
        echo "----------------------------" >> /var/log/update_script.log
        sudo apt-get update -y >> /var/log/update_script.log
        echo "----------------------------" >> /var/log/update_script.log
        sudo apt-get upgrade -y  >> /var/log/update_script.log
        echo "----------------------------" >> /var/log/update_script.log
        echo >> /var/log/update_script.log
        echo "Finished at: " >> /var/log/update_script.log
        date >> /var/log/update_script.log
        echo >> /var/log/update_script.log

It will execute an upgrade and save the results on the log file like this:

        ----------------------------
        - Scheduled Upgrade system -
        ----------------------------

        Started at:
        Tue 17 Dec 2019 05:34:28 PM EET

        ----------------------------
        Hit:1 http://security.debian.org/debian-security buster/updates InRelease
        Hit:2 http://deb.debian.org/debian buster InRelease
        Hit:3 http://deb.debian.org/debian buster-updates InRelease
        Reading package lists...
        ----------------------------
        Reading package lists...
        Building dependency tree...
        Reading state information...
        Calculating upgrade...
        0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
        ----------------------------

        Finished at:
        Tue 17 Dec 2019 05:34:29 PM EET

To schedule it once a week at 4am and every time the machine reboots we need to make a cronfile

        sudo crontab -e

This is an example of how the jobs should be defined:

        # Example of job definition:
        # .---------------- minute (0 - 59)
        # |  .------------- hour (0 - 23)
        # |  |  .---------- day of month (1 - 31)
        # |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
        # |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
        # |  |  |  |  |
        # *  *  *  *  * user-name command to be executed
        17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
        25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
        47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
        52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
        #

There we need to add the line

        SHELL=/bin/bash
        PATH=/sbin:/bin:/usr/sbin:/usr/bin

        @reboot sudo /etc/cron.d/update_script.sh
        0 4 * * 6 sudo /etc/cron.d/update_script.sh  

Save the file, and at 4am on Saturdays the script previously created will be executed.


# CHECK CRONTAB CHANGES #  

To configure mail, download mutt and postfix

        sudo apt-get install mutt postfix -y  

Edit the file  

        sudo nano /etc/mailname  

and add  

        student.hive.fi

Create a new alias  

        sudo nano /etc/aliases

Add an email alias for root    

        root: rebeca.cenamor@gmail.com

Activate the alias with   

        sudo newaliases

Start postfix mail system  

        sudo service postfix start

Test it with:  

        echo "Test email body" | mutt -s "Test Subject" root


Create a new script /etc/cron.d/check.sh and Write

        #!/bin/bash
        if [ $(($(date +%s) - $(date +%s -r /etc/crontab))) -lt 86400 ]
        then
              echo "Crontab file was modified" | mutt -s "Modification Alert for Crontab file" root
        fi

Give execution permissions

        sudo chmod 777 /etc/cron.d/check.sh

Edit the file /etc/crontab and check if the email is sent. If it works, modify crontab

        sudo crontab -e

Add the new job. The file should look like this:

        SHELL=/bin/bash
        PATH=/sbin:/bin:/usr/sbin:/usr/bin

        @reboot sudo /etc/cron.d/update_script.sh
        0 4 * * 6 sudo /etc/cron.d/update_script.sh
        0 0 * * * sudo /etc/cron.d/check.sh


# WEB PART

(following this tutorial)

Install apache2

        sudo apt update
        sudo apt install apache2

Test if it's working

        sudo systemctl status apache2

From the Mac, go to a browser and test the machine's ip 10.11.200.108. It should open the apache2 page.

Apache on Debian 10 has one server block enabled by default that is configured to serve documents from the /var/www/html directory. While this works well for a single site, it can become unwieldy if you are hosting multiple sites. Instead of modifying /var/www/html, let’s create a directory structure within /var/www for our your_domain site, leaving /var/www/html in place as the default directory to be served if a client request doesn’t match any other sites.

Create the directory for your_domain as follows, using the -p flag to create any necessary parent directories:

        sudo mkdir -p /var/www/rcenamor

Next, assign ownership of the directory with the $USER environmental variable:

        sudo chown -R $USER:$USER /var/www/rcenamor

The permissions of your web roots should be correct if you haven’t modified your unmask value, but you can make sure by typing:

        sudo chmod -R 755 /var/www/rcenamor

Next, create a sample index.html page using nano:

        sudo nano /var/www/rcenamor/index.html

In order for Apache to serve this content, it’s necessary to create a virtual host file with the correct directives. Instead of modifying the default configuration file located at /etc/apache2/sites-available/000-default.conf directly, let’s make a new one at /etc/apache2/sites-available/rcenamor.conf:

        sudo nano /etc/apache2/sites-available/rcenamor.conf

Paste in the following configuration block, which is similar to the default, but updated for our new directory and domain name:

        <VirtualHost *:80>
            ServerAdmin rebeca.cenamor@gmail.com
            ServerName 10.11.200.108
            ServerAlias www.rcenamor
            DocumentRoot /var/www/rcenamor
            ErrorLog ${APACHE_LOG_DIR}/error.log
            CustomLog ${APACHE_LOG_DIR}/access.log combined
        </VirtualHost>

Notice that we’ve updated the DocumentRoot to our new directory and ServerAdmin to an email that the rcenamor site administrator can access. We’ve also added two directives: ServerName, which establishes the base domain that should match for this virtual host definition, and ServerAlias, which defines further names that should match as if they were the base name.

Save and close the file when you are finished.

Let’s enable the file with the a2ensite tool:

        sudo a2ensite rcenamor.conf

Disable the default site defined in 000-default.conf:

        sudo a2dissite 000-default.conf

Next, let’s test for configuration errors:

        sudo apache2ctl configtest

You should see the following output:

        Output
        Syntax OK

Restart Apache to implement your changes:

        sudo systemctl restart apache2

Accessing 10.11.200.108 on a web browser should show the index.html result file.

If that's working, I wanna copy some files from my Mac to the virtual machine. For that I used:

        scp -P 5555 -r /Users/rcenamor/kvk3/Login_v1/* kip@10.11.200.108:/var/www/rcenamor


# SSL CERTIFICATES

Generate the ssl certificates with this command:

        sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -subj "/C=FR/ST=IDF/O=42/OU=Project-roger/CN=10.11.200.247" -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt

Create the /etc/apache2/conf-available/ssl-params.conf file to have this output:

        SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
        SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
        SSLHonorCipherOrder On

        Header always set X-Frame-Options DENY
        Header always set X-Content-Type-Options nosniff

        SSLCompression off
        SSLUseStapling on
        SSLStaplingCache "shmcb:logs/stapling-cache(150000)"

        SSLSessionTickets Off

Edit the /etc/apache2/sites-available/default-ssl.conf and /etc/apache2/sites-available/rcenamor-ssl.conf to have this output

        <IfModule mod_ssl.c>
        	<VirtualHost _default_:443>
        		ServerAdmin rebeca.cenamor@gmail.com
        		ServerName	10.11.200.108

        		DocumentRoot /var/www/rcenamor

        		ErrorLog ${APACHE_LOG_DIR}/error.log
        		CustomLog ${APACHE_LOG_DIR}/access.log combined

        		SSLEngine on

        		SSLCertificateFile	/etc/ssl/certs/apache-selfsigned.crt
        		SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

        		<FilesMatch "\.(cgi|shtml|phtml|php)$">
        				SSLOptions +StdEnvVars
        		</FilesMatch>
        		<Directory /usr/lib/cgi-bin>
        				SSLOptions +StdEnvVars
        		</Directory>

        	</VirtualHost>
        </IfModule>

And to load our new config run those commands:

        sudo a2enmod ssl
        sudo a2enmod headers
        sudo a2ensite default-ssl
        sudo a2enconf ssl-params
        systemctl reload apache2
