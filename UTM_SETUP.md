
# Setting Up UTM on MAC Silicon

We will be setting up some VMs to use in a test K8s cluster using UTM on MAC Silicon. This guide assumes you have already installed UTM.


Download the SHA256SUMS and ubuntu-24.04.1-live-server-arm64.iso files from the (arm archive)[https://cdimage.ubuntu.com/releases/24.04.1/release/]


Check the sha256sums file to ensure the integrity of the downloaded files:
```shell
% sha256sum -c SHA256SUMS
ubuntu-24.04.1-live-server-arm64.iso: OK
```

# Create your first VM

* Click the + button
* Click Virtualize
* Select Linux
* select the arm64 ISO file we just downloaded and click continue
* Give it 2048 MB of RAM and 2 vCPUs and click continue
* Give it 32GiB of disk space and click continue
* name the VM control-01
* Click save

# Configure and install

* Click the setting of the VM we just created
* Go to the network tab
* Select Shared Network and click advance settings so we can define the DHCP range
  * 10.50.50.0/24 as the guest network
  * 10.50.50.100 as the DHCP start
  * 10.50.50.200 as the DHCP end

* Power On the VM
* select Try or Install Ubuntu Server
* Select your language (English)
* Continue without updating the installer
* Select done to select the default keyboard layout
* Leave the selection as ubuntu server and select done
* Select done to accept the default DHCP address
* Select done to have not proxy
* Select done to accept the default mirror
* Select use the entire disk space, Select Setup this disk as a LVM Group and click done to continue
* select done to continue with the default partition table
* enter your user information and put control-01 for the hostname "your servers name"
* select continue to ignore ubuntu pro
* select install open-ssh server and select done to continue
* select done to skip the package selection
* once the installation is complete, click reboot

# Connect to your VM

* The VM will probably fail to reboot. Stop the VM and remove the CDROM from the VM. Once this is done the VM should boot up normally.
* log in to the VM console in UTM and run the `ip a` command to see the IP address of the VM.
* In your Mac terminal ssh to the IP you got from the `ip a` command. e.g. `ssh 10.50.50.100`

```shell
sudo -i;
apt update;
apt upgrade -y;
apt install net-tools dnsutils;
shutdown -h now;
```

# cloning our VM

Its important that we reset a few things as we clone the base VM such as mac address, the machine-id, ssh keys, and hostname. If we don't reinitialize these configurations, the VM will not be able to connect to the internet or join our K8s cluster.

* click the copy button
* on the new copy open settings
  * in the network section click the random button next to the mac address to generate a new mac address
  * Change the name of the new VM to worker-01
* repeat these steps for worker-02 and worker-03

* Start up the VM and SSH to if from your mac terminal
* `sudo -i` to become root
* `vim reset_vm.sh` and copy the following script into it.
* `chmod +x reset_vm.sh` to make it executable
* `./reset_vm.sh` to run the script.
* comment out the HOSTNAME, and IP_ADDRA lines as needed per server.

* ssh and run the script on all four VMs. Once the script is done reboot each the VM.


```shell
#!/bin/bash

set -euxo pipefail

OLD_HOSTNAME="control-01"

HOSTNAME="control-01"
IP_ADDRA="10.50.50.91"
# HOSTNAME="worker-01"
# IP_ADDRA="10.50.50.91"
# HOSTNAME="worker-02"
# IP_ADDRA="10.50.50.92"
# HOSTNAME="worker-03"
# IP_ADDRA="10.50.50.93"

echo "reconfiguring hostname"
echo -n "${HOSTNAME}" > /etc/hostname
sed -i "s/${OLD_HOSTNAME}/${HOSTNAME}/g" /etc/hosts
hostnamectl set-hostname "${HOSTNAME}"

echo "resetting the machine-id"
rm /etc/machine-id
rm /var/lib/dbus/machine-id
systemd-machine-id-setup

echo "resetting the ssh host keys"
rm /etc/ssh/ssh_host_*
dpkg-reconfigure openssh-server

echo "resetting the cloud-init state"
rm -rf /var/lib/cloud/*
cloud-init clean

echo "reconfiguring the network"
cat << EOF > /etc/netplan/00-static.yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: no
      addresses:
        - ${IP_ADDRA}/24
      routes:
        - to: default
          via: 10.50.50.100
      nameservers:
        addresses:
          - 10.50.50.100
EOF

echo "removing the cloud-init netplan configuration file"
rm /etc/netplan/50-cloud-init.yaml

echo "done with VM reset"
```
* reboot each VM
