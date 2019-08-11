# CuckooSetup

## Hardware requirements
* Ubuntu 18.04 LTS
* Quad Core CPU
* 4 GB RAM
* 320 GB HDD

## Initial Setup
````
sudo apt-get update
sudo apt-get -y install python virtualenv python-pip python-dev build-essential
````
````
sudo adduser --disabled-password --gecos "" cuckoo
````
````
sudo groupadd pcap
sudo usermod -a -G pcap cuckoo
sudo chgrp pcap /usr/sbin/tcpdump
sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
````
````
sudo apt-get install -y apparmor-utils
sudo aa-disable /usr/sbin/tcpdump
````
wget https://cuckoo.sh/win7ultimate.iso
mkdir /mnt/win7
sudo mount -o ro,loop win7ultimate.iso /mnt/win7
````
## Installing VirtualBox
wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- | sudo apt-key add -
wget -q https://www.virtualbox.org/download/oracle_vbox.asc -O- | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] http://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib"

sudo apt-get update
sudo apt-get install virtualbox-5.2
sudo usermod -a -G vboxusers cuckoo

## Cuckoo and VMCloak installation
sudo apt-get -y install build-essential libssl-dev libffi-dev python-dev genisoimage
sudo apt-get -y install zlib1g-dev libjpeg-dev
sudo apt-get -y install python-pip python-virtualenv python-setuptools swig

sudo su cuckoo
virtualenv ~/cuckoo
. ~/cuckoo/bin/activate

pip install -U cuckoo vmcloak

## Automatic VM creation
vmcloak-vboxnet0

vmcloak init --verbose --win7x64 win7x64base --cpus 2 --ramsize 2048

vmcloak clone win7x64base win7x64cuckoo

vmcloak install win7x64cuckoo adobepdf pillow dotnet java flash vcredist vcredist.version=2015u3 wallpaper

vmcloak install win7x64cuckoo ie11

vmcloak install win7x64cuckoo office office.version=2007 office.isopath=/path/to/office2007.iso office.serialkey=XXXXX-XXXXX-XXXXX-XXXXX-XXXXX

vmcloak snapshot --count 4 win7x64cuckoo win7x64cuckoo_ 192.168.56.101

vmcloak list vms

## Configuring Cuckoo
cuckoo init

## Postgres installation
sudo apt install postgresql postgresql-contrib

sudo apt-get install libpq-dev python-dev

pip install psycopg2

sudo -u postgres psql
CREATE DATABASE cuckoo;
CREATE USER cuckoo WITH ENCRYPTED PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE cuckoo TO cuckoo;
\q

nano /home/cuckoo/.cuckoo/conf/cuckoo.conf

#Change connection =  postgresql://cuckoo:password@localhost/cuckoo

## Adding VMs
nano /home/cuckoo/.cuckoo/conf/virtualbox.conf

#remove the entry cuckoo1 in the machines = cuckoo1

while read -r vm ip; do cuckoo machine --add $vm $ip; done < <(vmcloak list vms)

cuckoo community --force

## Network configuration
#change outgoing interface
sudo sysctl -w net.ipv4.conf.vboxnet0.forwarding=1
sudo sysctl -w net.ipv4.conf.outgoinginterface.forwarding=1

#change outgoing interface
sudo iptables -t nat -A POSTROUTING -o outgoinginterface -s 192.168.56.0/24 -j MASQUERADE
sudo iptables -P FORWARD DROP
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -s 192.168.56.0/24 -j ACCEPT
sudo apt-get install -y iptables-persistent


sudo gedit /etc/sysctl.conf
#uncomment the line net.ipv4.ip_forward=1


sudo apt-get install -y vim
sudo mkdir /opt/systemd/
sudo vim /opt/systemd/vboxhostonly
#!/bin/bash
vboxmanage hostonlyif create
vboxmanage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1

cd /opt/systemd/
sudo chmod a+x vboxhostonly

sudo touch /etc/systemd/system/vboxhostonlynic.service
sudo gedit /etc/systemd/system/vboxhostonlynic.service

Description=Setup VirtualBox Hostonly Adapter

After=vboxdrv.service



[Service]

Type=oneshot

ExecStart=/opt/systemd/vboxhostonly



[Install]

WantedBy=multi-user.target

systemctl daemon-reload
systemctl enable vboxhostonlynic.service
systemctl start vboxhostonlynic.service

## Cuckoo Web Interface
sudo apt-get install mongodb

#[MongoDB] section. Change enabled = no to enabled = yes
nano /home/cuckoo/.cuckoo/conf/virtualbox.conf

pip install uwsgi

sudo apt-get install uwsgi uwsgi-plugin-python nginx

cuckoo web --uwsgi > cuckoo-web.ini
sudo cp cuckoo-web.ini /etc/uwsgi/apps-available/cuckoo-web.ini
sudo ln -s /etc/uwsgi/apps-available/cuckoo-web.ini /etc/uwsgi/apps-enabled/cuckoo-web.ini

sudo adduser www-data cuckoo
sudo systemctl restart uwsgi


cuckoo web --nginx > cuckoo-web.conf
#edit cuckoo-web.conf to listen *:8000;
nano cuckoo-web.conf
sudo cp cuckoo-web.conf /etc/nginx/sites-available/cuckoo-web.conf
sudo ln -s /etc/nginx/sites-available/cuckoo-web.conf /etc/nginx/sites-enabled/cuckoo-web.conf
sudo systemctl restart nginx

## Starting Cuckoo
sudo su cuckoo
. ~/cuckoo/bin/activate
cuckoo --debug

## Freeing up space and exporting Sandbox to ova
sudo apt-get clean
sudo dd if=/dev/zero of=/EMPTY bs=1M
sudo rm -f /EMPTY
cat /dev/null > ~/.bash_history && history -c && exit

.\ovftool.exe "cuckoo.vmx" cuckoo.ova

## References
https://hatching.io/blog/cuckoo-sandbox-setup
https://tom-churchill.blogspot.com/2017/08/setting-up-cuckoo-sandbox-step-by-step.html
https://precisionsec.com/virtualbox-host-only-network-cuckoo-sandbox-0-4-2/
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-18-04
https://learning.oreilly.com/library/view/cuckoo-malware-analysis/9781782169239/ch01s05.html
https://community.spiceworks.com/topic/1984447-trying-to-export-a-vm-but-disk-is-too-big
https://springmerchant.com/bigcommerce/psycopg2-virtualenv-install-pg_config-executable-not-found/
