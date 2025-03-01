
# Setting Up UTM on MAC Silicon

We will be setting up some VMs to use in a test K8s cluster using UTM on MAC Silicon. This guide assumes you have already installed UTM.


Download the SHA256SUMS and ubuntu-24.04.1-live-server-arm64.iso files from the [arm archive](https://cdimage.ubuntu.com/releases/24.04.1/release/)


Check the sha256sums file to ensure the integrity of the downloaded files:
```shell
% sha256sum -c SHA256SUMS
ubuntu-24.04.1-live-server-arm64.iso: OK
```

# Create your first VM

* Click the + button
* Click Virtualize
* Select Linux
* Select the arm64 ISO file we just downloaded and click continue
* Give it 2048 MB of RAM and 2 vCPUs and click continue
* Give it 32GiB of disk space and click continue
* name the VM control-01
* Click save

# Configure and install

* Click the settings button of the VM we just created
* Go to the network tab
* Select `Shared Network` and click advance settings so we can define the DHCP range
  * 10.50.50.0/24 as the guest network
  * 10.50.50.100 as the DHCP start
  * 10.50.50.200 as the DHCP end

* Power On the VM
* Select Try or Install Ubuntu Server
* Select your language (English)
* Continue without updating the installer
* Select done to select the default keyboard layout
* Leave the selection as ubuntu server and select done
* Select done to accept the default DHCP address
* Select done to have no proxy settings
* Select done to accept the default packages mirror
* Select use the entire disk space, Select Setup this disk as a LVM Group and click done to continue
* Select done to continue with the default partition table
* Enter your user information and put control-01 for the hostname "your servers name"
* Select continue to ignore ubuntu pro
* Select install open-ssh server and select done to continue
* Select done to skip the package selection
* Once the installation is complete, click reboot

# Connect to your VM

* The VM will probably fail to reboot. Stop the VM and remove the CDROM from the VM. Once this is done the VM should boot up normally.
* log in to the VM console in UTM and run the `ip a` command to see the IP address of the VM.
* In your Mac terminal ssh to the IP you got from the `ip a` command. e.g. `ssh 10.50.50.100`

```shell
$ sudo -i;
# apt update;
# apt upgrade -y;
# apt install net-tools dnsutils;
# shutdown -h now;
```

# cloning our VM

Its important that we reset a few things as we clone the base VM such as mac address, the machine-id, ssh keys, and hostname. If we don't reinitialize these configurations, the VM will not be able to connect to the internet or join our K8s cluster.

* Click the copy button
* On the new copy open settings
  * In the network section click the random button next to the mac address to generate a new mac address
  * Change the name of the new VM to the coresponding control/worker hostname
* Repeat these steps until you have cloned all your VMs.. control-01, control-02, control-03, worker-01, worker-02, worker-03

* Start up the VM and SSH to if from your mac terminal
* `sudo -i` to become root
* `vim reset_vm.sh` and copy the following script into it.
* `chmod +x reset_vm.sh` to make it executable
* `./reset_vm.sh` to run the script.
* Comment in/out the HOSTNAME, and IP_ADDRA lines as needed per server.

* ssh and run the script on all VMs, even the origonal control-01 VM. Once the script is done reboot each VM.

```shell
#!/bin/bash

set -euxo pipefail

OLD_HOSTNAME="control-01"

HOSTNAME="control-01"
IP_ADDRA="10.50.50.81"
# HOSTNAME="control-02"
# IP_ADDRA="10.50.50.82"
# HOSTNAME="control-03"
# IP_ADDRA="10.50.50.83"
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
echo 'network: {config: disabled}' > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
rm /etc/netplan/50-cloud-init.yaml

echo "done with VM reset"
```
* reboot each VM

* The Base VM Image is now ready to be configured with ansible. See the [snowkiterdude/kube_ansible](https://github.com/snowkiterdude/kube_ansible) repo for future configuration to bootstrap these VMs into a Kubernetes cluster.


# Creating snapshot

UTM Does not have snapshot support however we can use the qemu-img command to create snapshots of the disk images. Before creating or restoring a snapshot, we need to ensure that the disk images are not mounted so make sure the VM is not running.

Install qemu
```shell
brew install qemu
```

First CD into the UTM data directory
```shell
% cd ~/Library/Containers/com.utmapp.UTM/Data/Documents;
% ls -l
drwxr-xr-x@  5 pete  staff  160 Feb 11 23:37 control-01.utm
drwxr-xr-x@  5 pete  staff  160 Feb 11 23:37 worker-01.utm
drwxr-xr-x@  5 pete  staff  160 Feb 11 23:37 worker-02.utm
drwxr-xr-x@  5 pete  staff  160 Feb 11 23:38 worker-03.utm
```

The images are inside the `<vm-name>.utm/Data/` directory. we will be using the after_init name for our snapshot name in these examples.

Create a snapshot of the disk image
```shell
qemu-img snapshot -c after_init control-01.utm/Data/ABCD-1234.qcow2
```

List the snapshots of the disk image
```shell
% qemu-img snapshot -l control-01.utm/Data/ABCD-1234.qcow2
Snapshot list:
ID      TAG               VM_SIZE                DATE        VM_CLOCK     ICOUNT
1       after_init            0 B 2025-02-11 23:55:02  0000:00:00.000          0
```

Restore a snapshot
```shell
% qemu-img snapshot -a after_init control-01.utm/Data/ABCD-1234.qcow2
```

Delete specific snapshot
```shell
% qemu-img snapshot -d after_init control-01.utm/Data/ABCD-1234.qcow2
```

Delete all snapshots
```shell
% qemu-img snapshot -D control-01.utm/Data/ABCD-1234.qcow2
```

Shutdown and revert the snapshot for all nodes.
since we cloned all our images from control-01 they will all have the same image UUID
Change the image UUID to match your setup by listing one of the images `ls control-01.utm/Data/`
shutdown all your VMs then run the following command to revert the snapshot for all VMs.
```shell
export IMAGE_UUID="ABCD-1234"
for i in {control-0{1..3},worker-0{1..3}}; do
qemu-img snapshot -a after_init "${i}.utm/Data/${IMAGE_UUID}.qcow2"
done
```
