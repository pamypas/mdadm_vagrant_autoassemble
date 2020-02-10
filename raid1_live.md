### Перенос на raid 1 на живую

Виртуалка centos 7 с дефолтной установко на один диск. Присоеденили второй:

```
NAME            MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT UUID
sda               8:0    0   10G  0 disk             
├─sda1            8:1    0    1G  0 part  /boot      a4f79993-1c1b-4bea-888a-1e4362d468f6
└─sda2            8:2    0    8G  0 part             4H7aH5-0JV6-rQar-YHhB-Q0jf-MYGZ-ncIOa9
  └─centos-root 253:0    0    8G  0 lvm   /          185da181-06e7-481b-af44-591809d7a6c4
sdb               8:16   0   10G  0 disk             
├─sdb1            8:17   0    1G  0 part             04cd2b43-207b-4b9e-1820-8bd252fd6e76
│ └─md0           9:0    0 1022M  0 raid1            34a66119-9afc-4568-ad5f-6736a11acd40
└─sdb2            8:18   0    8G  0 part             0e6cb076-7e39-e123-44dc-b10a47cec4d5
  └─md1           9:1    0    8G  0 raid1            D3Dxr1-dHSv-j7Vm-hwz3-0Vj2-u9xF-WEVfsQ
```

сделали ему 

```
sfdisk -d /dev/sda | sfdisk -f /dev/sdb

fdisk /dev/sdb
t
1
fd
t
2
fd
w
```

Собираем рейд с пропуском первого диска

```
mdadm --create /dev/md0 --level=1 --raid-disks=2 missing /dev/sdb1
mdadm --create /dev/md1 --level=1 --raid-disks=2 missing /dev/sdb2
mdadm --detail --scan > /etc/mdadm.conf
```

Переносим /boot, не забыв в /etc/fstab прописать ему uuid /dev/md0

```
mkfs.ext2 /dev/md0
mount /dev/md0 /mnt
cp -R /boot /mnt
umount /mnt
umount /boot
mount /dev/md0 /boot
dracut --mdadmconf --force /boot/initramfs-$(uname –r).img $(uname –r)
```

Давайте начнем переносить корневой раздел и собирать рейд

```
pvcreate /dev/md1
vgextend centos /dev/md1
pvmove /dev/sda2 /dev/md1
vgreduce centos /dev/sda2
pvremove /dev/sda2
fdisk /dev/sda
t
1
fd
t
2
fd
w
mdamd --add /dev/md0 /dev/sda1
mdamd --add /dev/md1 /dev/sda2
```

Теперь поправим /etc/default/grub.cfg
568... - UUID /dev/sd[ab]2
04c... - UUID /dev/sd[ab]1

```
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.md.uuid=568dec3a:85aad6cf:912a6c34:99b0f362 rd.md.uuid=04cd2b43:207b4b9e:18208bd2:52fd6e76 dhgb quiet"
```

Обновим и поставим grub на диски

```
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-install /dev/sda
grub2-install /dev/sdb
```

Ребутаем и видим:

```
NAME              MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT UUID
sda                 8:0    0   10G  0 disk             
├─sda1              8:1    0    1G  0 part             04cd2b43-207b-4b9e-1820-8bd252fd6e76
│ └─md0             9:0    0 1022M  0 raid1 /boot      34a66119-9afc-4568-ad5f-6736a11acd40
└─sda2              8:2    0    8G  0 part             568dec3a-85aa-d6cf-912a-6c3499b0f362
  └─md1             9:1    0    8G  0 raid1            Pv7zqe-b090-M8C7-7VDf-EQ9K-dJkE-5YAcXN
    └─centos-root 253:0    0    8G  0 lvm   /          185da181-06e7-481b-af44-591809d7a6c4
sdb                 8:16   0   10G  0 disk             
├─sdb1              8:17   0    1G  0 part             04cd2b43-207b-4b9e-1820-8bd252fd6e76
│ └─md0             9:0    0 1022M  0 raid1 /boot      34a66119-9afc-4568-ad5f-6736a11acd40
└─sdb2              8:18   0    8G  0 part             568dec3a-85aa-d6cf-912a-6c3499b0f362
  └─md1             9:1    0    8G  0 raid1            Pv7zqe-b090-M8C7-7VDf-EQ9K-dJkE-5YAcXN
    └─centos-root 253:0    0    8G  0 lvm   /          185da181-06e7-481b-af44-591809d7a6c4
```

Ради интереса можно оторвать диск (первый, например) во время ребута

### Литература

Полезный материал по centos 6 - https://cmgd.net/archive/Setup_RAID1_CentOS6.pdf
