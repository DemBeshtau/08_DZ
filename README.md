# Работа с NFS #
1. Запустить 2 настроенные виртуальные машины (сервер NFS и клиент) без дополнительных<br/>
действий вручную;<br/>
2. На сервере NFS подготовить и экспортировать директорию;<br/>
3. В экспортированной директории создать поддиректорию upload с правами на запись в неё;<br/>
4. Настроить автоматическое монтирование жкспортированной директории на кленте при старте<br/>
виртуальной машины (systemd, autofs или fstab - любым способом);<br/>
5. Организовать монтирование и работу NFS на клиенте с использованием NFSv3 по протоколу UDP;<br/>
6. Настроить firewall и на сервере и на клиенте.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.14 и образа<br/> 
&ensp;&ensp;CentOS 7 версии 1804_2
### Ход решения ###
#### 1. Настройка сервера NFS ####
1.1. Включение firewall:<br/>
```shell
systemctl enable firewalld --now
systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2024-04-22 17:31:28 UTC; 23min ago
     Docs: man:firewalld(1)
 Main PID: 577 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─577 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid

Apr 22 17:31:27 nfss systemd[1]: Starting firewalld - dynamic firewall daemon...
Apr 22 17:31:28 nfss systemd[1]: Started firewalld - dynamic firewall daemon.
Hint: Some lines were ellipsized, use -l to show in full.
```
1.2. Включение серевера NFS:<br/>
```shell
systemctl enable nfs --now
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/generator/nfs-server.service.d
           └─order-with-mounts.conf
   Active: active (exited) since Mon 2024-04-22 17:31:31 UTC; 25min ago
  Process: 997 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
  Process: 989 ExecStartPre=/bin/sh -c /bin/kill -HUP `cat /run/gssproxy.pid` (code=exited, status=0/SUCCESS)
  Process: 986 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 997 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service

Apr 22 17:31:31 nfss systemd[1]: Starting NFS server and services...
Apr 22 17:31:31 nfss systemd[1]: Started NFS server and services.
```
1.3. Добавление разрешений к сервисам nfs3, rpc-bind, mountd в firewall:<br>
```shell
firewall-cmd --add-service="nfs3" --add-service="rpc-bind" --add-service="mountd" --permanent
firewall-cmd --reload
```
1.4. Проверка наличия прослушиваемых портов 2049/udp, 2049/tcp, 20048/udp, 20048/tcp, 111/udp, 111/tcp:<br/>
```shell
ss -ntlpu | grep *:2049 && ss -ntlpu | grep *:20048 && ss -ntlpu | grep *:111
udp    UNCONN     0      0         *:2049                  *:*                  
tcp    LISTEN     0      64        *:2049                  *:*                  
udp    UNCONN     0      0         *:20048                 *:*                   users:(("rpc.mountd",pid=985,fd=7))
tcp    LISTEN     0      128       *:20048                 *:*                   users:(("rpc.mountd",pid=985,fd=8))
udp    UNCONN     0      0         *:111                   *:*                   users:(("rpcbind",pid=553,fd=6))
tcp    LISTEN     0      128       *:111                   *:*                   users:(("rpcbind",pid=553,fd=8))
```
1.5. Создание и настройка директории для экспортирования:<br/>
```shell
mkdir -p /srv/share/upload
chown -R nfsnobody:nfsnobody /srv/share
chmod 0777 /srv/share/upload
```
1.6. Внесение в файл /etc/exports данных об экспортируемой директории:<br/>
```shell
echo "/srv/share 192.168.50.11/32(rw,sync,root_squash)" >> /etc/exports
cat /etc/exports
/srv/share 192.168.50.11/32(rw,sync,root_squash)
```
1.7. Экспорт директории и её проверка:<br/>
```shell
exportfs -r
exportfs -s
/srv/share  192.168.50.11/32(rw,sync,wdelay,hide,no_subtree_check,sec=sys,secure,root_squash,no_all_squash)
```
1.8. Создание файла в сетевой директории upload:
```shell
cd /srv/share/upload/
touch check_file
ll
-rw-r--r--. 1 root root 0 Apr 22 18:30 check_file
```
#### 2. Настройка клиента NFS ####
2.1. Включение firewall:<br/>
```shell
systemctl enable firewalld --now
systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2024-04-22 18:36:06 UTC; 12s ago
     Docs: man:firewalld(1)
 Main PID: 2941 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─2941 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid

Apr 22 18:36:06 nfsc systemd[1]: Starting firewalld - dynamic firewall daemon...
Apr 22 18:36:06 nfsc systemd[1]: Started firewalld - dynamic firewall daemon.
Hint: Some lines were ellipsized, use -l to show in full.
```
2.2. Добавление автомонтирования сетевой директории /etc/fstab:<br/>
```shell
echo "192.168.50.10:/srv/share/ /mnt nfs vers=3,proto=udp,x-systemd.automount 0 0" >> /etc/fstab
cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults        0 0
192.168.50.10:/srv/share/ /mnt nfs vers=3,proto=udp,x-systemd.automount 0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
```
2.3. Актуализация внесённых изменений и проверка успешности монтирования:<br/>
```shell
systemctl daemon-reload
systemctl restart remote-fs.target
cd /mnt/upload
mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=22,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=11522)
192.168.50.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.50.10)
```
#### 3. Проверка работоспособности реализованной системы ####
3.1. Создание файла в директории /srv/share/upload на сервере NFS:<br/>
```shell
[root@nfss /]# touch /srv/share/upload/check_file
[root@nfss /]# ll /srv/share/upload/check_file
-rw-r--r--. 1 root root 0 Apr 22 18:58 /srv/share/upload/check_file
```
3.2. Проверка наличия файла в сетевой директории на клиенте NFS:<br/>
```shell
[root@nfsc /]# cd /mnt/upload/
[root@nfsc upload]# ll
total 0
-rw-r--r--. 1 root root 0 Apr 22 18:58 check_file
[root@nfsc upload]# 
```
3.3. Перезагрузка клиента и сервера NFS;<br/>
3.4. Проверка сервера:<br/>
```shell
[root@nfss ~]# ll /srv/share/upload      
total 0
-rw-r--r--. 1 root root 0 Apr 22 18:58 check_file
[root@nfss ~]# systemctl status nfs
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/generator/nfs-server.service.d
           └─order-with-mounts.conf
   Active: active (exited) since Mon 2024-04-22 19:06:34 UTC; 3min 0s ago
  Process: 996 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
  Process: 990 ExecStartPre=/bin/sh -c /bin/kill -HUP `cat /run/gssproxy.pid` (code=exited, status=0/SUCCESS)
  Process: 985 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 996 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service

Apr 22 19:06:34 nfss systemd[1]: Starting NFS server and services...
Apr 22 19:06:34 nfss systemd[1]: Started NFS server and services.
[root@nfss ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2024-04-22 19:06:31 UTC; 3min 19s ago
     Docs: man:firewalld(1)
 Main PID: 581 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─581 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid

Apr 22 19:06:30 nfss systemd[1]: Starting firewalld - dynamic firewall daemon...
Apr 22 19:06:31 nfss systemd[1]: Started firewalld - dynamic firewall daemon.
[root@nfss ~]# showmount -a 192.168.50.10
All mount points on 192.168.50.10:
192.168.50.11:/srv/share
```
3.5. Проверка клиента NFS:<br/>
```shell
[root@nfsc upload]# showmount -a 192.168.50.10
All mount points on 192.168.50.10:
192.168.50.11:/srv/share

```
