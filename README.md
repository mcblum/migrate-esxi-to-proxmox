# migrate-esxi-to-proxmox
I spent a long time trying to figure out how to migrate my ESXi VMs to Proxmox.
Let me preface this guide by saying that you shouldn't need it.
Your servers should be cattle, not pets.
However, we all start somewhere, so if you have some pet servers and you're like "Oh my God I don't even remember _how_ to install this app I wrote it so long ago in Fortran and if I edit a single text file the bank will go offline" then maybe you shouldn't be using Proxmox or reading this at all where was I again?

I used these steps with ESXi 6 and migrated to Proxmox 7.1.
I have no idea if it will work with other versions, but I assume it will.
Proxmox has some official instructions but they didn't work for me due to my older version of ESXi.
Perhaps there are a bunch of things I'm missing here or better ways to do this, and if that's the case, please open a PR!

What you'll need:
1. SSH access to both machines
2. A working knowledge of the Linux command line
3. Faith in the lord glob

# Get the virtual disk
1. Shutdown your VM on ESXi (I'm not 100% sure if this is necessary, but it's what I did)
2. SSH to the ESXi server. If you cannot SSH, you will need to enable SSH in the VSphere client for that host
3. You in? Excellent.
4. Find the VM's disk by doing `cd /vmfs/volumes` and then continue `cd`ing until you see the folder with the name of your VM like `Application Server 01`
5. `cd` inside that folder and find the vmdk file that has the `-flat` in it. Something like `Applicaton Server 01-flat.vmdk`
6. Make note of the complete path which will be something like `/vmfs/volumes/58860994-b9457732-b97a-a0369fb7d064/Application Server 01/Application Server 01-flat.vmdk`
7. Open up a new terminal window and use SCP to copy that vmdk file either to your local machine or directly to your Proxmox host. The command will look like `scp root@ip.of.your.esxi.host:/vmfs/volumes/58860994-b9457732-b97a-a0369fb7d064/Application Server 01/Application Server 01-flat.vmdk ~/Deskop` for your local machine. You could also export an NFS share and mount it to both Proxmox and ESXi, I just did this because I was lazy and wanted to save 5 minutes at the start only for it to cost 2 hours later

# Create the Proxmox VM
1. In the Proxmox UI, create a new VM and use a config that exactly mirrors your ESXi setup. Make sure you:  
  a. Use `raw` for the harddrive format   
  b. Make the hard drive the exact same size as what it was
  c. Use a datastore that _isn't_ `local-lvm` such as `local` or an NFS share. You'll need access to the path in order to make this work
  d. Use the same MAC address as was created for you in ESXi. You can find that in your VM config in the VSphere client or in settings in VMWare Fusion (what I use to quickly browse VMs)
2. Now that your VM is created, go find that hard disk. It will probably be somewhere like `/mnt/pve/nfs-share/images/{120}` where 120 is the ID of the Proxmox VM. If you used local, find that path, I used NFS so I don't know where it is.
3. In that directory you'll see a file like m-120-disk-0.raw. Do the unthinkable and `rm ./m-120-disk-0.raw`

# Upload your virtual disk
**If you SCP'd the virtual disk to your machine**  
Use SCP to upload it to exactly where you just deleted the file, naming it exactly the same as the file you just deleted. For example, `scp ~/Desktop/Application Server 01-flat.vmdk root@ip.of.your.proxmox.host:/mnt/pve/nfs-share/images/{120}/m-120-disk0.raw`. You'll notice we just straight up changed the file extension from `vmdk` to `raw` like a bunch of crazy people

**If you mounted an NFS share**
SSH to the Proxmox host and run `cp /mnt/nfs-share-you-mounted-on-both-machines/Application Server 01-flat.vmdk /mnt/pve/nfs-share/images/{120}/m-120-disk0.raw`, copying the virtual disk and changing its extension from vmdk to raw like the mad-dog programmer you are

# Detach + Re-attach your virtual disk
1. In the Proxmox UI, go to `Hardware` and select the drive. In the top menu bar, click `Detach`.
2. Once detached, select `Attach` and chose `IDE` for the protocol

# Set the boot order
1. In the Proxmox UI, go to `Options` and select `Boot Order`. Enable `IDE` and drag it so that it's first

# Start and reconfigure the VM
1. Start the VM and weap as it boots like the sweet, sweet snowflake that it is
2. Log in to the vm
3. Run `ip link show` and make note of the new interface name (mine is ens18)
4. If you're like me, VMWare used the interface `ens32` so you'll need to update your networking
5. Run `vim /etc/network/interfaces` and change any references to `ens32` to `ens18`
6. Run `ifconfig ens18 up` (or whatever your interface name is)
7. Run `ifconfig` to see that it's up
8. Reboot

# Verify networking and install packages
1. Log in to the VM after a reboot and run `ping google.com`
2. If you're connected, huzzah! 
3. Install the QEMU guest agent with `apt install qemu-guest-agent`
4. I should mention, if you're using CentOS than you're probably smarter than me and some of these steps may need (will need) to be changed I'm sure
5. After the package is installed, shut down the VM

# Detach and reattach drive
1. In the Proxmox UI, navigate to `Hardware`, select the drive and click `Detach`
2. Now click `Attach` and this time choose the `SCSI` option
3. Navigate to `Options` and then `Boot Order`, enabling the `SCSI` option and dragging it to be the first in the list

# Final steps
1. Power on the VM and check to see if it's working
2. If so, congratulations! Now please learn Ansible or Puppet or something so this doesn't happen again
3. If not, open an issue and I'll try to help, though I'm not great at this so I may not be much help

# (Optional) Move hard disk to desired end location
I don't currently have an NFS share that has 10gb networking, so I typically run my VMs from the hosts local SSDs.
After my VMs were working, I used the `Move Disk` option to move the virtual drive to the desired location.
At this point, you can also change the hard drive format (if desired).

I hope this worked for you and that you've now made a wonderful transition to Proxmox!
It's an incredible piece of software and while perhaps it's not suited for the most demanding prod environments, for a homelabber like me it's plenty adaquate!
