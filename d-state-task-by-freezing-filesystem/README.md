

# Reproduce a D-State Task by Freezing a Filesystem

## ⚠️ Disclaimer – Read Before Use ⚠️

**WARNING: This project may contain scripts, modules, and configurations designed to deliberately break or destabilize systems.**

This reproducer is strictly for testing, debugging, validation, and data gathering practice in controlled lab environments. Running it on production systems can cause data loss, service disruption, system hangs, kernel panics, or complete system failure.

🛑 **DO NOT run anything from this project on systems you do not own, administer, or have explicit permission to test.** 🛑

🛑 **By continuing, you acknowledge these risks and accept full responsibility for any resulting impact.** 🛑

## 🎯 Goal

Reproduce a task stuck (blocked) in uninterruptible sleep state, also known as `D-state`, by freezing a dedicated filesystem and then attempting to access a file inside it.

## 💡 Why

This is useful for learning how blocked I/O can create `D-state` tasks, which is commonly relevant when investigating storage, filesystem, or hung task scenarios.

## 🖥️ Environment  
 
- KVM Host 
- Rocky Linux 10 Guest VM
	- `6.12.0-124.56.1.el10_1.x86_64` (reproducer may also work on most older `el*` versions)
	- 2 Virtual CPUs (in order to expedite high load examples)
- Dedicated additional guest VM disk 
-  `collectl.service` Active/Enabled ( `# dnf install perl-English` then install [Collectl](https://github.com/sharkcz/collectl) ) 
- `sysstat.service` Active/Enabled
- `kdump.service` Active/Enabled and manually pre-tested (in order to manually collect a vmcore)



## 🛠️ Setup

For this reproducer, we are creating a separate virtual disk instead of using the guest VM's existing `/` or `/boot` filesystems.

This keeps the test isolated from the OS disk. Later, we will intentionally freeze the test filesystem to force a task into `D-state`. 

Using a dedicated disk and mount point, such as `/dstate-test`, gives us a safer lab target. If the reproducer causes filesystem access problems, or requires cleanup, the impact is limited to the test filesystem rather than the core OS filesystems.

### From the Rocky Linux guest VM (system info)

```
[/root] # head -1 /etc/os-release ; uname -r ; dmidecode | grep -i 'system info' -A1 
NAME="Rocky Linux"
6.12.0-124.56.1.el10_1.x86_64
System Information
	Manufacturer: QEMU


[/root] # lscpu  | grep '^CPU('
CPU(s):                                  2


[/root] # df ; printf "\n\n" ; lsblk
Filesystem          1K-blocks    Used Available Use% Mounted on
/dev/mapper/rl-root  17756160 5957404  11798756  34% /
devtmpfs              3805072       0   3805072   0% /dev
tmpfs                 3834288       0   3834288   0% /dev/shm
tmpfs                 1533716    8876   1524840   1% /run
tmpfs                    1024       0      1024   0% /run/credentials/systemd-journald.service
/dev/vda2              983040  364928    618112  38% /boot
tmpfs                    1024       0      1024   0% /run/credentials/getty@tty1.service
tmpfs                  766856       4    766852   1% /run/user/0


NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0          11:0    1 1024M  0 rom  
vda         252:0    0   20G  0 disk 
├─vda1      252:1    0    1M  0 part 
├─vda2      252:2    0    1G  0 part /boot
└─vda3      252:3    0   19G  0 part 
  ├─rl-root 253:0    0   17G  0 lvm  /
  └─rl-swap 253:1    0    2G  0 lvm  [SWAP]

```



### From the KVM host:

From the KVM host, create an additional qcow2 disk image we will later attach to the guest VM:


```
[KVM-HOST - ~]$ virsh list --all | grep rocky10
 3    git-hub-rocky10           running


[KVM-HOST - ~]$ sudo qemu-img create -f qcow2 /var/lib/libvirt/images/rocky10-dstate-test-vdb.qcow2 5G
Formatting '/var/lib/libvirt/images/rocky10-dstate-test-vdb.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=5368709120 lazy_refcounts=off refcount_bits=16


[KVM-HOST - ~]$ sudo ls -lh /var/lib/libvirt/images/rocky10-dstate-test-vdb.qcow2
-rw-r--r--. 1 root root 193K May 30 11:45 /var/lib/libvirt/images/rocky10-dstate-test-vdb.qcow2
```


Attach the new qcow2 disk to the VM (in this case my VM is named: `git-hub-rocky10`):

```
[KVM-HOST - ~]$ sudo virsh attach-disk git-hub-rocky10 \
  /var/lib/libvirt/images/rocky10-dstate-test-vdb.qcow2 \
  vdb \
  --targetbus virtio \
  --driver qemu \
  --subdriver qcow2 \
  --cache none \
  --live \
  --config
Disk attached successfully
```

### Back in the guest VM:

We should now see the new disk (`vdb`) in the guest VM:

```
[/root] # lsblk 
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0          11:0    1 1024M  0 rom  
vda         252:0    0   20G  0 disk 
├─vda1      252:1    0    1M  0 part 
├─vda2      252:2    0    1G  0 part /boot
└─vda3      252:3    0   19G  0 part 
  ├─rl-root 253:0    0   17G  0 lvm  /
  └─rl-swap 253:1    0    2G  0 lvm  [SWAP]
vdb         252:16   0    5G  0 disk               <<<------
```

Next, (still from within the guest VM) we create a partition on  `/dev/vdb`:



```
[/root] # parted /dev/vdb
GNU Parted 3.6
Using /dev/vdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted)


(parted) print                                                            
Error: /dev/vdb: unrecognised disk label
Model: Virtio Block Device (virtblk)                                      
Disk /dev/vdb: 5369MB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags: 


(parted) mklabel gpt                                                      
(parted)                                                                  


(parted) print                                                            
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 5369MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start  End  Size  File system  Name  Flags

                                                                
(parted) mkpart dstate-test xfs 1MiB 100%                                 
(parted)                                                                  


(parted) print                                                            
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 5369MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name         Flags
 1      1049kB  5368MB  5367MB  xfs          dstate-test
                                                             
(parted) 


(parted) quit                                                             
Information: You may need to update /etc/fstab.

[/root] #  
```

The above `parted` utility commands helped us create a new GPT partition table on the dedicated test disk `/dev/vdb`. The disk initially had no recognized disk label, so a GPT label was created with `mklabel gpt`. A new `XFS` type partition named `dstate-test` was then created using the full available disk range from `1MiB` to `100%`. The final `parted` output confirms that `/dev/vdb` now contains one partition, `/dev/vdb1`, with the name `dstate-test` and an approximate size of `5367MB`.

```
[/root] # cat /proc/partitions
major minor  #blocks  name

 252        0   20971520 vda
 252        1       1024 vda1
 252        2    1048576 vda2
 252        3   19919872 vda3
  11        0    1048575 sr0
 253        0   17821696 dm-0
 253        1    2097152 dm-1
 252       16    5242880 vdb
 252       17    5240832 vdb1            <<<------


[/root] # lsblk 
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0          11:0    1 1024M  0 rom  
vda         252:0    0   20G  0 disk 
├─vda1      252:1    0    1M  0 part 
├─vda2      252:2    0    1G  0 part /boot
└─vda3      252:3    0   19G  0 part 
  ├─rl-root 253:0    0   17G  0 lvm  /
  └─rl-swap 253:1    0    2G  0 lvm  [SWAP]
vdb         252:16   0    5G  0 disk 
└─vdb1      252:17   0    5G  0 part            <<<------
```

The `parted` utility does not create the filesystem itself.

Next, we create the `XFS` filesystem on the new partition:



```
[/root] # mkfs.xfs /dev/vdb1
meta-data=/dev/vdb1              isize=512    agcount=4, agsize=327552 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=0  
data     =                       bsize=4096   blocks=1310208, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=0
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
[/root] # 



[/root] # lsblk -f
NAME        FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sr0                                                                                          
vda                                                                                          
├─vda1                                                                                       
├─vda2      xfs                        fee2fd28-0f68-458c-a785-005bc14448d8    603.6M    37% /boot
└─vda3      LVM2_member LVM2 001       mXYVco-wTh6-2aW1-QBRY-WoYe-3XWS-f5UGTq                
  ├─rl-root xfs                        3e18a142-ebc0-4096-9b04-3e36fb03efc0     11.3G    34% /
  └─rl-swap swap        1              fd6adf23-7893-4e65-af82-02194f78c4f9                  [SWAP]
vdb                                                                                          
└─vdb1      xfs                        b63555f0-0377-491a-9154-babdc99cbfea   

```

Next step is to create the mount:

```
[/root] # mkdir /dstate-test
[/root] # 
[/root] # mount /dev/vdb1 /dstate-test
[/root] # 
[/root] # lsblk 
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0          11:0    1 1024M  0 rom  
vda         252:0    0   20G  0 disk 
├─vda1      252:1    0    1M  0 part 
├─vda2      252:2    0    1G  0 part /boot
└─vda3      252:3    0   19G  0 part 
  ├─rl-root 253:0    0   17G  0 lvm  /
  └─rl-swap 253:1    0    2G  0 lvm  [SWAP]
vdb         252:16   0    5G  0 disk 
└─vdb1      252:17   0    5G  0 part /dstate-test       <<<------

```

The next step is optional. I am adding the new dedicated test filesystem to `/etc/fstab` only so the same mount point is available again after a reboot for future similar testing.

Using the filesystem `UUID` is safer than using `/dev/vdb1` because device names can change across future changes and reboots:

```
[/root] # blkid /dev/vdb1
/dev/vdb1: UUID="b63555f0-0377-491a-9154-babdc99cbfea" BLOCK_SIZE="512" TYPE="xfs" PARTLABEL="dstate-test" PARTUUID="e6a0fea3-e8f6-41b3-8a82-ed56d114350e"
```
```
[/root] # echo 'UUID=b63555f0-0377-491a-9154-babdc99cbfea /dstate-test xfs defaults,nofail 0 0' >> /etc/fstab
[/root] # 
```
This makes `/dstate-test` mount automatically after reboot. The `nofail` option is useful for a lab testing disks because the system can still boot if this optional disk is missing.

```
[/root] # tail -n 5 /etc/fstab
#
UUID=3e18a142-ebc0-4096-9b04-3e36fb03efc0 /                       xfs     defaults        0 0
UUID=fee2fd28-0f68-458c-a785-005bc14448d8 /boot                   xfs     defaults        0 0
UUID=fd6adf23-7893-4e65-af82-02194f78c4f9 none                    swap    defaults        0 0
UUID=b63555f0-0377-491a-9154-babdc99cbfea /dstate-test xfs defaults,nofail 0 0
```




We now need to create a test file (`testfile`) on the dedicated `XFS` filesystem. This file will later be used as the target for a filesystem access attempt after the filesystem is intentionally frozen. When the frozen filesystem prevents the metadata update, the accessing task will become blocked in uninterruptible sleep state, also known as `D-state`.

```
[/root] # dd if=/dev/zero of=/dstate-test/testfile bs=100M count=3
3+0 records in
3+0 records out
314572800 bytes (315 MB, 300 MiB) copied, 0.0449762 s, 7.0 GB/s
[/root] # 
[/root] # 
[/root] # ls -1 /dstate-test/testfile
/dstate-test/testfile
```

## 🚀 Reproducer Time

Now we can continue with the reproducer.

Intentionally freeze the dedicated test filesystem so a later file access under `dstate-test` is blocked:

```
[/root] # fsfreeze --freeze /dstate-test
[/root] # 
```

_NOTE: `fsfreeze` temporarily suspends access to a mounted filesystem so a stable image can be captured, such as by backup, snapshot, or storage related tools. In production, filesystem freeze and thaw may be triggered by third party tooling. If that workflow hangs, takes too long, or fails to thaw cleanly, the system may become unavailable and tasks accessing that filesystem may block in `D-state`. For this lab, we intentionally freeze `/dstate-test` to safely reproduce that behavior on our dedicated test filesystem._

Now we will try to update the existing file on the frozen filesystem. Because the filesystem is frozen, the `touch` process should block instead of completing.

```
[/root] # touch /dstate-test/testfile &
[1] 2222
[/root] # 
```

Check PID 2222 and confirm whether that specific `touch` task is in `D-state`:

```
[/root] # ps -p 2222 -o pid,stat,cmd
    PID STAT CMD
   2222 D    touch /dstate-test/testfile
        ^
        ^
```
Next, let's create 20 more blocked `touch` tasks:

```
[/root] # for i in $(seq 1 20); do touch /dstate-test/testfile & done
[2] 2232
[3] 2233
[4] 2234
[5] 2235
[6] 2236
[7] 2237
[8] 2238
[9] 2239
[10] 2240
[11] 2241
[12] 2242
[13] 2243
[14] 2244
[15] 2245
[16] 2246
[17] 2247
[18] 2248
[19] 2249
[20] 2250
[21] 2251
[/root] # 
```
Now, let’s confirm all `touch` tasks are blocked and show whether they are in `D-state`.

```
[/root] # ps -C touch -o pid,stat,cmd
    PID STAT CMD
   2222 D    touch /dstate-test/testfile
   2232 D    touch /dstate-test/testfile
   2233 D    touch /dstate-test/testfile
   2234 D    touch /dstate-test/testfile
   2235 D    touch /dstate-test/testfile
   2236 D    touch /dstate-test/testfile
   2237 D    touch /dstate-test/testfile
   2238 D    touch /dstate-test/testfile
   2239 D    touch /dstate-test/testfile
   2240 D    touch /dstate-test/testfile
   2241 D    touch /dstate-test/testfile
   2242 D    touch /dstate-test/testfile
   2243 D    touch /dstate-test/testfile
   2244 D    touch /dstate-test/testfile
   2245 D    touch /dstate-test/testfile
   2246 D    touch /dstate-test/testfile
   2247 D    touch /dstate-test/testfile
   2248 D    touch /dstate-test/testfile
   2249 D    touch /dstate-test/testfile
   2250 D    touch /dstate-test/testfile
   2251 D    touch /dstate-test/testfile
[/root] # 
```

Next, we temporarily enable `hung_task_panic` so the kernel panics when the `khungtaskd` daemon finds a task blocked longer than the configured timeout. Since kdump is already configured and tested on this system, we expect the intentional kernel panic to generate a vmcore for later analysis.

```
[/root] # sysctl -w kernel.hung_task_panic=1
kernel.hung_task_panic = 1
[/root] #
[/root] #
[/root] #
[/root] # ps -ef | grep '[k]hungtaskd'
root          38       2  0 14:44 ?        00:00:00 [khungtaskd]
[/root] #
```

_NOTE: Enabling `kernel.hung_task_panic=1` does not always immediately panic the system. It changes the action taken by the kernel's `khungtaskd` daemon the next time it detects a task blocked longer than `kernel.hung_task_timeout_secs` (120 seconds by default). Because `khungtaskd` checks periodically, there may be a delay before the panic occurs, even if `D-state` tasks already exist._


The system panicked as expected after `khungtaskd` detected a blocked `touch` task. Then, the kdump service (via the `crashkernel`) helped collect the vmcore data and afterwards recover (reboot) the system. The vmcore was saved under `/var/crash`:

```
[/root] # ls -lh /var/crash/127.0.0.1-2026-05-30-15\:40\:03/
total 112M
-rw-------. 1 root root  79K May 30 15:40 kexec-dmesg.log
-rw-------. 1 root root 112M May 30 15:40 vmcore
-rw-------. 1 root root 152K May 30 15:40 vmcore-dmesg.txt
```


```
[/root] # less /var/crash/127.0.0.1-2026-05-30-15\:40\:03/vmcore-dmesg.txt 
[...]
[...]
[...]
[...]
[...]
[...]
[ 3318.595272] INFO: task touch:2251 blocked for more than 491 seconds.             <<<--------
[ 3318.595434]       Not tainted 6.12.0-124.56.1.el10_1.x86_64 #1
[ 3318.595585] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[ 3318.595737] task:touch           state:D stack:0     pid:2251  tgid:2251  ppid:1976   task_flags:0x400000 flags:0x00000002
[ 3318.595901] Call Trace:
[ 3318.596053]  <TASK>
[ 3318.596201]  __schedule+0x2aa/0x660
[ 3318.596349]  schedule+0x27/0xa0
[ 3318.596504]  percpu_rwsem_wait+0x10f/0x140
[ 3318.596652]  ? __pfx_percpu_rwsem_wake_function+0x10/0x10
[ 3318.596806]  __percpu_down_read+0x6c/0x120
[ 3318.596965]  mnt_want_write+0x8f/0xc0
[ 3318.597127]  vfs_utimes+0x248/0x270
[ 3318.597277]  do_utimes+0x65/0x140
[ 3318.597433]  ? syscall_exit_to_user_mode+0x32/0x190
[ 3318.597617]  __x64_sys_utimensat+0x9f/0xf0
[ 3318.597767]  ? srso_alias_return_thunk+0x5/0xfbef5
[ 3318.597925]  do_syscall_64+0x7d/0x160
[ 3318.598077]  ? srso_alias_return_thunk+0x5/0xfbef5
[ 3318.598225]  ? filp_flush+0x5c/0x70
[ 3318.598374]  ? srso_alias_return_thunk+0x5/0xfbef5
[ 3318.598534]  ? filp_close+0x1d/0x30
[ 3318.598684]  ? srso_alias_return_thunk+0x5/0xfbef5
[ 3318.598841]  ? do_dup2+0xac/0x130
[ 3318.598991]  ? srso_alias_return_thunk+0x5/0xfbef5
[ 3318.599141]  ? syscall_exit_work+0xf3/0x120
[ 3318.599292]  ? srso_alias_return_thunk+0x5/0xfbef5
[ 3318.599450]  ? syscall_exit_to_user_mode+0x32/0x190
[ 3318.599601]  ? srso_alias_return_thunk+0x5/0xfbef5
[ 3318.599752]  ? do_syscall_64+0x89/0x160
[ 3318.599908]  ? srso_alias_return_thunk+0x5/0xfbef5
[ 3318.600063]  ? srso_alias_return_thunk+0x5/0xfbef5
[ 3318.600218]  entry_SYSCALL_64_after_hwframe+0x76/0x7e
[ 3318.600368] RIP: 0033:0x7fc42d458c3e
[ 3318.600529] RSP: 002b:00007ffcf55acfb8 EFLAGS: 00000246 ORIG_RAX: 0000000000000118
[ 3318.600682] RAX: ffffffffffffffda RBX: 00007ffcf55ad56a RCX: 00007fc42d458c3e
[ 3318.600852] RDX: 0000000000000000 RSI: 0000000000000000 RDI: 0000000000000000
[ 3318.601007] RBP: 00007ffcf55ad0d0 R08: 0000000000000000 R09: 0000000000000000
[ 3318.601167] R10: 0000000000000000 R11: 0000000000000246 R12: 0000000000000000
[ 3318.601322] R13: 0000000000000000 R14: 00007fc42d52d248 R15: 00007ffcf55ad208
[ 3318.601489]  </TASK>
[ 3318.601643] Future hung task reports are suppressed, see sysctl kernel.hung_task_warnings

[ 3318.601802] Kernel panic - not syncing: hung_task: blocked tasks                    <<<<<<--------- panic  here

[ 3318.601963] CPU: 0 UID: 0 PID: 38 Comm: khungtaskd Kdump: loaded Not tainted 6.12.0-124.56.1.el10_1.x86_64 #1 PREEMPT(voluntary) 
[ 3318.602122] Hardware name: QEMU Standard PC (Q35 + ICH9, 2009), BIOS 1.17.0-9.fc43 06/10/2025
[ 3318.602281] Call Trace:
[ 3318.602442]  <TASK>
[ 3318.602599]  dump_stack_lvl+0x4e/0x70
[ 3318.602757]  panic+0x113/0x2dd
[ 3318.602919]  check_hung_uninterruptible_tasks.cold+0xc/0x12
[ 3318.603074]  ? __pfx_watchdog+0x10/0x10
[ 3318.603227]  watchdog+0x9d/0xa0
[ 3318.603377]  kthread+0xfa/0x240
[ 3318.603524]  ? __pfx_kthread+0x10/0x10
[ 3318.603667]  ret_from_fork+0x31/0x50
[ 3318.603812]  ? __pfx_kthread+0x10/0x10
[ 3318.603962]  ret_from_fork_asm+0x1a/0x30
[ 3318.604110]  </TASK>
(END)
```


## 🔍 Analyze

The chaos has been created. To inspect the blocked `touch` task, panic, vmcore, and kernel stack traces, head over to my separate `linux-debugging` GitHub project: [linux-debugging | hung-task-panic-from-frozen-filesystem-analysis.md](https://github.com/Swansonite/linux-debugging/blob/main/hung-task-panic-from-frozen-filesystem-analysis.md)
