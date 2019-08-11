# CuckooSetup

## Hardware requirements
* Ubuntu 18.04 LTS
* Quad Core CPU
* 4 GB RAM
* 320 GB HDD

## Initial Setup
* Update the package manager and install core tools
````
sudo apt-get update
sudo apt-get -y install python virtualenv python-pip python-dev build-essential
````
* Create a new user to run Cuckoo
````
sudo adduser --disabled-password --gecos "" cuckoo
````
* Giving the cuckoo user permission to create network dumps
````
sudo groupadd pcap
sudo usermod -a -G pcap cuckoo
sudo chgrp pcap /usr/sbin/tcpdump
sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
````
* Disabling AppArmor
````
sudo apt-get install -y apparmor-utils
sudo aa-disable /usr/sbin/tcpdump
````
* Downloading and mounting Windows 7
````
wget https://cuckoo.sh/win7ultimate.iso
sudo mkdir /mnt/win7
sudo mount -o ro,loop win7ultimate.iso /mnt/win7
````

## Installing VirtualBox
* Adding VirtualBox repository keys
````
wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- | sudo apt-key add -
wget -q https://www.virtualbox.org/download/oracle_vbox.asc -O- | sudo apt-key add -
````
* Adding VirtualBox repository
````
sudo add-apt-repository "deb [arch=amd64] http://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib"
````
* Installing VirtualBox 5.2 and adding the cuckoo user to the vboxusers group
````
sudo apt-get update
sudo apt-get install virtualbox-5.2
sudo usermod -a -G vboxusers cuckoo
````

## Cuckoo and VMCloak installation
* Installing required packages for VMCloak and Cuckoo
````
sudo apt-get -y install build-essential libssl-dev libffi-dev python-dev genisoimage
sudo apt-get -y install zlib1g-dev libjpeg-dev
sudo apt-get -y install python-pip python-virtualenv python-setuptools swig
````
* Creating a virtualenv for Cuckoo
````
sudo su cuckoo
virtualenv ~/cuckoo
. ~/cuckoo/bin/activate
````
* Installing Cuckoo and VMCloak
````
pip install -U cuckoo vmcloak
````

## Automatic VM creation
* Instantiating a VirtualBox Host-Only network adapter for the VMs to use
````
vmcloak-vboxnet0
````
* Setting up a Windows VM
````
vmcloak init --verbose --win7x64 win7x64base --cpus 2 --ramsize 2048
````
* Cloning the base image
````
vmcloak clone win7x64base win7x64cuckoo
````
* Installing software packages
````
vmcloak install win7x64cuckoo adobepdf pillow dotnet java flash vcredist vcredist.version=2015u3 wallpaper
````
* Installing ie11
````
vmcloak install win7x64cuckoo ie11
````
* Installing Office 2007
````
vmcloak install win7x64cuckoo office office.version=2007 office.isopath=/path/to/office2007.iso office.serialkey=XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
````
* Creating 4 snapshots
````
vmcloak snapshot --count 4 win7x64cuckoo win7x64cuckoo_ 192.168.56.101
````

## Configuring Cuckoo
* Initializing Cuckoo and its configuration
````
cuckoo init
````

## Postgres installation
* Installing Postgres
````
sudo apt install postgresql postgresql-contrib
sudo apt-get install libpq-dev python-dev
````
* Installing Postgres database driver for Cuckoo
````
pip install psycopg2
````
* Creating a user and database for Cuckoo to use
````
sudo -u postgres psql
CREATE DATABASE cuckoo;
CREATE USER cuckoo WITH ENCRYPTED PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE cuckoo TO cuckoo;
\q
````
* Editing the cuckoo.conf file to use Postgres instead of SQLite
````
nano /home/cuckoo/.cuckoo/conf/cuckoo.conf
````
* Change the connection = line to connection =  postgresql://cuckoo:password@localhost/cuckoo

## Adding VMs
* Preparing the virtualbox.conf file by removing the cuckoo1 entry from machines = cuckoo1
````
nano /home/cuckoo/.cuckoo/conf/virtualbox.conf
````
* Adding snapshots to virtualbox.conf
````
while read -r vm ip; do cuckoo machine --add $vm $ip; done < <(vmcloak list vms)
````
* Installing Cuckoo Signatures
````
cuckoo community --force
````

## Network configuration
* Replace outgoinginterface with your outgoing interface
````
sudo sysctl -w net.ipv4.conf.vboxnet0.forwarding=1
sudo sysctl -w net.ipv4.conf.outgoinginterface.forwarding=1
````
* Replace outgoinginterface with your outgoing interface
````
sudo iptables -t nat -A POSTROUTING -o outgoinginterface -s 192.168.56.0/24 -j MASQUERADE
sudo iptables -P FORWARD DROP
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -s 192.168.56.0/24 -j ACCEPT
sudo apt-get install -y iptables-persistent
````

## Enabling IP forwarding at startup after reboot
* Uncomment the line net.ipv4.ip_forward=1
````
sudo gedit /etc/sysctl.conf
````

## Auto starting the VirtualBox network interface on reboot
* Create the /opt/system/vboxhostonly directory and create the bash script to run the vboxmanage commands
````
sudo mkdir /opt/systemd/
sudo vim /opt/systemd/vboxhostonly
````
* Copy in the text below and save with vim (hit esc key and type “:w” and hit enter)
````
#!/bin/bash

vboxmanage hostonlyif create

vboxmanage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1
````
* Go to the directory where you saved the vboxhostonly file and make the file executable
````
cd /opt/systemd/
sudo chmod a+x vboxhostonly
````
* Create the vboxhostonlynic.service file in /etc/systemd/system/ directory
````
sudo touch /etc/systemd/system/vboxhostonlynic.service
sudo gedit /etc/systemd/system/vboxhostonlynic.service
````
* Copy in the code below and save the file
````
Description=Setup VirtualBox Hostonly Adapter

After=vboxdrv.service



[Service]

Type=oneshot

ExecStart=/opt/systemd/vboxhostonly



[Install]

WantedBy=multi-user.target
````
* Now install the systemd service and enable it so it will be executed at boot time
````
systemctl daemon-reload
systemctl enable vboxhostonlynic.service
systemctl start vboxhostonlynic.service
````

## Cuckoo Web Interface
* Installing MongoDB
````
sudo apt-get install mongodb
````
* Change the MongoDB section from enabled = no to enabled = yes
````
nano /home/cuckoo/.cuckoo/conf/virtualbox.conf
````
* Installing uWSGI
````
pip install uwsgi
````
* Installing uWSGI and nginx packages
````
sudo apt-get install uwsgi uwsgi-plugin-python nginx
````
* Generating the configuration files for uWSGI
````
cuckoo web --uwsgi > cuckoo-web.ini
sudo cp cuckoo-web.ini /etc/uwsgi/apps-available/cuckoo-web.ini
sudo ln -s /etc/uwsgi/apps-available/cuckoo-web.ini /etc/uwsgi/apps-enabled/cuckoo-web.ini
````
* Ensuring that the www-data user can read the Cuckoo web files by adding it to the cuckoo group
````
sudo adduser www-data cuckoo
sudo systemctl restart uwsgi
````
* Generating the configuration files for nginx. Edit cuckoo-web.conf to listen *:8000;
````
cuckoo web --nginx > cuckoo-web.conf
nano cuckoo-web.conf
sudo cp cuckoo-web.conf /etc/nginx/sites-available/cuckoo-web.conf
sudo ln -s /etc/nginx/sites-available/cuckoo-web.conf /etc/nginx/sites-enabled/cuckoo-web.conf
sudo systemctl restart nginx
````

## Starting Cuckoo
* Switching user, starting the venv and starting Cuckoo
````
sudo su cuckoo
. ~/cuckoo/bin/activate
cuckoo --debug
````

## Freeing up space
* Freeing up space to make the ova file as small as possible
````
sudo apt-get clean
sudo dd if=/dev/zero of=/EMPTY bs=1M
sudo rm -f /EMPTY
cat /dev/null > ~/.bash_history && history -c && exit
````

## Export to ova
* Exporting to ova
````
.\ovftool.exe "cuckoo.vmx" cuckoo.ova
````

## References
* https://cuckoo.sh/docs/index.html#
* https://hatching.io/blog/cuckoo-sandbox-setup
* https://tom-churchill.blogspot.com/2017/08/setting-up-cuckoo-sandbox-step-by-step.html
* https://precisionsec.com/virtualbox-host-only-network-cuckoo-sandbox-0-4-2/
* https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-18-04
* https://learning.oreilly.com/library/view/cuckoo-malware-analysis/9781782169239/ch01s05.html
* https://community.spiceworks.com/topic/1984447-trying-to-export-a-vm-but-disk-is-too-big
* https://springmerchant.com/bigcommerce/psycopg2-virtualenv-install-pg_config-executable-not-found/
