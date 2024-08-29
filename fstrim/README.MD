# fstrim.timer
The role enables or disables fstrim.timer. This SystemD timer is responsible for periodic launch of fstrim.service, which uses fstrim utility under the hood to perfrom SCSI UNMAP operations. In my case it's utilized to reclaim space on VMFS from thin provisioned disks, see the "Thin disk example" section.<br />
Useful link (RU): https://habr.com/ru/articles/497004/.

## trim/discard support inside a guest OS (in VM)
The following method isn't solid in case of virtual machines (at least not for vSphere VMs), but still is good for physical machines/disks:
```
sudo hdparm -I /dev/sda | grep TRIM
```
However you can use lsblk to check if trim/discard is supported inside a guest OS:
```
sudo lsblk --discard
```
Values of DISC-GRAN and DISC-MAX should be non-zero ("0" means that trim/discard is not supported).<br />
Also you can just try to use fstrim utility manually against a specific mountpoint. In case trim/discard is not supported, you will get the corresponding message:
```
sudo fstrim /
fstrim: /: the discard operation is not supported
```
If trim/discard is not supported inside a guest OS of vSphere VM, then it's probably a virtualization layer problem. E.g. space reclamation isn't enabled on a VMFS or vSAN layer.

## Variables
All the variables present in defaults.
- **fstrim_timer_enabled** - ansible bool (true | false). Enables or disables fstrim.timer (and periodic fstrim runs);
- **fstrim_calendar** - string in SystemD cron format. The more often fstrim runs, the less load storage system encounters at once;
- **fstrim_accuracy** - string, check https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html#AccuracySec=;
- **fstrim_randomize_period** - string, check https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html#RandomizedDelaySec=;
- **fstrim_timer_persistent** - string (not ansible bool!), possible values: "true" and "false". Check https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html#Persistent=.

## Recommendations
```
In order to optimally stretch timer events over a certain range of time, set AccuracySec=1us and RandomizedDelaySec= to some higher value.
```
SystemD timer documentation: https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html.<br />
SystemD timer calendar (cron) examples: https://silentlad.com/systemd-timers-oncalendar-(cron)-format-explained.<br />
With the current default values fstrim runs daily at 20:00:00 (8 P.M.) with a random delay up to 1 hour (to mitigate the storage system load caused by SCSI UNMAP in case there are multiple VMs). In case a VM was powered off during the fstrim runtime period (so the fstrim launch was skipped), then fstrim.service will be launched right after the next boot (due to fstrim_timer_persistent="true").<br />
Fstrim execution usually takes a few seconds. I've tested it with 100GB data (1000 files, each file is ~100MB). However it's from a guest OS perspective. On a storage system side it can take a significant amount of time, which depends on the data amount to trim and the storage system capabilities (performance potential). It's better to launch fstrim before any VM backup process (so the backup system is going to transfer less data), but make sure you have a 30-60 minutes room ahead of backup job start.

## Fstrim status
You can check that fstrim works periodically by checking the fstrim.service journal:
```
sudo journalctl -u fstrim.service
```
Of course, the launch time must be around the time set in **fstrim_calendar** with some delay (fstrim_randomize_period/fstrim_accuracy).<br />
Also you can check backup sizes and data transfer sizes of backup jobs, but these parameters aren't very clear due to an unpredictable "useful" data growth rate.

## Thin disk example
This is just an example of how VM with a thin disk behaves in vSphere, so you can safely skip it.

VM with thin-provisioned disks takes exactly the same space on a vSphere datastore (e.g. VMFS) as the data inside this VM. However, hypervisor isn't aware that some data was removed from the guest OS if the OS (linux) doesn't use SCSI UNMAP operations, so the hypervisor thinks the sectors/blocks are still in use.

Let's say our VM has 100GB thin disk, and some file system (e.g. ext4) resides on it inside the guest OS, but there are no files on this file system. This thin disk takes almost no space on VMFS datastore. Then we create a 10GB file inside the guest OS on this ext4. Now this thin disk takes 10GB on VMFS volume. Now we are going to delete this 10GB file, but even after the deletion, the thin disk still takes 10GB on the VMFS. This is because linux doesn't use SCSI UNMAP (trim/discard) and doesn't inform its virtual disk controller (e.g. PVSCSI) and the hypervisor (via the disk controller), that those sectors are free now, in other words there is no "useful" data on those ext4 sectors. So if you check the free space inside the guest OS (df -hT), the whole file system is free, but the thin disk takes 10GB on VMFS. If we will create and then delete another 10GB file, the ext4 FS will be almost empty from the guest OS perspective, but the thin disk is going to take 20G (first 10GB + second 10GB) on VMFS. So, if the guest OS doesn't use trim/unmap on a regular basis, the disk size will eventually grow to it's configured limit (100GB in my example). If your VMs reside in some IaaS, then you can consider it as a prvider problem, however the disk size also impacts backup jobs windows (amount of time to make a backup) and the backup repository size. So, basically it's a good idea to perform periodic trim or discard.<br />
Take a look at the arcticle (RU, translate it with a browser if needed): https://habr.com/ru/articles/497004/.

Both methods (trim and discard) solve the problem. Discard works more smooth, cause it's more stretched in time: SCSI UNMAP is performed after a short period of time after a file(s) removal. However, it can be hard to set up discard in a huge environment, cause it's configured via file system super-block (e.g. tune2fs), or via mount options in /etc/fstab for every mount point. Just a reminder: "issue_discards" in LVM config (/etc/lvm/lvm.conf) works only with LVM volumes removal/shrink, not with files removal.

Fstrim utility on the other hand works on demand (e.g. sudo fstrim -a, sudo fstrim /somedir). So it's not so smooth as discard, BUT you can use fstrim.timer (this Ansible role is about turning fstrim.timer on/off), which triggers fstrim.service (and thus fstrim utility) to run periodically. Fstrim.service isn't running all the time, it's waiting for the fstrim.timer to trigger it. The main advantage of this solution - it's easy to turn it on and off (in case you will encounter any troubles).
