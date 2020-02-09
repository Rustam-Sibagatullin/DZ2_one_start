1. выбрать тип RAID. В данном случае RAID-5. Подразумевает 4 диска.
2. изменяем Vagrantfile:

MACHINES = {
  :otuslinux => {
        :box_name => "centos/7",
        :ip_addr => '192.168.11.101',
	:disks => {
		:sata1 => {
			:dfile => home + '/VirtualBox VMs/disks/sata1.vdi',
			:size => 250,
			:port => 1
		},
		:sata2 => {
            :dfile => home + '/VirtualBox VMs/disks/sata2.vdi',
            :size => 250, # Megabytes
			:port => 2
		},
        :sata3 => {
            :dfile => home + '/VirtualBox VMs/disks/sata3.vdi',
            :size => 250,
            :port => 3
        },
        :sata4 => {
            :dfile => home + '/VirtualBox VMs/disks/sata4.vdi',
            :size => 250, # Megabytes
            :port => 4
        },
	}
  },
}


должно быть диска.

3. запускаем: vagrant up
4. проверяем на хостовой машине созданные диски:
s -l ~/VirtualBox\ VMs/disks/
итого 1032208
-rw------- 1 rustam3 rustam3 264241152 фев  8 19:59 sata1.vdi
-rw------- 1 rustam3 rustam3 264241152 фев  8 19:59 sata2.vdi
-rw------- 1 rustam3 rustam3 264241152 фев  8 19:59 sata3.vdi
-rw------- 1 rustam3 rustam3 264241152 фев  8 19:59 sata4.vdi

6. подлючамемся : vagrant ssh
7. проверяем существующую конфигурацию:

[vagrant@otuslinux ~]$ cat /proc/mdstat 
Personalities : 
unused devices: <none>
[vagrant@otuslinux ~]$ sudo lshw -short | grep disk
/0/100/1.1/0.0.0    /dev/sda   disk        42GB VBOX HARDDISK
/0/100/d/0          /dev/sdb   disk        262MB VBOX HARDDISK
/0/100/d/1          /dev/sdc   disk        262MB VBOX HARDDISK
/0/100/d/2          /dev/sdd   disk        262MB VBOX HARDDISK
/0/100/d/3          /dev/sde   disk        262MB VBOX HARDDISK


cat /proc/mdstat показывает что нет зеркалирования дисков.

8. создаем массивы:

[vagrant@otuslinux ~]$ sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e}
mdadm: Unrecognised md component device - /dev/sdb
mdadm: Unrecognised md component device - /dev/sdc
mdadm: Unrecognised md component device - /dev/sdd
mdadm: Unrecognised md component device - /dev/sde
[vagrant@otuslinux ~]$ sudo mdadm --create --verbose /dev/md0 -l 5 -n 4 /dev/sd{b,c,d,e}
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.

9. проверка:

[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[4] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
      
unused devices: <none>


10. созадаем mdadm.conf 

echo "DEVICE partitions" | sudo tee -a /etc/mdadm/mdadm.conf
sudo mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' | sudo tee -a /etc/mdadm/mdadm.conf


11. имитация замены сломанного диска:

11.1. проверка до, все должно быть [UUUU]

vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[4] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]

[vagrant@otuslinux ~]$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sat Feb  8 15:03:07 2020
        Raid Level : raid5
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Sat Feb  8 15:03:11 2020
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : af2f34f7:e7bb0c71:1bc032da:90174c8c
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       4       8       64        3      active sync   /dev/sde


11.2. перевод в аварийное состояние:

vagrant@otuslinux ~]$ sudo mdadm /dev/md0 --fail /dev/sde
mdadm: set /dev/sde faulty in /dev/md0

11.3. проверка статуса (нет одного диска [UUU_]):

[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[4](F) sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/3] [UUU_]
      
unused devices: <none>


[vagrant@otuslinux ~]$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sat Feb  8 15:03:07 2020
        Raid Level : raid5
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Sat Feb  8 16:00:31 2020
             State : clean, degraded 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 1
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : af2f34f7:e7bb0c71:1bc032da:90174c8c
            Events : 20

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       -       0        0        3      removed

       4       8       64        -      faulty   /dev/sde




11.4. удаляем диск для замены:

[vagrant@otuslinux ~]$ sudo mdadm /dev/md0 --remove /dev/sde


vagrant@otuslinux ~]$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sat Feb  8 15:03:07 2020
        Raid Level : raid5
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Sat Feb  8 16:01:12 2020
             State : clean, degraded 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : af2f34f7:e7bb0c71:1bc032da:90174c8c
            Events : 21

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       -       0        0        3      removed


11.5. проводим замену и запускаем:

[vagrant@otuslinux ~]$ sudo mdadm /dev/md0 --add /dev/sde
mdadm: added /dev/sde


11.6. проверяем статус (должно быть [UUUU]):

[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[4] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
      
unused devices: <none>


[vagrant@otuslinux ~]$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sat Feb  8 15:03:07 2020
        Raid Level : raid5
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Sat Feb  8 16:01:56 2020
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : af2f34f7:e7bb0c71:1bc032da:90174c8c
            Events : 40

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       4       8       64        3      active sync   /dev/sde



12. Создание GPT разделов, пяти партиций и монтирование их на диск

[vagrant@otuslinux ~]$ sudo parted -s /dev/md0 mklabel gpt
[vagrant@otuslinux ~]$ sudo parted /dev/md0 mkpart primary ext4 0% 20%
[vagrant@otuslinux ~]$ sudo parted /dev/md0 mkpart primary ext4 20% 40%
[vagrant@otuslinux ~]$ sudo parted /dev/md0 mkpart primary ext4 40% 60%
[vagrant@otuslinux ~]$ sudo parted /dev/md0 mkpart primary ext4 60% 80%
[vagrant@otuslinux ~]$ sudo parted /dev/md0 mkpart primary ext4 80% 100% 

[vagrant@otuslinux ~]$ for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
[vagrant@otuslinux ~]$ su -
Password: 
[root@otuslinux ~]# for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done


















