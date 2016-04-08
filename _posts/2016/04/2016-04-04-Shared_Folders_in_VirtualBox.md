---
title: "Shared Folders in VirtualBox"
categories:
- virtualbox
keywords: virtualbox, shared folders
summary: "VirtualBox Shared Folders are a great but often under-utilized feature. He's how I use it to make my virtual machines even better."
thumb: virtualbox_logo_124x124.png
---

![Feature Image](/images/virtualbox/virtualbox_logo_124x124.png)


I build a lot of [VirtualBox](https://www.virtualbox.org/) virtual machines. Mostly to test data replication solutions from an Oracle database to somewhere using Oracle's GoldenGate, Dbvisit's Replicate, or Dell's SharePlex.

Some of these solutions require a significant amount of scripting to make them easy to set up and repeat. Of course, the scripts need to be accessible from the guest VM but that's not the best place to store them as I'm often destroying and rebuilding VMs  and don't want to lose any of my custom scripts. Enter VirtualBox Shared Folders.

VirtualBox Shared Folders enable you to access files on your VirtualBox host from your VirtualBox guest. The real trick here is it can be done without any type of networking configured. The only requirement is that you must have the Guest Additions installed on your VirtualBox guest, and the VirtualBox guest needs to be one of Windows 2000 (or newer), Linux, or Solaris.

For example, I'm using a Windows host, so I've created a folder called `shared` on my `C:` drive in a folder called `VirtualBox VMs`. This is where I place all my custom scripts I want to access from a VirtualBox guest (either Windows or Linux). This is also a very handy way of getting files on and off the virtual machine. Just place the file in the shared folder directory on the VirtualBox host and it will be visible and available in the shared folder on the guest.

To share this folder with a VirtualBox guest, under the Shared Folders section of the Settings window for the VirtualBox guest, click the `Add a New Share Folder Definition` button.

![Shared Folder Settings](/images/virtualbox/Shared_Folders_Settings.png)

Specify the folder path on the VirtualBox host to the folder and give it a folder name (same as a share name) that will be used by the VirtualBox guest to access it.
Check the `Auto-mount` box so the shared folder will be available when the guest machine (re)starts and users log in.

![Shared Folder Settings](/images/virtualbox/Shared_Folders_Add_Share.png)

If the VirtualBox guest is a Linux machine, when the shared folder is mounted, it will appear as a separate file system under a mount point of `/media/sf_<folder_name>`. So for me, that's `/media/sf_shared`. Access to auto-mounted shared folders under Linux is granted via membership of the `vboxsf` group on the VirtualBox guest (which is created by the VirtualBox Guest Additions Installer). Users have to be a member of this group for read/write access.
 
If the VirtualBox guest is a Windows machine, when the shared folder is mounted, it will be given it's own drive letter (eg; `D:`).

One thing I have discovered is that the shared folder mechanism is not very fast compared to local (VM disk) access. It's great for keeping scripts outside the VM or for transferring files like I've described, but if you're writing large files, or many files like a GoldenGate trail, or a Replicate PLOG files, keep those on the local (VM) disk.

There are more options available when configuring shared folders, I've described the settings I use to solve the problem of preserving custom scripts outside the VM. Please see the [Shared Folder](https://www.virtualbox.org/manual/ch04.html#sharedfolders) section of the VirtualBox Manual for a more complete discussion.

Provided I keep all my customizations outside of the VM, I can reuse the single copy of a script in any number of VMs and safety destroy and rebuild a VM without losing any of my hard work!
