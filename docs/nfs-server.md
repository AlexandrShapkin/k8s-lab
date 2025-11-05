ОС: alpine-virt-3.22.1-x86_64
CPU: 1/1
RAM: 768MB
HD: 8GB - sys; 40GB - nfs
Интерфейсы: eht0 (host-only) - 00:A0:A0:A0:A0:10 (172.10.10.10)

- Вход `root` без пароля
- `setub-alpine`
- Раскладка только `us`, другие не понадобятся
- hostname - `nfs0.lab`
- Сетевой интерфейс `eth0`, получение адреса по dhcp, без ручной настройки интерфейса
- Пароль снова `1111`, для простоты
- Временная зона `Asia/Barnaul` - мой регион проживания
- Без прокси
- NTP клиент - `busybox`
- Найти быстрейшее зеркало
- Без создания сервера
- SSH-сервер - `openssh`
- Разрешить подключение `root` пользователя по паролю, без ssh ключа
- Диск установки системы 8GB sda, на всякий случай lvm, в качестве системного диска, понятное дело с очисткой места на диске

- `apk add nfs-utils`

```bash
nfs0:~# fdisk /dev/sdb
Device contains neither a valid DOS partition table, nor Sun, SGI, OSF or GPT disklabel
Building a new DOS disklabel. Changes will remain in memory only,
until you decide to write them. After that the previous content
won't be recoverable.


The number of cylinders for this disk is set to 5221.
There is nothing wrong with that, but this is larger than 1024,
and could in certain setups cause problems with:
1) software that runs at boot time (e.g., old versions of LILO)
2) booting and partitioning software from other OSs
   (e.g., DOS FDISK, OS/2 FDISK)

Command (m for help): m
Command Action
a       toggle a bootable flag
b       edit bsd disklabel
c       toggle the dos compatibility flag
d       delete a partition
l       list known partition types
n       add a new partition
o       create a new empty DOS partition table
p       print the partition table
q       quit without saving changes
s       create a new empty Sun disklabel
t       change a partition's system id
u       change display/entry units
v       verify the partition table
w       write table to disk and exit
x       extra functionality (experts only)

Command (m for help): n
Partition type
   p   primary partition (1-4)
   e   extended
p
Partition number (1-4): 1
First sector (63-83886079, default 63): 
Using default value 63
Last sector or +size{,K,M,G,T} (63-83886079, default 83886079): 
Using default value 83886079

Command (m for help): p
Disk /dev/sdb: 40 GB, 42949672960 bytes, 83886080 sectors
5221 cylinders, 255 heads, 63 sectors/track
Units: sectors of 1 * 512 = 512 bytes

Device  Boot StartCHS    EndCHS        StartLBA     EndLBA    Sectors  Size Id Type
/dev/sdb1    0,1,1       1023,254,63         63   83886079   83886017 39.9G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table
```

- `apk add xfsprogs-extra`
- `mkfs.xfs /dev/sdb1`
- `mkdir /data`
- `blkid | grep sdb1`
> /dev/sdb1: UUID="b7c222ae-97aa-4aee-ac2e-ccc886a8bf82" BLOCK_SIZE="512" TYPE="xfs"
- `echo "UUID=b7c222ae-97aa-4aee-ac2e-ccc886a8bf82    /data xfs rw,relatime 0 0" >> /etc/fstab`
- `modprobe nfsd`
- `modprobe nfs`
- `rc-service rpcbind start`
- `rc-update add rpcbind default`
- `rc-update add nfs`
- `rc-service rpcbind start`
- `rc-update add rpcbind default`
- `rc-service nfs start`
- `echo "/data    172.10.10.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports`
- `exportfs -afv`
- `reboot`
- `exportfs -v`
