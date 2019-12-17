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

      #DROP all the packets
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

To configure mail, download mutt

        sudo apt-get install mutt -y  

Test it with  

        echo "Test Mail" | mutt -s "Test Subject" rebeca.cenamor@gmail.com

Edit the file

        sudo nano /etc/mailname

and add

        student.hive.fi

Create a new alias

        sudo nano /etc/aliases

Add an email alias for reboot

        root: rebeca.cenamor@gmail.com

Test it with:

        echo "Test email body" | mutt -s "Test Subject" root

Install postfix  

        sudo apt-get install postfix -y

Start postfix mail system

        sudo service postfix start

        

