
# HA Pi-Hole DNS Ad Filter Ansible Playbook  

This playbook is a collection of tasks that will download, install, and configure the software necessary to run the
popular pi-hole ad filter on 2 raspberry pi's in a highly available redundant configuration.  Keepalived uses vrrp to allow
the 2 pi servers to share an ip address for backup redundancy purposes.  
## History  
I like running the pi-hole software on my home network because it really does filter out a lot of trash. Running it on a raspberry pi keeps it separate from other devices on my network, and it's surprisingly stable! I also use this for LAN name resolution by keeping /etc/hosts updated with my local server names/ip.  
What I wanted was 2 dns servers that would share a single ip (vip), so when A goes down, B takes over and answers requests directed at the vip. This setup was chosen over simply running 2 servers for a few reasons.  

1. Maintenance is easy. Remove a pi, and DNS services will remain up.
2. Clients shouldn't have to wait for a failed dns server to timeout during a request before trying the alive server.
3. Assigning 2 dns servers will allow the client to pick which one to use (this is not desired).  
4. Assigning a vip to a load balanced group of servers allows the application to decide which server to use, not the client.  
5. Clients get assigned 1 DNS ip address from the DHCP server (the vip). The active dns server will respond to the request sent to the vip.  


## What You Get  
You will end up with 2 raspberry pi's, each running a copy of pi-hole with the settings specified to your environment.
Each pi will have its own IP address, and will share an IP address between them known as a vip. This vip is the IP address for the DNS server that will be handed out to your client devices, usually from your dhcp server.
If the primary server goes down, the secondary server will take over and answer dns requests that have been sent to the vip IP. When the primary comes back online, the secondary will step down, and allow the primary to resume control of the vip  

## What You Need  

2x Raspberry Pi's (I suggest Pi4, or Pi3)  
2x Micro SD Card (1 for each pi)  
1x HDMI to micro-HDMI cable (for temporary video)  
2x USB-C power cables (1 for each pi)  
3x IP addresses (1 for each pi, and a shared ip)  
2x Available ports on your network  


**Obviously**  
monitor  
keyboard  
ethernet cables  
a network that has DHCP services  


## What You Don't Get
Synchronization of files between the 2 dns servers. I had no need for it since I don’t have any custom block lists.  I also use ansible to make sure the hosts file that never changes much, is the same between the 2 devices from the start.  If you needed to make updates to your hosts file, you can edit it in this playbook and re-run it. The script will detect the hosts file update and push only that.  
Support.  If you're attempting to implement this on your home network, you've already moved well past your support stage in life.  You are support  

## What You Need to Do  

1. Make the raspberry pi dns servers’ network ready (see below).  
2. Ensure the Ansible controller can connect to the dns servers using ssh-key-based-auth, and password-less sudo.  
3. Run playbook.  
4. Test.  
5. Change DNS server IP address on all client devices.  


### How to make your pi network ready  

Image both SD cards with the Raspberry Pi OS Lite image file.  I used the `dd` command in the terminal on macOS, but refer to the
raspberry pi website for instructions on downloading the OS and imaging your SD card  

**Change raspi-config settings**  
`sudo raspi-config`  
change timezone  
change keyboard layout if needed  
enable ssh server  
change hostname to `dns1` or `dns2` depending on which device you're configuring. The script will fail if the hostnames are not `dns1` or `dns2`  
Want a different hostname? (see the important note in the Editing Variables section)  


**Create new user and add to groups (for security)**  
Replace YOUR_USER with the username of your preference  

`sudo adduser YOUR_USER`  
`sudo usermod -a -G adm,dialout,cdrom,sudo,audio,video,plugdev,games,users,input,netdev,gpio,i2c,spi YOUR_USER`  

**Add new user to password-less sudo (append to the bottom of the file)**  
`sudo visudo`  
YOUR_USER    ALL=(ALL) NOPASSWD:ALL  

**Change ip address of server**  
`sudo vi /etc/dhcpcd.conf`  
interface eth0  
  static ip_address=192.168.1.51/24  
  static routers=192.168.1.1  
  static domain_name_servers=8.8.8.8  

Make sure to change the ip for each server  

**Reboot**  
`sudo reboot`  

**Log on as YOUR_USER and remove the old pi user**  
`sudo deluser -remove-home pi`  

**Install NTP/Update/Upgrade**  
Need NTP to make sure the time is correct, otherwise apt won't work  

`sudo apt install ntp`  
`sudo apt update`  
`sudo apt upgrade -y`  

**Copy ssh key from ansible controller to dns servers**  
Run these commands from the device that will be executing your playbook, not on the dns servers themselves.  
`ssh-copy-id YOUR_USER@192.168.1.51`  
`ssh-copy-id YOUR_USER@192.168.1.52`  

Make sure you can ssh into the dns servers without having to type in a password, and that you can escalate sudo priv without a password as well.  

### Editing Variables  
To set the servers up for your environment, you'll need to edit the vars section in the dns-install.yml file.  

**Important Note**  
If you want to use a hostname other than dns1 and dns2, you will need to change it in the dns-install.yml vars section, and rename the 4 jinja2 templates in the dns folder.  
dns1-keepalived.j2 ----->  newhostname1.keepalived.j2  
dns2-keepalived.j2 ----->  newhostname2.keepalived.j2  
dns1-setupVars.j2 ----->  newhostname1.setupVars.j2  
dns2-setupVars.j2 ----->  newhostname2.setupVars.j2  

The script will look at the remote server’s hostname and try to match it with the corresponding name in the jinja2 template.  

...or you can just leave it dns1 and dns2  

  `vars:`    
   `pass_hash: ''                               # Add your password hash for webgui logon, or leave blank to change in the webgui later.`  
   `vrrp_auth: 'dayman-fighter-of-the-nightman' # password for HA vrrp auth between servers. please change.`  
   `pihole_intf: 'eth0'                         # ethernet nic of the pi, change if necessary.`  
   `dns1_ip: '192.168.1.51/24'                  # The ip of the first dns server.`  
   `dns1_name: 'dns1'                           # The name of the first dns server.`  
   `dns2_ip: '192.168.1.52/24'                  # The ip of the second dns server.`  
   `dns2_name: 'dns2'                           # The name of the second dns server.`  
   `dns_vip: '192.168.1.53/24'                  # The shared ip address to hand out to clients.`  
   `dns_vip_name: 'dns'                         # The name of the shared ip.`  
   `pub_dns1_ip: '8.8.8.8'                      # Upstream public dns server 1.`  
   `pub_dns2_ip: '1.1.1.1'                      # Upstream public dns server 2.`  

Then save the file.   

Edit the dns_inventory file to add your devices ip and username.  

`x.x.x.x ansible_connection=ssh ansible_user=YOUR_USER ansible_python_interpreter=/usr/bin/python3`  
`y.y.y.y ansible_connection=ssh ansible_user=YOUR_USER ansible_python_interpreter=/usr/bin/python3`  


By default, you will get a hosts file that will include the name/ip association for the 2 servers and the vip.  Edit the dns-hosts-file in the dns folder to include any additional host names you want to resolve.  


### Executing the playbook  
From the ansible controller, in the pi-hole_HA_playbook directory, execute the following command.  
`ansible-playbook -i dns_inventory dns-install.yml`   


### Check and Test  
After successful play execution, Run these tests.  

Ping the vip to make sure vrrp is operational.  
`ping 192.168.1.53`  

Run nslookup against each server and make sure you get a response.  
`nslookup pi-hole.net 192.168.1.51`  
`nslookup pi-hole.net 192.168.1.52`  
`nslookup pi-hole.net 192.168.1.53`  

Access the webgui of each dns server and the vip. Make sure accessing the vip sends you to dns1 and all settings are correct.  
`http://192.168.1.51/admin`  
`http://192.168.1.52/admin`  
`http://192.168.1.53/admin`  

While logged into the webgui of the vip `http://192.168.1.53/admin`, look at the upper right hand corner of the gui and notice the hostname is `dns1`.
Reboot `dns1` and hit the refresh button in the browser.  Notice the hostname change to `dns2`.  When `dns1` comes back, hit refresh again.  
Notice the hostname change back to `dns1`.  
If all works, change the dns server ip that gets handed out to clients in your dhcp server.  Once you're using
the new dns servers, you can resolve the hostnames `dns1`,`dns2`, and `dns` to their respective IP's.


### Built With

* [Ansible](https://www.ansible.com/)

* [Jinja2](https://jinja.palletsprojects.com/)

### Authors

* **James Kayser** - *Initial work* - [kayserj](https://github.com/kayserj)



