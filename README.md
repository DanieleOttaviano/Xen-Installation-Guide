# Xen-Installation-Guide
In this reposity i explain how to test and install Xen on a Virtual Machine for testing and teching purpose.  

## Introduction

Xen is born as **Type-1** or bare-metal hypervisor for server, and it uses the technique of **Para-virtualization** due to which guest operating systems must be aware of being virtualized and must be modified to perform hypercalls to communicate with the underlying hypervisor. One of the main advantages of Xen, especially for the server environment, is the ability to take advantage of both para-virtualization and hardware support of new architectures to allow virtual machines to achieve performance very close to non-virtualized execution. 
Therefore, Xen turns out to be interesting as open-source software easily configurable and usable and characterized by a very small footprint and interface (about 1 Mb occupancy) due to the microkernel design used, which also makes it very reliable and safe. 
A VM that runs on Xen is called **domain**, and the most important domain is the so-called domain 0 (or **Dom0**). The latter is the first guestOS started at boot and it has drivers for all devices in the system. Moreover, Dom0 contains a control stack and other system services to manage a system based on XEN; it has functions to create virtual machines, destroy them, monitor them, etc. All other virtual machines run by XEN are called **DomU** (Unprivileged Domains) and generally do not have direct access to the hardware. Hence, they must rely on the hypervisor mechanisms and Dom0 to access the resources. 
To cope with the demands of the IIoT (Industrial Internet of Things) world of hypervisors able to manage execution environments with strict real-time constraints and mixed criticality issues, the Xen community has developed alternative schedulers for the management of virtual CPUs on physical CPUs and has reduced the latency generated by the management of interruptions.
The first real-time scheduler proposed by XEN is an implementation of the **Real-Time Deferrable Scheduler** (RTDS) to ensure a certain amount of CPU defined a priori to guestOS running on the system.
-	The RTDS algorithm assigns to each vCPU of each domain a budget and a period, such that each VM is supposed to execute its budget, even in a fragmented manner, in each period. The deadline corresponds to the end of each period and the budget is restored at the beginning of each period. Each vCPU must necessarily finish its budget by the end of the period, but in case there is no demand in a period or not enough to consume the entire budget, the unused part of the budget will be discarded at the end of the period. RTDS follows the theory of the Preemptive global Earliest Deadline First (EDF) to schedule the vCPU: At each point in time, the vCPU with the closest deadline will be scheduled. To schedule a vCPU means to run it on a physical CPU, and this happens, according to what is said, when a physical CPU is idle or when there is a lower priority task running on it.

Besides RTDS, Xen offers the possibility to also use the null scheduler: 
-	This scheduler assigns each vCPU to physical ones, ensuring their execution and removing the overhead of scheduling decisions at the cost of the entire use of the physical CPU chosen.

Xen also allows splitting the available CPUs into sets called pools characterized by different schedulers. This allows to run VMs with high throughput required but without real-time constraints on a pool that runs non-real-time scheduling algorithms, while VMs with time constraints run on a CPU pool with real-time scheduling (RTDS or null scheduler).

## References and further information
- What is Xen and how it works? https://wiki.xenproject.org/wiki/Xen_Project_Software_Overview
- Xen virtualization mode PV, HVM, PVHVM, and PVH and differences between them. https://wiki.xenproject.org/wiki/Understanding_the_Virtualization_Spectrum
- Xen Schedulers (Real-Time and non Real-Time). https://wiki.xenproject.org/wiki/Xen_Project_Schedulers#Use_cases_and_Support_Status
-	Installation guide.	https://wiki.xenproject.org/wiki/Xen_Project_Beginners_Guide
-	List of Xen Commands.	https://xenbits.xen.org/docs/unstable/man/xl.1.html
- RTDS scheduler.	https://xenbits.xen.org/docs/4.10-testing/features/sched_rtds.html
- Xen Best Practices. https://wiki.xenproject.org/wiki/Xen_Project_Best_Practices
- How to manage CPU pools? https://wiki.xenproject.org/wiki/Cpupools_Howto

## Index
In this tutorial we see:
- Preparation of the environment (VMware/VirtualBox etc…)
- Installation of Debian-11
- Installation of Xen on Debian
- Network Setup
- GRUB Setup
- LVM Setup for guests
- Creation of a PV Guest from scratch
- Debian PV Guest with xen-tools
- Setup of RTDS on Xen
- CPU pools
- Setup of NULL-SCHEDULER on Xen
- Fixed amount of memory for Xen Project dom0

## Preparation of the environment (VMware/VirtualBox etc…)
If you want to install Xen on a physical machine you can skip this part. I decided to install Xen on a Virtual Machine for teaching and portability purposes. Of course, tests done in virtual environments will not be reliable in terms of execution times due to the unpredictability of the entire software stack on which the hypervisor is running. But this is the easiest way to learn how to configure a bare-metal hypervisor. If you need to test the real delay of Xen solution you must install Xen on a physical machine.
First, to execute a virtual machine inside another virtual machine you must enable the Virtualization extensions. If you don’t know how to do it, you can follow the following guides:

- For VMware Workstation, and VirtualBox: https://www.tactig.com/enable-intel-vt-x-amd-virtualization-pc-vmware-virtualbox/ 
- For VMware Fusion: https://techgenix.com/vmware-fusion-5-enable-vt-xept-inside-a-virtual-machine-288/

Then, you must download and install a Virtual Machines Manager like Virtual Box, VMware or any other. 

-	VirtualBox download: https://www.virtualbox.org/. 
-	VMware download: https://www.vmware.com/it/products/workstation-player/workstation-player-evaluation.html

Once the Virtual Machine Manager is installed you need the most recent Debian ISO image, that you can find here: 

-	http://cdimage.debian.org/debian-cd/current/amd64/iso-cd/

Debian is a simple, stable, and well supported Linux distribution. It has included Xen Project Hypervisor support since Debian 3.1 “Sarge” released in 2005. It uses a simple Apt package management system which is both powerful and simple to use. (e.g., apt-get install <<name_of_the_package>>).
We have to create a new Debian Virtual Machine using the downloaded ISO (The example is done on VirtualBox, but it is similar on other VM Managers). 
Open VirtualBox, click on **New** and compile the widget. You must choice **Linux** as type and **Debian (64bit)** as Version. Then you can assign the quantity of RAM to the virtual machine. I decided to assign 4GB but the more you have the better! Remember that we will deploy several OS on this VM.

<p align="center">
    <img src="/../main/Images/environment1.png" width=50% height=50%>
</p>


Click Create. 
Now you must choice the quantity of disk reserved for the VM. As before, the more the better. I suggest you to not assign less than 40gb.

<p align="center">
<img src="/../main/Images/environment2.png" width=50% height=50%>
</p>

Click Create again. Finally, you can **start** the VM.
Once started the VM, you must choice the start-up disk. You can insert the ISO image we downloaded before. 

<p align="center">
<img src="/../main/Images/environment3.png" width=50% height=50%>
</p>

Click **Choose** and then **Start**.


##Installation of Debian-11
Once the VM is started you should see a menu, choose the default “**Install**” option to begin the installation process. Install the system The Debian installer is very straight forward. Follow the prompts until you reach the **disk partitioning section**.
Choose **advanced/custom**, we are going to configure a few partitions here, one for **Boot** “/boot” another for **RootFS** “/”, one more for **swap** and a final partition to setup as an **LVM** (Logical Volume Manager) volume group for our guest machines.

- First create the “/boot” partition by choosing the disk and hitting enter, make the partition 300MB and format it as ext2, choose /boot as the mountpoint.
- Repeat the process for “/” but of course changing the mountpoint to “/” and making it 15GB or so large. Format it as ext3.
- Create another partition approximately 1.5x the amount of RAM you have in size and elect to have it used as a swap volume (6GB in my case).
- Finally create a partition that consumes the rest of the diskspace and reserve it for LVM

We should now have a layout that looks like this assuming your disk device is /dev/sda :

    sda1 - /boot 200MB
    sda2 - / 15GB
    sda3 – swap 	6GB
    sda4 - reserved for LVM


