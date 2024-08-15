# EVE-NG-CE Setup

## End result

 - Multiple independant interfaces:
 1. 1 for accessing the GUI
 2. for connecting the virtual nodes to a separate VLAN (to simulate WAN) or connecting hardware nodes to virtual topologies

 - Self-signed certificate, to encrypt the GUI-traffic + HTTP-Redirect

## Basic requirements

Bare-metal PC / Server with 2 (or more) network-interfaces.

EVE-NG installed on the machine (via official ISO or manually via the script)

(External DHCP-Server)

## Configuration

### Start (EVE-NG-Setup)

>[!Note] The Setup wizard can be restarted using the following command:
>```rm -f /opt/ovf/.configured && su -``` as written in the [FAQ](https://www.eve-ng.net/index.php/faq/)

1. Use the IP that you want want to access the GUI with
(configure dns, ntp etc. aswell for updates etc.)

2. Set the hostname to match your wished URL / FQDN. Eg: eve-ng.labs.local

### Interfaces

#### Primary / "MGMT"-Interface

Leave the "MGMT"-Interface (as configured in the setup) as is.

#### Other Interfaces / "BRIDGES" / Cloud1, Cloud2...

The wished physical interfaces will bridge all traffic. This way, it is possible to connect the virtual nodes to physical routers, switches, dhcp-server, computers etc. that are connected to that port.

1. Connect the EVE-NG-Host via SSH (root-user)

2. Open the /etc/network/interfaces - file

3. Scroll down to the "Cloud"-Interfaces

4. Make sure they are configured as following:

The physical interfaces (eg. eth1, eth2) should be configured to "manual", with no IP set.
The "Cloud"-/"Pnet"-Interfaces should be set to "bridge_ports" and your desired physical port like eth1,eth2,enp1s0 etc.)

Example Cloud1 and 2:
```
iface eth1 inet manual
auto pnet1
iface pnet1 inet manual
  bridge_ports eth1
  bridge_stp off

iface enp1s0 inet manual
auto pnet2
iface pnet2 inet manual
  bridge_ports enp1s0
  bridge_stp off
```

[(Or see the Cloud7-example in the official Cook-Book, on page 135)](https://www.eve-ng.net/images/EVE-COOK-BOOK-1.2.pdf)

5. Apply the changes made to the interfaces
```
sudo systemctl restart NetworkManager.service
```

6. Example of connecting virtual R1 with physical R2 in the LAB:

Add a new network (like via right click). Under "Type" select your Interface (Like Cloud1/eth1), rename it and save.

Then connect the virtual R1 with it. (with the little plug icon)

IRL, connect R2 with your chosen physical interface (eg. eth1/Cloud1)

### Self-signed certificate for the GUI + HTTP-Redirect

#### 1. Generating and installing the cert: [EVE-NG Docs](https://www.eve-ng.net/index.php/documentation/howtos/howto-enable-ssl-eve-community-with-self-sign/)

1. Create certificate

Copy and paste :

```
sudo a2enmod ssl
```
```
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```

fill up all requested fields for certificate

2. Create config files

On CLI, copy/paste following lines:

```
cat << EOF > /etc/apache2/sites-enabled/default-ssl.conf
<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
        ServerAdmin webmaster@localhost
        DocumentRoot /opt/unetlab/html/
        ErrorLog /opt/unetlab/data/Logs/ssl-error.log
        CustomLog /opt/unetlab/data/Logs/ssl-access.log combined
        Alias /Exports /opt/unetlab/data/Exports
        Alias /Logs /opt/unetlab/data/Logs
        SSLEngine on
        SSLCertificateFile    /etc/ssl/certs/apache-selfsigned.crt
        SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
        <FilesMatch "\.(cgi|shtml|phtml|php)$">
                SSLOptions +StdEnvVars
        </FilesMatch>
        <Directory /usr/lib/cgi-bin>
                SSLOptions +StdEnvVars
        </Directory>
        <Location /html5/>
                Order allow,deny
                Allow from all
                ProxyPass http://127.0.0.1:8080/guacamole/ flushpackets=on
                ProxyPassReverse http://127.0.0.1:8080/guacamole/
        </Location>

        <Location /html5/websocket-tunnel>
                Order allow,deny
                Allow from all
                ProxyPass ws://127.0.0.1:8080/guacamole/websocket-tunnel
                ProxyPassReverse ws://127.0.0.1:8080/guacamole/websocket-tunnel
        </Location>
    </VirtualHost>
</IfModule>
EOF
```

3. Restart Apache2

```
/etc/init.d/apache2 restart
```

#### 2. Auto-redirect from port 80:

Add the following line to the  /etc/apache2/sites-avaible/unetlab.conf -file:

```
Redirect permanent "/" "https://<EVE-IG-IP/URL>"
```


Eg:
```
<VirtualHost *:80>

ServerAdmin webmaster@unl01.example.com

DocumentRoot /opt/unetlab/html


# Force HTTPS-Redirect

Redirect permanent "/" "https://10.2.3.100/"



ErrorLog /opt/unetlab/data/Logs/error.txt

CustomLog /opt/unetlab/data/Logs/access.txt combined


remaining content hidden...
....
```


## Custom Node-Templates

Custom Node-Templates for missing or broken devices.

### PaloAlto (AMD64)

```
# CUSTOM PANOS TEMPLATE
# FIXES MANY STARTUP ERRORS 07/2023
---
type: qemu
config_script: config_paloalto.py
description: Palo Alto
cpulimit: 1
icon: PaloAlto.png
cpu: 2
ram: 8192
ethernet: 4
console: telnet
qemu_arch: x86_64
qemu_options: -*machine type=pc,accel=kvm -smbios type=1,product=KVM -serial mon:stdio -nographic -no-user-config -nodefaults -display none -vga std -rtc base=utc -cpu host
...
```

### Juniper vJunos Switch (AMD64)

```
#CUSTOM VJUNOS SWITCH TEMPLATE FOR AMD
# FIXES MANY STARTUP ERRORS 10/2023
---
type: qemu
description: Juniper vEX Switch
name: vEX
cpulimit: 4
icon: JunipervQFXpfe.png
cpu: 4
ram: 8192
eth_name:
- fxp0
eth_format: ge-0/0/{0-9}
ethernet: 11
console: telnet
qemu_arch: x86_64
qemu_version: 5.2.0
qemu_nic: virtio-net-pci
qemu_options: -machine type=pc,accel=kvm -serial mon:stdio -nographic -smbios type=1,product=VM-VEX -cpu host,ibpb=on,md-clear=on,spec-ctrl=on,ssbd=on,vmx=on
...
```

## Other Known Errors / Workarounds for Nodes

### Palo-Alto-NGFW (PANOS) - Multiple Fixes for various errors:

Correct QEMU-Options:
```
-*machine type=pc,accel=kvm -smbios type=1,product=KVM -serial mon:stdio -nographic -no-user-config -nodefaults -display none -vga std -rtc base=utc -cpu host*
```

Give it atleast 8GB RAM and 2 Cores.

After starting the PANOS-Node:

As soon as you see the CLI-Login, spam the Return-/"Enter"-Key.

After a couple of minutes of doing so, the correct Login should appear.


### VyOS not starting

After having followed the [install-guide for VyOS]() and added the final Node to the lab, it won't start.

Fix:

After adding the Node to the lab, right-click it and select "Wipe", then it should start fine.
