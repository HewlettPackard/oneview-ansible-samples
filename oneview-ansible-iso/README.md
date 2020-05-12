ISO installs with OS customizations using ansible and dynamic kickstart files.

1. I found it easier just to package and run everything from my Ansible server so I installed the HTTP server on the Ansible Server.

2. You'll need to PIP install the following, tested with Python 3.6.3/2.7.9 and OV 4.2/5.x and Ansible 2.8.2

hpe3par-sdk         1.2.0 (Optional)
hpICsp              1.0.2
hpOneView           5.0.0b0
python-3parclient   4.2.9 (Optional)
python-hpilo        4.3


3. You'll need to create custom ISO images to allow for unattended installs. The main thing to change is where the ks.cfg file lives. In my example my kickstart files will be generated and places on the Ansible Server under /var/www/html/build directory

4. To create custom ISO images, copy the ISO to a Linux server (Again I just used my Ansible Servers) mount the ISO and copy the contents to a
directory.. ( Mount -o loop esxi.iso /media; cp -rp /media/* /iso)

5. Edit the appropriate files (UEFI/Non-UEFI) to change the location of the ks-cfg

6. There are many examples out on the Web on how to create custom ISO images, I just included 2 links (Google for more!!)

ESXI Example
Edit the boot.cfg under root of the CDROM and under the /efi/boot directory.

bootstate=0
title=Loading ESXi installer
timeout=5
prefix=
kernel=/b.b00
kernelopt=cdromBoot  ks=http://10.10.197.71/media/esxautock/ks_custom67.cfg

On-Line ESXi Docs

https://docs.vmware.com/en/VMware-vSphere/6.7/com.vmware.esxi.upgrade.doc/GUID-C03EADEA-A192-4AB4-9B71-9256A9CB1F9C.html

Linux Example Edit the grub.conf under the EFI/BOOT directory and/or the isolinux.conf file under the isolinux directory



On-Line Linux Docs

http://www.tuxfixer.com/mount-modify-edit-repack-create-uefi-iso-including-kickstart-file/#more-2686


set default="0"

function load_video {
  insmod efi_gop
  insmod efi_uga
  insmod video_bochs
  insmod video_cirrus
  insmod all_video
}

load_video
set gfxpayload=keep
insmod gzio
insmod part_gpt
insmod ext2

set timeout=60
### END /etc/grub.d/00_header ###

search --no-floppy --set=root -l 'RHEL-7.5 Server.x86_64'

### BEGIN /etc/grub.d/10_linux ###
menuentry 'Install Red Hat Enterprise Linux 7.5' --class fedora --class gnu-linux --class gnu --class os {
        linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:LABEL=RHEL-7.5\x20Server.x86_64 quiet inst.ks=http://10.10.69.16/build/rhelks.cfg
        initrdefi /images/pxeboot/initrd.img
}

7. After the ISOs have been created copy them to your Ansible/HTTP server directory.

(These are the actual commands that WORKED!!)

ESXi (Ran from the directory that contained the extracted ISO files)

mkisofs -relaxed-filenames -J -R -b isolinux.bin -c boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -eltorito-alt-boot -e efiboot.img -boot-load-size 1 -no-emul-boot -o /tmp/customesxi.iso .

RHEL (Ran from the /root directory)

mkisofs -o /tmp/rhel75custom.iso -b isolinux/isolinux.bin -J -R -l -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -eltorito-alt-boot -e images/efiboot.img -no-emul-boot -graft-points -V "RHEL-7.5 Server.x86_64" /iso2

8. Create kickstart templates, and add in any variables you want to have replaced. (In the Ansible Playbook only the IP and Hostname variables are defined, but you could add more.)

ESXI kickstart Example

# Sample scripted installation file
# Accept the VMware End User License Agreement
vmaccepteula
# Set the root password for the DCUI and ESXi Shell
rootpw HP1nvent!
# Install on the first local disk available on machine
clearpart --firstdisk --overwritevmfs
install --firstdisk=local --overwritevmfs
# Set the network to DHCP on the first network adapater, use the specified hostname and # Create a portgroup for the VMs
network --bootproto=static --addvmportgroup=1 --ip={{ipaddr}} --netmask=255.255.0.0 --gateway=10.10.1.4 --nameserver=10.10.197.3 --hostname={{hostn}} --device=vmnic0
# reboots the host after the scripted installation is completed
reboot

%firstboot --interpreter=busybox
# Add an extra nic to vSwitch0 (vmnic2)
esxcli network vswitch standard uplink add --uplink-name=vmnic1 --vswitch-name=vSwitch0
# Assign an IP-Address to the first VMkernel, this will be used for management
# esxcli network ip interface ipv4 set --interface-name=vmk0 --type=dhcp
# esxcli network vswitch standard portgroup add --portgroup-name=vMotion --vswitch-# name=vSwitch0
esxcli network vswitch standard portgroup set --portgroup-name=vMotion
esxcli network ip interface add --interface-name=vmk1 --portgroup-name=vMotion
# esxcli network ip interface ipv4 set --interface-name=vmk1 --type=dhcp
# Enable vMotion on the newly created VMkernel vmk1
vim-cmd hostsvc/vmotion/vnic_set vmk1
# Add new vSwitch for VM traffic, assign uplinks, create a portgroup and assign a VLAN ID
# esxcli network vswitch standard add --vswitch-name=vSwitch1
# esxcli network vswitch standard uplink add --uplink-name=vmnic1 --vswitch-name=vSwitch1
# esxcli network vswitch standard uplink add --uplink-name=vmnic3 --vswitch-name=vSwitch1
# esxcli network vswitch standard portgroup add --portgroup-name=Production --vswitch-name=vSwitch1
# esxcli network vswitch standard portgroup set --portgroup-name=Production --vlan-id=10
# Set DNS and hostname
# esxcli system hostname set --fqdn=esxi5.localdomain
esxcli network ip dns search add --domain=atlpss.hp.net
esxcli network ip dns server add --server=10.10.197.3
esxcli network ip dns server add --server=10.10.197.2
# Set the default PSP for EMC V-MAX to Round Robin as that is our preferred load balancing mechanism
# esxcli storage nmp satp set --default-psp VMW_PSP_RR --satp VMW_SATP_SYMM
# Enable SSH and the ESXi Shell
vim-cmd hostsvc/enable_ssh
vim-cmd hostsvc/start_ssh
vim-cmd hostsvc/enable_esx_shell
vim-cmd hostsvc/start_esx_shell


RHEL kickstart Example

lang en_US.UTF-8
keyboard us
timezone --utc America/New_York
text
install
skipx
network  --bootproto=static --ip={{ipaddr}} --netmask=255.255.0.0 --gateway=10.10.1.4 --nameserver=10.10.197.3 --hostname={{hostn}}
authconfig --enable shadow --enablemd5
firstboot --enable
cdrom
rootpw HP1nvent
ignoredisk --only-use=sda
zerombr
clearpart --all --initlabel
autopart --type=lvm
reboot

# Disable firewall and selinux
firewall --disabled
selinux --disabled

%pre
%end

%packages
@Base
@Core
%end