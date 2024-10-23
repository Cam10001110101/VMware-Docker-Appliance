# Create a Docker VM on VMware Photon OS

## 1. Create Photon VM
1. Download the Photon OS 5 ISO from the VMware Github repo (Don't use OVA).  
   https://github.com/vmware/photon/wiki/Downloading-Photon-OS
2. Upload your ISO to a folder in your VMware datastore.
3. Create a new VMware virtual machine from the ISO.
4. Install Photon OS 3 as your Docker host.

#### Write File Command Reference
ctrl + c  
:wr + enter  
:qa + enter 

##  2. Configuring a Static IP Address and Restarting Network



## Option 1:
### Open the network configuration file:

vi /etc/systemd/network/99-dhcp-en.network

#### Insert the following content:

[Match]
Name=e*

[Network]
Address=your-photon-vm-ip/24
Gateway=x.x.x.x
DNS=x.x.x.x
Domains=mydomain.local
NTP=pool.ntp.org

#### Set the correct file permissions:

chmod 644 /etc/systemd/network/99-dhcp-en.network

#### Apply the new configuration:

systemctl restart systemd-networkd

#### Verify the network configuration:

ifconfig

## Option 2: 

#### Create the configuration file for eth0:

cat > /etc/systemd/network/10-static-en.network << "EOF"  

[Match]
Name=eth0

[Network]
Address=your-photon-vm-ip/24
Gateway=x.x.x.x.1
EOF

#### Set the correct file permissions:

chmod 644 /etc/systemd/network/10-static-en.network

#### Apply the new configuration:

systemctl restart systemd-networkd


#### Disable DHCP for interfaces matching the pattern e*:

cat > /etc/systemd/network/99-dhcp-en.network << "EOF"
[Match]
Name=e*

[Network]
DHCP=no
EOF

## 3. Rename host

1. Run hostname to show current host name  
2. Edit hosts file by vi /etc/hosts  
3. Press esc then :wq to write file  
4. Reboot  

## 4. Enable Remote Root Login
The ssh daemon does not allow for remote root login by default. If you are OK with not creating special system users, then you need to enable root login by changing “PermitRootLogin no” to “PermitRootLogin yes” in the daemon config file.

#### Edit ssh Daemon Config:  
vi /etc/ssh/sshd_config

#### Search for "PermitRootLogin no" located at line 125. Change it to:  
PermitRootLogin yes

#### Restart sshd:  
systemctl restart sshd  

## 5. Start and Enable Docker

#### Update to Latest Docker Version
yum update -y

#### Start Docker for the First Time
systemctl start docker

#### Enable Docker to Start Automatically
systemctl enable docker

#### Verify
docker info  
docker run hello-world
# GUI Setup

## Option 1: Manage Docker Remotely from Docker Desktop:

## 1. Override docker.service to allow remote connections:

#### Command
systemctl edit docker.service

#### Add or modify the following lines:  
[Service]  
ExecStart=  
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375  

## 2. Save the file
ctrl + c  
:wr + enter  
:qa + enter  

## 3. Reload the systemctl Configuration.
systemctl daemon-reload

## 4. Restart Docker
systemctl restart docker.service

## 5. Verify that the change has gone through.
netstat -lntp | grep dockerd  
tcp        0      0 127.0.0.1:2375          0.0.0.0:*               LISTEN      3758/dockerd

## 6. Save and exit
1. Press Ctrl + X to exit.
2. Press Y to confirm saving the changes.
3. Press Enter to save the file with the same name.
4. Restart Docker:
    systemctl restart docker

## 7. Connect Docker Desktop to Remote Daemon:
1. Open Docker Desktop on your Windows machine.
2. Go to Settings > Docker Engine.
3. Add the following configuration (json):  

```json
{
  "hosts": ["tcp://<PhotonOS_IP>:2375"]
}
```

Default Docker Desktop json config  
```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false
}
```


#### Example:  
Includes default Docker Desktop json config + new remote host
```json
{
    "builder": {
        "gc": {
            "defaultKeepStorage": "20GB",
            "enabled": true
        }
    },
    "experimental": false,
    "hosts": [
        "tcp://x.x.x.x:2375"
    ]
}
```

4. Apply and restart Docker Desktop.

##### Troubleshooting:
1. Review photon logs
sudo journalctl -u docker.service


## Option 2: Portainer GUI Setup:

### Start Docker
systemctl start docker
systemctl enable docker

### Install Portainer:
docker volume create portainer_data
docker run -d -p 9000:9000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
    
### Access Portainer
1. Open a web browser and navigate to http://your-photon-vm-ip:9000
2. Follow the prompts to set up your admin user and connect to the local Docker environment.
#### References  
##### https://vmware.github.io/photon/docs-v5/administration-guide/managing-network-configuration/network-management-commands/
##### https://vmware.github.io/photon/assets/files/html/3.0/photon_admin/setting-a-static-ip-address.html
##### https://docs.docker.com/config/daemon/remote-access/
##### https://docs.docker.com/config/daemon/
##### https://docs.docker.com/reference/cli/dockerd/

#### Notes  
##### VMware Photon OS 5.0_Build=dde71ec57
