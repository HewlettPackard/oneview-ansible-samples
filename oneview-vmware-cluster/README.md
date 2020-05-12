This is an example of creating a a VMware cluster with customizations, the customization are made by
generating an custom kickstart on the fly. The  ESXi_server_profile_loop.yml playboog takes inputs from
the settings.csv file then loops through vsphere_automate.yml playbook to build out am ESXi cluster using OneView.

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


Tested with
800 API
Ansible 2.8.2
Python > 2.7.9
OneView 5.X
Vsphere 6.7
ESXi 6.7


