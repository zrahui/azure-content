<properties 
	pageTitle="Create and upload a Red Hat Enterprise Linux VHD for use in Azure" 
	description="Learn to create and upload an Azure virtual hard disk (VHD) that contains a Red Hat Linux operating system." 
	services="virtual-machines" 
	documentationCenter="" 
	authors="SuperScottz" 
	manager="timlt" 
	editor="tysonn"/>

<tags 
	ms.service="virtual-machines" 
	ms.workload="infrastructure-services" 
	ms.tgt_pltfrm="vm-linux" 
	ms.devlang="na" 
	ms.topic="article" 
	ms.date="11/23/2015" 
	ms.author="mingzhan"/>


# Prepare a Red Hat-based Virtual Machine for Azure
In this article, you will learn how to prepare a Red Hat Enterprise Linux (RHEL) Virtual Machine for for use in Azure.  Versions of RHEL covered in this article are 6.7, 7.1 and 7.2 and hypervisors for preparation covered in this article are Hyper-V, KVM and VMWare.  For more information on eligibility requirements for participating in Red Hat's Cloud Access program, see [Red Hat's Cloud Access website](http://www.redhat.com/en/technologies/cloud-computing/cloud-access) and [Running RHEL on Azure](https://access.redhat.com/articles/1989673). 




##Prepare an image from Hyper-V Manager 
###Prerequisites
This section assumes that you have already installed a RHEL image from an ISO file obtained from Red Hat's website to a virtual hard disk (VHD). For more details on how to use Hyper-V manager to install an operating system image, see [Install the Hyper-V Role and Configure a Virtual Machine](http://technet.microsoft.com/library/hh846766.aspx). 

**RHEL Installation Notes**

- The newer VHDX format is not supported in Azure. You can convert the disk to VHD format using Hyper-V Manager or the convert-vhd powershell cmdlet.

- When installing the Linux system it is recommended that you use standard partitions rather than LVM (often the default for many installations). This will avoid LVM name conflicts with cloned VMs, particularly if an OS disk ever needs to be attached to another VM for troubleshooting. LVM or RAID may be used on data disks if preferred.

- Do not configure a swap partition on the OS disk. The Linux agent can be configured to create a swap file on the temporary resource disk. More information about this can be found in the steps below.

- All of the VHDs must have sizes that are multiples of 1 MB.

- When using qemu-img to convert disk images to VHD format, note that there is a known bug in qemu-img versions >=2.2.1 that results in an improperly formatted VHD. The issue will be fixed in an upcoming release of qemu-img.  For now it is recommended to use qemu-img version 2.2.0 or lower.


###RHEL 6.7

1.	In Hyper-V Manager, select the virtual machine.

2.	Click **Connect** to open a console window for the virtual machine.

3.	Uninstall NetworkManager by running the following command:

        # sudo rpm -e --nodeps NetworkManager

    **Note:** If the package is not already installed, this command will fail with an error message. This is expected.

4.	Create a file named **network** in the `/etc/sysconfig/` directory  that contains the following text:

        NETWORKING=yes
        HOSTNAME=localhost.localdomain

5.	Create a file named **ifcfg-eth0** in the `/etc/sysconfig/network-scripts/` directory that contains the following text:

        DEVICE=eth0
        ONBOOT=yes
        BOOTPROTO=dhcp
        TYPE=Ethernet
        USERCTL=no
        PEERDNS=yes
        IPV6INIT=no

6.	Move (or remove) udev rules to avoid generating static rules for the Ethernet interface. These rules cause problems when cloning a virtual machine in Microsoft Azure or Hyper-V:
            
        # sudo mkdir -m 0700 /var/lib/waagent
        # sudo mv /lib/udev/rules.d/75-persistent-net-generator.rules /var/lib/waagent/
        # sudo mv /etc/udev/rules.d/70-persistent-net.rules /var/lib/waagent/

7.	Ensure the network service will start at boot time by running the following command:

        # sudo chkconfig network on

8.	Register your Red Hat subscription to enable installation of packages from the RHEL repository by running the following command:

        # sudo subscription-manager register --auto-attach --username=XXX --password=XXX

9.	The WALinuxAgent package `WALinuxAgent-<version>` has been pushed to the Fedora EPEL 6 repository. Enable the epel repository by running the following command:

        # wget http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
        # rpm -ivh epel-release-6-8.noarch.rpm

10.	Modify the kernel boot line in your grub configuration to include additional kernel parameters for Azure. To do this open `/boot/grub/menu.lst` in a text editor and ensure that the default kernel includes the following parameters:

        console=ttyS0 
        earlyprintk=ttyS0 
        rootdelay=300 
        numa=off

    This will also ensure all console messages are sent to the first serial port, which can assist Azure support with debugging issues. This will disable NUMA due to a bug in the kernel version used by RHEL 6.

    In addition to the above, it is recommended to remove the following parameters:

        rhgb quiet crashkernel=auto

    Graphical and quiet boot are not useful in a cloud environment where we want all the logs to be sent to the serial port.

    The crashkernel option may be left configured if desired, but note that this parameter will reduce the amount of available memory in the VM by 128MB or more, which may be problematic on the smaller VM sizes.

11.	Ensure that the SSH server is installed and configured to start at boot time. This is usually the default. Modify the /etc/ssh/sshd_config to include following line:

        ClientAliveInterval 180

12.	Install the Azure Linux Agent by running the following command:

        # sudo yum install WALinuxAgent
        # sudo chkconfig waagent on

    **Note:** that installing the WALinuxAgent package will remove the NetworkManager and NetworkManager-gnome packages if they were not already removed as described in step 2.

13.	Do not create swap space on the OS disk
The Azure Linux Agent can automatically configure swap space using the local resource disk that is attached to the VM after provisioning on Azure. Note that the local resource disk is a temporary disk, and might be emptied when the VM is deprovisioned. After installing the Azure Linux Agent (see previous step), modify the following parameters in /etc/waagent.conf appropriately:

        ResourceDisk.Format=y
        ResourceDisk.Filesystem=ext4
        ResourceDisk.MountPoint=/mnt/resource
        ResourceDisk.EnableSwap=y
        ResourceDisk.SwapSizeMB=2048    ## NOTE: set this to whatever you need it to be.

14.	Unregister the subscription (if necessary) by run the following command:

        # sudo subscription-manager unregister

15.	Run the following commands to deprovision the virtual machine and prepare it for provisioning on Azure:

        # sudo waagent -force -deprovision
        # export HISTSIZE=0
        # logout

16.	Click **Action -> Shut Down** in Hyper-V Manager. Your Linux VHD is now ready to be uploaded to Azure.
 

###RHEL 7.1/7.2

1.  In Hyper-V Manager, select the virtual machine.

2.	Click Connect to open a console window for the virtual machine.

3.	Create a file named **network** in the `/etc/sysconfig/` directory that contains the following text:

        NETWORKING=yes
        HOSTNAME=localhost.localdomain

4.	Create a file named **ifcfg-eth0** in the `/etc/sysconfig/network-scripts/` directory that contains the following text:

        DEVICE=eth0
        ONBOOT=yes
        BOOTPROTO=dhcp
        TYPE=Ethernet
        USERCTL=no
        PEERDNS=yes
        IPV6INIT=no

5.	Ensure the network service will start at boot time by running the following command:

        # sudo chkconfig network on

6.	Register your Red Hat subscription to enable installation of packages from the RHEL repository by running the following command:

        # sudo subscription-manager register --auto-attach --username=XXX --password=XXX

7.	Modify the kernel boot line in your grub configuration to include additional kernel parameters for Azure. To do this open `/etc/default/grub` in a text editor and edit the **GRUB_CMDLINE_LINUX** parameter, for example:

        GRUB_CMDLINE_LINUX="rootdelay=300 
        console=ttyS0 
        earlyprintk=ttyS0"

    This will also ensure all console messages are sent to the first serial port, which can assist Azure support with debugging issues. In addition to the above, it is recommended to remove the following parameters:

        rhgb quiet crashkernel=auto
    Graphical and quiet boot are not useful in a cloud environment where we want all the logs to be sent to the serial port.
    The crashkernel option may be left configured if desired, but note that this parameter will reduce the amount of available memory in the VM by 128MB or more, which may be problematic on the smaller VM sizes.

8.	Once you are done editing `/etc/default/grub` per above, run the following command to rebuild the grub configuration:

        # sudo grub2-mkconfig -o /boot/grub2/grub.cfg

9.	Ensure that the SSH server is installed and configured to start at boot time. This is usually the default. Modify the `/etc/ssh/sshd_config` to include following line:

        ClientAliveInterval 180

10.	The WALinuxAgent package `WALinuxAgent-<version>` has been pushed to the Fedora EPEL 6 repository. Enable the epel repository by running the following command:

        # wget http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
        # rpm -ivh epel-release-7-5.noarch.rpm

11.	Install the Azure Linux Agent by running the following command:

        # sudo yum install WALinuxAgent
        # sudo systemctl enable waagent.service 

12.	Do not create swap space on the OS disk. The Azure Linux Agent can automatically configure swap space using the local resource disk that is attached to the VM after provisioning on Azure. Note that the local resource disk is a temporary disk, and might be emptied when the VM is deprovisioned. After installing the Azure Linux Agent (see previous step), modify the following parameters in `/etc/waagent.conf` appropriately:

        ResourceDisk.Format=y
        ResourceDisk.Filesystem=ext4
        ResourceDisk.MountPoint=/mnt/resource
        ResourceDisk.EnableSwap=y
        ResourceDisk.SwapSizeMB=2048    ## NOTE: set this to whatever you need it to be.

13.	If you want to unregister the subscription, run the following command:

        # sudo subscription-manager unregister

14.	Run the following commands to deprovision the virtual machine and prepare it for provisioning on Azure:
        
        # sudo waagent -force -deprovision
        # export HISTSIZE=0
        # logout

15.	Click **Action -> Shut Down** in Hyper-V Manager. Your Linux VHD is now ready to be uploaded to Azure.
 


##Prepare an image from KVM 
###RHEL 6.7

1.	Download the KVM image of RHEL 6.7 from Red Hat's web site.

2.	Set a root password

    Generate encrypted password, and copy the output of the command:

        # openssl passwd -1 changeme
    Set a root password with guestfish:
       
        # guestfish --rw -a <image-name>
        ><fs> run
        ><fs> list-filesystems
        ><fs> mount /dev/sda1 /
        ><fs> vi /etc/shadow
        ><fs> exit
    Change the second field of root user from “!!” to the encrypted password.

3.	Create a virtual machine in KVM from the qcow2 image, set the disk type to **qcow2**, set the Virtual Network Interface device model to **virtio**. Then start the virtual machine and log in as root.

4.	Create a file named **network** in the `/etc/sysconfig/` directory that contains the following text:

        NETWORKING=yes
        HOSTNAME=localhost.localdomain

5.	Create a file named **ifcfg-eth0** in the `/etc/sysconfig/network-scripts/` directory that contains the following text:

        DEVICE=eth0
        ONBOOT=yes
        BOOTPROTO=dhcp
        TYPE=Ethernet
        USERCTL=no
        PEERDNS=yes
        IPV6INIT=no

6.	Move (or remove) udev rules to avoid generating static rules for the Ethernet interface. These rules cause problems when cloning a virtual machine in Microsoft Azure or Hyper-V:

        # mkdir -m 0700 /var/lib/waagent
        # mv /lib/udev/rules.d/75-persistent-net-generator.rules /var/lib/waagent/
        # mv /etc/udev/rules.d/70-persistent-net.rules /var/lib/waagent/

7.	Ensure the network service will start at boot time by running the following command:

        # chkconfig network on

8.	Register your Red Hat subscription to enable installation of packages from the RHEL repository by running the following command:

        # subscription-manager register --auto-attach --username=XXX --password=XXX

9.	Modify the kernel boot line in your grub configuration to include additional kernel parameters for Azure. To do this open `/boot/grub/menu.lst` in a text editor and ensure that the default kernel includes the following parameters:

        console=ttyS0 earlyprintk=ttyS0 rootdelay=300 numa=off

    This will also ensure all console messages are sent to the first serial port, which can assist Azure support with debugging issues. This will disable NUMA due to a bug in the kernel version used by RHEL 6.

    In addition to the above, it is recommended to remove the following parameters:

        rhgb quiet crashkernel=auto

    Graphical and quiet boot are not useful in a cloud environment where we want all the logs to be sent to the serial port.
    The crashkernel option may be left configured if desired, but note that this parameter will reduce the amount of available memory in the VM by 128MB or more, which may be problematic on the smaller VM sizes.

10.	Uninstall cloud-init:

        # yum remove cloud-init

11.	Ensure that the SSH server is installed and configured to start at boot time:
 
        # chkconfig sshd on

    Modify the /etc/ssh/sshd_config to include following lines:

        PasswordAuthentication yes
        ClientAliveInterval 180

    Restart sshd:

		# service sshd restart

12.	The WALinuxAgent package `WALinuxAgent-<version>` has been pushed to the Fedora EPEL 6 repository. Enable the epel repository by running the following command:

        # wget http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
        # rpm -ivh epel-release-6-8.noarch.rpm

13.	Install the Azure Linux Agent by running the following command:

        # yum install WALinuxAgent
        # chkconfig waagent on

14.	The Azure Linux Agent can automatically configure swap space using the local resource disk that is attached to the VM after provisioning on Azure. Note that the local resource disk is a temporary disk, and might be emptied when the VM is deprovisioned. After installing the Azure Linux Agent (see previous step), modify the following parameters in **/etc/waagent.conf** appropriately:

        ResourceDisk.Format=y
        ResourceDisk.Filesystem=ext4
        ResourceDisk.MountPoint=/mnt/resource
        ResourceDisk.EnableSwap=y
        ResourceDisk.SwapSizeMB=2048    ## NOTE: set this to whatever you need it to be.

15.	Unregister the subscription (if necessary) by running the following command:
        
        # subscription-manager unregister

16.	Run the following commands to deprovision the virtual machine and prepare it for provisioning on Azure:

        # waagent -force -deprovision
        # export HISTSIZE=0
        # logout

17.	Shut down the VM in KVM.

18.	Convert the qcow2 image to vhd format:
    First convert the image to raw format:
         
         # qemu-img convert -f qcow2 –O raw rhel-6.7.qcow2 rhel-6.7.raw
    Make sure the size of raw image is aligned with 1MB, otherwise round up the size to align with 1MB:
         # MB=$((1024*1024))
         # size=$(qemu-img info -f raw --output json "rhel-6.7.raw" | \
                  gawk 'match($0, /"virtual-size": ([0-9]+),/, val) {print val[1]}')
         # rounded_size=$((($size/$MB + 1)*$MB))

         # qemu-img resize rhel-6.7.raw $rounded_size

    Convert the raw disk to fixed-sized vhd:

         # qemu-img convert -f raw -o subformat=fixed -O vpc rhel-6.7.raw rhel-6.7.vhd


###RHEL 7.1/7.2

1.	Download the KVM image of RHEL 7.1(or 7.2) from the Red Hat web site, we will use RHEL 7.1 as the example here.

2.	Set a root password

    Generate encrypted password, and copy the output of the command

        # openssl passwd -1 changeme

    Set a root password with guestfish

        # guestfish --rw -a <image-name>
        ><fs> run
        ><fs> list-filesystems
        ><fs> mount /dev/sda1 /
        ><fs> vi /etc/shadow
        ><fs> exit

    Change the second field of root user from “!!” to the encrypted password

3.	Create a virtual machine in KVM from the qcow2 image, set the disk type to **qcow2**, set the Virtual Network Interface device model to **virtio**. Then start the virtual machine and log in as root.

4.	Create a file named **network** in the `/etc/sysconfig/` directory that contains the following text:

        NETWORKING=yes
        HOSTNAME=localhost.localdomain

5.	Create a file named **ifcfg-eth0** in the `/etc/sysconfig/network-scripts/` directory that contains the following text:

        DEVICE=eth0
        ONBOOT=yes
        BOOTPROTO=dhcp
        TYPE=Ethernet
        USERCTL=no
        PEERDNS=yes
        IPV6INIT=no

6.	Ensure the network service will start at boot time by running the following command:

        # chkconfig network on

7.	Register your Red Hat subscription to enable installation of packages from the RHEL repository by running the following command:

        # subscription-manager register --auto-attach --username=XXX --password=XXX

8.	Modify the kernel boot line in your grub configuration to include additional kernel parameters for Azure. To do this open `/etc/default/grub` in a text editor and edit the **GRUB_CMDLINE_LINUX** parameter, for example:

        GRUB_CMDLINE_LINUX="rootdelay=300 
        console=ttyS0 
        earlyprintk=ttyS0"

    This will also ensure all console messages are sent to the first serial port, which can assist Azure support with debugging issues. In addition to the above, it is recommended to remove the following parameters:

        rhgb quiet crashkernel=auto

    Graphical and quiet boot are not useful in a cloud environment where we want all the logs to be sent to the serial port.
    The crashkernel option may be left configured if desired, but note that this parameter will reduce the amount of available memory in the VM by 128MB or more, which may be problematic on the smaller VM sizes.

9.	Once you are done editing `/etc/default/grub` per above, run the following command to rebuild the grub configuration:

        # grub2-mkconfig -o /boot/grub2/grub.cfg
        
10.	Add Hyper-V modules into initramfs: 

    Edit `/etc/dracut.conf`, add content:

        add_drivers+=”hv_vmbus hv_netvsc hv_storvsc”

    Rebuild the initramfs:

        # dracut –f -v
        
11.	Uninstall cloud-init:

        # yum remove cloud-init

12.	Ensure that the SSH server is installed and configured to start at boot time: 

        # systemctl enable sshd

    Modify the /etc/ssh/sshd_config to include following lines:

        PasswordAuthentication yes
        ClientAliveInterval 180

    Restart sshd:

        systemctl restart sshd	

13.	The WALinuxAgent package `WALinuxAgent-<version>` has been pushed to the Fedora EPEL 6 repository. Enable the epel repository by running the following command:

        # wget http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
        # rpm -ivh epel-release-7-5.noarch.rpm

14.	Install the Azure Linux Agent by running the following command:

        # yum install WALinuxAgent

    Enable the waagent service:

        # systemctl enable waagent.service

15.	Do not create swap space on the OS disk. The Azure Linux Agent can automatically configure swap space using the local resource disk that is attached to the VM after provisioning on Azure. Note that the local resource disk is a temporary disk, and might be emptied when the VM is deprovisioned. After installing the Azure Linux Agent (see previous step), modify the following parameters in `/etc/waagent.conf` appropriately:

        ResourceDisk.Format=y
        ResourceDisk.Filesystem=ext4
        ResourceDisk.MountPoint=/mnt/resource
        ResourceDisk.EnableSwap=y
        ResourceDisk.SwapSizeMB=2048    ## NOTE: set this to whatever you need it to be.

16.	Unregister the subscription(if necessary) by running the following command:

        # subscription-manager unregister

17.	Run the following commands to deprovision the virtual machine and prepare it for provisioning on Azure:

        # sudo waagent -force -deprovision
        # export HISTSIZE=0
        # logout

18.	Shut down the virtual machine in KVM.

19.	Convert the qcow2 image to vhd format:

    First convert the image to raw format:

         # qemu-img convert -f qcow2 –O raw rhel-7.1.qcow2 rhel-7.1.raw

    Make sure the size of raw image is aligned with 1MB, otherwise round up the size to align with 1MB:

         # MB=$((1024*1024))
         # size=$(qemu-img info -f raw --output json "rhel-7.1.raw" | \
                  gawk 'match($0, /"virtual-size": ([0-9]+),/, val) {print val[1]}')
         # rounded_size=$((($size/$MB + 1)*$MB))

         # qemu-img resize rhel-7.1.raw $rounded_size

    Convert the raw disk to fixed-sized vhd:

         # qemu-img convert -f raw -o subformat=fixed -O vpc rhel-7.1.raw rhel-7.1.vhd


##Prepare an image from VMWare
###Prerequisites
This section assumes that you have already installed a RHEL virtual machine in VMWare. For details on how to install an operating system in VMWare, please see [VMWare Guest Operating System Installation Guide](http://partnerweb.vmware.com/GOSIG/home.html).
 
- When installing the Linux system it is recommended that you use standard partitions rather than LVM (often the default for many installations). This will avoid LVM name conflicts with cloned VMs, particularly if an OS disk ever needs to be attached to another VM for troubleshooting. LVM or RAID may be used on data disks if preferred.

- Do not configure a swap partition on the OS disk. The Linux agent can be configured to create a swap file on the temporary resource disk. More information about this can be found in the steps below.

- When creating the virtual hard disk, select **Store virtual disk as a single file**.

###RHEL 6.7
1.	Uninstall NetworkManager by running the following command:

         # sudo rpm -e --nodeps NetworkManager

    **Note:** If the package is not already installed, this command will fail with an error message. This is expected.

2.	Create a file named **network** in the /etc/sysconfig/ directory that contains the following text:

        NETWORKING=yes
        HOSTNAME=localhost.localdomain

3.	Create a file named **ifcfg-eth0** in the /etc/sysconfig/network-scripts/ directory that contains the following text:

        DEVICE=eth0
        ONBOOT=yes
        BOOTPROTO=dhcp
        TYPE=Ethernet
        USERCTL=no
        PEERDNS=yes
        IPV6INIT=no

4.	Move (or remove) udev rules to avoid generating static rules for the Ethernet interface. These rules cause problems when cloning a virtual machine in Microsoft Azure or Hyper-V:

        # sudo mkdir -m 0700 /var/lib/waagent
        # sudo mv /lib/udev/rules.d/75-persistent-net-generator.rules /var/lib/waagent/
        # sudo mv /etc/udev/rules.d/70-persistent-net.rules /var/lib/waagent/

5.	Ensure the network service will start at boot time by running the following command:

        # sudo chkconfig network on

6.	Register your Red Hat subscription to enable installation of packages from the RHEL repository by running the following command:

        # sudo subscription-manager register --auto-attach --username=XXX --password=XXX

7.	The WALinuxAgent package `WALinuxAgent-<version>` has been pushed to the Fedora EPEL 6 repository. Enable the epel repository by running the following command::

        # wget http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
        # rpm -ivh epel-release-6-8.noarch.rpm

8.	Modify the kernel boot line in your grub configuration to include additional kernel parameters for Azure. To do this open "/boot/grub/menu.lst" in a text editor and ensure that the default kernel includes the following parameters:

        console=ttyS0 
        earlyprintk=ttyS0 
        rootdelay=300 
        numa=off

    This will also ensure all console messages are sent to the first serial port, which can assist Azure support with debugging issues. This will disable NUMA due to a bug in the kernel version used by RHEL 6.
    In addition to the above, it is recommended to remove the following parameters:

        rhgb quiet crashkernel=auto

    Graphical and quiet boot are not useful in a cloud environment where we want all the logs to be sent to the serial port.
    The crashkernel option may be left configured if desired, but note that this parameter will reduce the amount of available memory in the VM by 128MB or more, which may be problematic on the smaller VM sizes.

9.	Ensure that the SSH server is installed and configured to start at boot time. This is usually the default. Modify the `/etc/ssh/sshd_config` to include following line:

        ClientAliveInterval 180

10.	Install the Azure Linux Agent by running the following command:

        # sudo yum install WALinuxAgent
        # sudo chkconfig waagent on

11.	Do not create swap space on the OS disk:
    
    The Azure Linux Agent can automatically configure swap space using the local resource disk that is attached to the VM after provisioning on Azure. Note that the local resource disk is a temporary disk, and might be emptied when the VM is deprovisioned. After installing the Azure Linux Agent (see previous step), modify the following parameters in `/etc/waagent.conf` appropriately:

        ResourceDisk.Format=y
        ResourceDisk.Filesystem=ext4
        ResourceDisk.MountPoint=/mnt/resource
        ResourceDisk.EnableSwap=y
        ResourceDisk.SwapSizeMB=2048    ## NOTE: set this to whatever you need it to be.

12.	Unregister the subscription (if necessary) by running the following command:

        # sudo subscription-manager unregister

13.	Run the following commands to deprovision the virtual machine and prepare it for provisioning on Azure:

        # sudo waagent -force -deprovision 
        # export HISTSIZE=0
        # logout

14.	Shut down the VM, and convert the VMDK file to VHD file.

    First convert the image to raw format:

        # qemu-img convert -f vmdk –O raw rhel-6.7.vmdk rhel-6.7.raw

    Make sure the size of raw image is aligned with 1MB, otherwise round up the size to align with 1MB:

        # MB=$((1024*1024))
        # size=$(qemu-img info -f raw --output json "rhel-6.7.raw" | \
                gawk 'match($0, /"virtual-size": ([0-9]+),/, val) {print val[1]}')
        # rounded_size=$((($size/$MB + 1)*$MB))
        # qemu-img resize rhel-6.7.raw $rounded_size

    Convert the raw disk to fixed-sized vhd:

        # qemu-img convert -f raw -o subformat=fixed -O vpc rhel-6.7.raw rhel-6.7.vhd

###RHEL 7.1/7.2

1.	Create a file named **network** in the /etc/sysconfig/ directory that contains the following text:

        NETWORKING=yes
        HOSTNAME=localhost.localdomain

2.	Create a file named **ifcfg-eth0** in the /etc/sysconfig/network-scripts/ directory that contains the following text:

        DEVICE=eth0
        ONBOOT=yes
        BOOTPROTO=dhcp
        TYPE=Ethernet
        USERCTL=no
        PEERDNS=yes
        IPV6INIT=no

3.	Ensure the network service will start at boot time by running the following command:

        # sudo chkconfig network on

4.	Register your Red Hat subscription to enable installation of packages from the RHEL repository by running the following command:

        # sudo subscription-manager register --auto-attach --username=XXX --password=XXX

5.	Modify the kernel boot line in your grub configuration to include additional kernel parameters for Azure. To do this open `/etc/default/grub` in a text editor and edit the **GRUB_CMDLINE_LINUX** parameter, for example:

        GRUB_CMDLINE_LINUX="rootdelay=300 
        console=ttyS0 
        earlyprintk=ttyS0"

    This will also ensure all console messages are sent to the first serial port, which can assist Azure support with debugging issues. In addition to the above, it is recommended to remove the following parameters:

        rhgb quiet crashkernel=auto

    Graphical and quiet boot are not useful in a cloud environment where we want all the logs to be sent to the serial port.
    The crashkernel option may be left configured if desired, but note that this parameter will reduce the amount of available memory in the VM by 128MB or more, which may be problematic on the smaller VM sizes.

6.	Once you are done editing `/etc/default/grub` per above, run the following command to rebuild the grub configuration:

         # sudo grub2-mkconfig -o /boot/grub2/grub.cfg

7.	Add Hyper-V modules into initramfs: 

    Edit `/etc/dracut.conf`, add content:

        add_drivers+=”hv_vmbus hv_netvsc hv_storvsc”

    Rebuild the initramfs:

        # dracut –f -v

8.	Ensure that the SSH server is installed and configured to start at boot time. This is usually the default. Modify the `/etc/ssh/sshd_config` to include following line:

        ClientAliveInterval 180

9.	The WALinuxAgent package `WALinuxAgent-<version>` has been pushed to the Fedora EPEL 6 repository. Enable the epel repository by running the following command:


        # wget http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
        # rpm -ivh epel-release-7-5.noarch.rpm

10.	Install the Azure Linux Agent by running the following command:

        # sudo yum install WALinuxAgent
        # sudo systemctl enable waagent.service

11.	Do not create swap space on the OS disk. The Azure Linux Agent can automatically configure swap space using the local resource disk that is attached to the VM after provisioning on Azure. Note that the local resource disk is a temporary disk, and might be emptied when the VM is deprovisioned. After installing the Azure Linux Agent (see previous step), modify the following parameters in `/etc/waagent.conf` appropriately:

        ResourceDisk.Format=y
        ResourceDisk.Filesystem=ext4
        ResourceDisk.MountPoint=/mnt/resource
        ResourceDisk.EnableSwap=y
        ResourceDisk.SwapSizeMB=2048    ## NOTE: set this to whatever you need it to be.

12.	If you want to unregister the subscription, run the following command:

        # sudo subscription-manager unregister

13.	Run the following commands to deprovision the virtual machine and prepare it for provisioning on Azure:

        # sudo waagent -force -deprovision
        # export HISTSIZE=0
        # logout

14.	Shut down the VM, and convert the VMDK file to VHD format.

    First convert the image to raw format:

        # qemu-img convert -f vmdk –O raw rhel-7.1.vmdk rhel-7.1.raw

    Make sure the size of raw image is aligned with 1MB, otherwise round up the size to align with 1MB:

        # MB=$((1024*1024))
        # size=$(qemu-img info -f raw --output json "rhel-7.1.raw" | \
                 gawk 'match($0, /"virtual-size": ([0-9]+),/, val) {print val[1]}')
        # rounded_size=$((($size/$MB + 1)*$MB))
        # qemu-img resize rhel-7.1.raw $rounded_size

    Convert the raw disk to fixed-sized vhd:

        # qemu-img convert -f raw -o subformat=fixed -O vpc rhel-7.1.raw rhel-7.1.vhd


##Prepare from an ISO using kickstart file automatically
###RHEL 7.1/7.2

1.	Create the kickstart file with below content, and save the file. For details about kickstart installation, please refer to the [Kickstart Installation Guide](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Installation_Guide/chap-kickstart-installations.html).


        # Kickstart for provisioning a RHEL 7 Azure VM

        # System authorization information
        auth --enableshadow --passalgo=sha512

        # Use graphical install
        text

        # Do not run the Setup Agent on first boot
        firstboot --disable

        # Keyboard layouts
        keyboard --vckeymap=us --xlayouts='us'

        # System language
        lang en_US.UTF-8

        # Network information
        network  --bootproto=dhcp

        # Root password
        rootpw --plaintext "to_be_disabled"

        # System services
        services --enabled="sshd,waagent,NetworkManager"

        # System timezone
        timezone Etc/UTC --isUtc --ntpservers 0.rhel.pool.ntp.org,1.rhel.pool.ntp.org,2.rhel.pool.ntp.org,3.rhel.pool.ntp.org

        # Partition clearing information
        clearpart --all --initlabel

        # Clear the MBR
        zerombr

        # Disk partitioning information
        part /boot --fstype="xfs" --size=500
        part / --fstyp="xfs" --size=1 --grow --asprimary

        # System bootloader configuration
        bootloader --location=mbr

        # Firewall configuration
        firewall --disabled

        # Enable SELinux
        selinux --enforcing

        # Don't configure X
        skipx

        # Power down the machine after install
        poweroff

        # Primary Fedora repo
        repo --name="epel7" --baseurl="http://dl.fedoraproject.org/pub/epel/7/x86_64/"

        %packages
        @base
        @console-internet
        chrony
        sudo
        python-pyasn1
        parted
        ntfsprogs
        WALinuxAgent
        -dracut-config-rescue

        %end

        %post --log=/var/log/anaconda/post-install.log

        #!/bin/bash

        # Disable the root account
        usermod root -p '!!'

        # Configure swap in WALinuxAgent
        sed -i 's/^\(ResourceDisk\.EnableSwap\)=[Nn]$/\1=y/g' /etc/waagent.conf
        sed -i 's/^\(ResourceDisk\.SwapSizeMB\)=[0-9]*$/\1=2048/g' /etc/waagent.conf

        # Set the cmdline
        sed -i 's/^\(GRUB_CMDLINE_LINUX\)=".*"$/\1="console=tty1 console=ttyS0 earlyprintk=ttyS0 rootdelay=300"/g' /etc/default/grub

        # Enable SSH keepalive
        sed -i 's/^#\(ClientAliveInterval\).*$/\1 180/g' /etc/ssh/sshd_config

        # Build the grub cfg
        grub2-mkconfig -o /boot/grub2/grub.cfg

        # Configure network
        cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth0
        DEVICE=eth0
        ONBOOT=yes
        BOOTPROTO=dhcp
        TYPE=Ethernet
        USERCTL=no
        PEERDNS=yes
        IPV6INIT=no
        NM_CONTROLLED=yes
        EOF

        # Deprovision and prepare for Azure
        waagent -force -deprovision

        %end

 

2.	Place the kickstart file to a place reachable from the installation system. 
 
3.	In Hyper-V Manager, create a new VM. In the **Connect Virtual Hard Disk** page, select **Attach a virtual hard disk later**, and complete the New Virtual Machine Wizard.

4.	Open the VM settings:

    a.	Attach a new virtual hard disk to the VM, make sure to select **VHD Format** and **Fixed Size**.
    
    b.	Attach the installation ISO to DVD drive.

    c.	Set BIOS to boot from CD.

5.	Start the VM, when the installation guide appears, press **Tab** to configure boot options.

6.	Enter `inst.ks=<the location of the Kickstart file>` at the end of the boot options, and press **Enter**.

7.	Wait for the installation to finish, when it’s finished, the VM will be shutdown automatically. Your Linux VHD is now ready to be uploaded to Azure.

##Known issues
There are known issues when you are using RHEL 7.1 in Hyper-V and Azure.

###Issue: Disk I/O freeze 

This issue may occur during frequent storage disk I/O acitivities with RHEL 7.1 in Hyper-V and Azure.   

Repro Rate:

This issue is intermittent, however, it occurs more freqently during frequent disk I/O operations in Hyper-V and Azure.   

    
[AZURE.NOTE] This known issue has already been addressed by Red Hat. To install the associated fixes, run the following command:

    # sudo yum update


## Next Steps
You're now ready to use your Red Hat Enterprise Linux .vhd  to create new Azure Virtual Machines in Azure. For more details about the hypervisors that are certified to run Red Hat Enterprise Linux, visit [the Red Hat website](https://access.redhat.com/certified-hypervisors).
