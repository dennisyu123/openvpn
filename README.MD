
## Setup openvpn on ubuntu 16.04

Step 1: Install OpenVPN

```
sudo apt-get update
sudo apt-get install openvpn easy-rsa
```

Step 2: Set Up the CA Directory

To begin, we can copy the easy-rsa template directory into our home directory with the make-cadir command:
```
make-cadir ~/openvpn-ca
```

Move into the newly created directory to begin configuring the CA:

```
cd openvpn-ca
```

Step 3: Configure the CA Variables

```
nano vars
```

edit the value below

```
# These are the default values for fields
# which will be placed in the certificate.
# Don't leave any of these fields blank.
export KEY_COUNTRY="HK"
export KEY_PROVINCE="HK"
export KEY_CITY="Hong Kong"
export KEY_ORG="MyVpn"
export KEY_EMAIL="dennisyu@mail.com"
export KEY_OU="MyVpn"

# X509 Subject Field
export KEY_NAME="vpnserver"
```

Step 4: Build the Certificate Authority

```
cd ~/openvpn-ca
source vars
```

output
```
NOTE: If you run ./clean-all, I will be doing a rm -rf on /home/sammy/openvpn-ca/keys
```

Make sure we're operating in a clean environment by typing:

```
./clean-all
```

Now, we can build our root CA by typing:
```
./build-ca
```

This will initiate the process of creating the root certificate authority key and certificate. 
Since we filled out the vars file, all of the values should be populated automatically. 
Just press ENTER through the prompts to confirm the selections:

```
Generating a 2048 bit RSA private key
..........................................................................................+++
...............................+++
writing new private key to 'ca.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [US]:
State or Province Name (full name) [NY]:
Locality Name (eg, city) [New York City]:
Organization Name (eg, company) [DigitalOcean]:
Organizational Unit Name (eg, section) [Community]:
Common Name (eg, your name or your server's hostname) [DigitalOcean CA]:
Name [server]:
Email Address [admin@email.com]:
```

Step 5: Create the Server Certificate, Key, and Encryption Files

Note: If you choose a name other than server here, you will have to adjust some of the instructions below. 
For instance, when copying the generated files to the /etc/openvpn directroy, you will have to substitute the correct names. 
You will also have to modify the /etc/openvpn/server.conf file later to point to the correct .crt and .key files.

```
./build-key-server server
```

Once again, the prompts will have default values based on the argument we just passed in (server) and the contents of our vars file we sourced.

Feel free to accept the default values by pressing ENTER. Do not enter a challenge password for this setup. Towards the end, you will have to enter y to two questions to sign and commit the certificate:

```
. . .

Certificate is to be certified until May  1 17:51:16 2026 GMT (3650 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

Afterwards, we can generate an HMAC signature to strengthen the server's TLS integrity verification capabilities:

```
openvpn --genkey --secret keys/ta.key
```

Step 7: Configure the OpenVPN Service

Copy the Files to the OpenVPN Directory

```
cd ~/openvpn-ca/keys
sudo cp ca.crt server.crt server.key ta.key dh2048.pem /etc/openvpn
```

get openvpn server.conf from 
https://github.com/OpenVPN/openvpn/blob/master/sample/sample-config-files/server.conf

copy it to /etc/openvpn/

edit the following things:
```
#if your server is not hosting a website, you can use 443 / 80 as these port won't be blocked by someone
port 443 <- do not use default becuase it is probably blocked by gov / greatwall.

proto udp <- change to tcp if the udp protocol is restricted 

dev tun
;dev-node

ca ca.crt
cert server.crt
key server.key  # This file should be kept secret
dh dh2048.pem
;topology subnet

server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt

push "redirect-gateway def1"
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 1.0.0.1"

keepalive 10 120

tls-auth ta.key 0 # This file is secret
key-direction 0

cipher AES-128-CBC
auth SHA256

user openvpn
group nogroup

persist-key
persist-tun
```

Allow IP Forwarding

First, we need to allow the server to forward traffic. This is fairly essential to the functionality we want our VPN server to provide.

We can adjust this setting by modifying the /etc/sysctl.conf file:

```
sudo nano /etc/sysctl.conf
```

Inside, look for the line that sets net.ipv4.ip_forward. Remove the "#" character from the beginning of the line to uncomment that setting:

```
net.ipv4.ip_forward=1
```

Save and close the file when you are finished.

To read the file and adjust the values for the current session, type:
```
sudo sysctl -p
```

Adjust the UFW Rules to Masquerade Client Connections

Before we open the firewall configuration file to add masquerading, we need to find the public network interface of our machine. To do this, type:
```
ip route | grep default
```

output
```
default via 172.31.16.1 dev eth0 
```

which means this machine is using eth0 network interface

add the relevant configuration:
```
sudo nano /etc/ufw/before.rules
```
```
# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
# Allow traffic from OpenVPN client to eth0
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES
```

We need to tell UFW to allow forwarded packets by default as well. To do this, we will open the /etc/default/ufw file:
```
sudo nano /etc/default/ufw
```

change drop to accept
```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Open the OpenVPN Port and Enable the Changes
edit your port and protocol instead of copying 1194/udp
```
sudo ufw allow 1194/udp
```

Now, we can disable and re-enable UFW to load the changes from all of the files we've modified:
```
sudo ufw disable
sudo ufw enable
```

Step 9: Start and Enable the OpenVPN Service
```
sudo systemctl start openvpn@server
```
Double-check that the service has started successfully by typing:
```
sudo systemctl status openvpn@server
```
If everything went well, enable the service so that it starts automatically at boot:
```
sudo systemctl enable openvpn@server
```


Step 10: Create Client Configuration Infrastructure

Create a directory structure within your home directory to store the files:
```
mkdir -p ~/client-configs/files
chmod 700 ~/client-configs/files
```

Creating a Base Configuration
https://github.com/OpenVPN/openvpn/blob/master/sample/sample-config-files/client.conf

open this file as base.conf
```
nano ~/client-configs/base.conf
```

following need to be edited
```
dev tun
remote our_server_domain port
proto udp

user nobody
group nobody

remote-cert-tls server

#ca ca.crt
#cert client.crt
#key client.key

cipher AES-128-CBC
auth SHA256
key-direction 1

```

Creating a Configuration Generation Script

```
nano ~/client-configs/make_config.sh
```

```
#!/bin/bash

# First argument: Client identifier

KEY_DIR=~/openvpn-ca/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn
```

Mark the file as executable by typing:
```
chmod 700 ~/client-configs/make_config.sh
```

Step 11: Generate Client Configurations

which mac1 is the key you want to build
```
cd ~/openvpn-ca
source vars
./build-key mac1
```

Use the key to build client files
```
cd ~/client-configs
./make_config.sh mac1
```

If everything went well, we should have a mac1.ovpn file in our ~/client-configs/files directory:

```
ls ~/client-configs/files
```

output
```
mac1.ovpn
```
