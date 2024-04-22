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
1.2. Добавление разрешений к сервисам nfs3, rpc-bind, mountd в firewall:<br>
```shell
firewall-cmd --add-service="nfs3" --add-service="rpc-bind" --add-service="mountd" --permanent
firewall-cmd --reload
```
1.3. Проверка наличия прослушиваемых портов 2049/udp, 2049/tcp, 20048/udp, 20048/tcp, 111/udp, 111/tcp:<br/>
```shell
ss -ntlpu
etid State      Recv-Q Send-Q                                         Local Address:Port                                                        Peer Address:Port              
udp   UNCONN     0      0                                                          *:2049                                                                   *:*                  
udp   UNCONN     0      0                                                          *:40198                                                                  *:*                  
udp   UNCONN     0      0                                                  127.0.0.1:323                                                                    *:*                   users:(("chronyd",pid=557,fd=1))
udp   UNCONN     0      0                                                          *:68                                                                     *:*                   users:(("dhclient",pid=2199,fd=6))
udp   UNCONN     0      0                                                          *:20048                                                                  *:*                   users:(("rpc.mountd",pid=985,fd=7))
udp   UNCONN     0      0                                                          *:58989                                                                  *:*                   users:(("rpc.statd",pid=975,fd=8))
udp   UNCONN     0      0                                                          *:111                                                                    *:*                   users:(("rpcbind",pid=553,fd=6))
udp   UNCONN     0      0                                                          *:720                                                                    *:*                   users:(("rpcbind",pid=553,fd=7))
udp   UNCONN     0      0                                                  127.0.0.1:727                                                                    *:*                   users:(("rpc.statd",pid=975,fd=5))
udp   UNCONN     0      0                                                         :::2049                                                                  :::*                  
udp   UNCONN     0      0                                                        ::1:323                                                                   :::*                   users:(("chronyd",pid=557,fd=2))
udp   UNCONN     0      0                                                         :::20048                                                                 :::*                   users:(("rpc.mountd",pid=985,fd=9))
udp   UNCONN     0      0                                                         :::56401                                                                 :::*                   users:(("rpc.statd",pid=975,fd=10))
udp   UNCONN     0      0                                                         :::111                                                                   :::*                   users:(("rpcbind",pid=553,fd=9))
udp   UNCONN     0      0                                                         :::55184                                                                 :::*                  
udp   UNCONN     0      0                                                         :::720                                                                   :::*                   users:(("rpcbind",pid=553,fd=10))
tcp   LISTEN     0      64                                                         *:2049                                                                   *:*                  
tcp   LISTEN     0      64                                                         *:34601                                                                  *:*                  
tcp   LISTEN     0      128                                                        *:111                                                                    *:*                   users:(("rpcbind",pid=553,fd=8))
tcp   LISTEN     0      128                                                        *:20048                                                                  *:*                   users:(("rpc.mountd",pid=985,fd=8))
tcp   LISTEN     0      128                                                        *:59762                                                                  *:*                   users:(("rpc.statd",pid=975,fd=9))
tcp   LISTEN     0      128                                                        *:22                                                                     *:*                   users:(("sshd",pid=970,fd=3))
tcp   LISTEN     0      100                                                127.0.0.1:25                                                                     *:*                   users:(("master",pid=1223,fd=13))
tcp   LISTEN     0      64                                                        :::2049                                                                  :::*                  
tcp   LISTEN     0      64                                                        :::43757                                                                 :::*                  
tcp   LISTEN     0      128                                                       :::111                                                                   :::*                   users:(("rpcbind",pid=553,fd=11))
tcp   LISTEN     0      128                                                       :::20048                                                                 :::*                   users:(("rpc.mountd",pid=985,fd=10))
tcp   LISTEN     0      128                                                       :::48658                                                                 :::*                   users:(("rpc.statd",pid=975,fd=11))
tcp   LISTEN     0      128                                                       :::22                                                                    :::*                   users:(("sshd",pid=970,fd=4))
tcp   LISTEN     0      100                                                      ::1:25                                                                    :::*                   users:(("master",pid=1223,fd=14))

```


# Работа с загрузчиком #
1. Попасть систему без пароля несколькими способами;<br/>
2. Установить систему с LVM, после чего переименовать VG;<br/>
3. Добавить модуль в initrd.<br/>
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.14 и образа<br/> 
&ensp;&ensp;CentOS 7 версии 1804_2
### Ход решения ###
