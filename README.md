## Описание файлов в директории
logFileFull.log - полный лог выполнения  
used_commands.txt - команды которые использовал  
errorsResolve.txt - ошибки которые встретил и их решение  
errorsResolve.txt - ошибки с которыми встретился и их решение  
usedInetResorce.txt - ресурсы в интернете которые мне помогли решить ДЗ

## 1. Попасть в систему без пароля несколькими способами
### 1. Используем rd.break  
Когда находимся в меню grub нажимаем e, редактируем строчку загрузки, чтобы она получилось такой  
До
```
linux16 /boot/vmlinuz-3.10.0-1127.el7.x86_64 root=UUID=1c419d6c-5064-4a2b-953c-05b2c67edb15 ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto
````
Убираем упоминание console и добавляем rd.break  
После
```
linux16 /boot/vmlinuz-3.10.0-1127.el7.x86_64 root=UUID=1c419d6c-5064-4a2b-953c-05b2c67edb15 ro no_timer_check rd.break net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto
```
Попадаем в emergency mode и выполняем нужные комманды
```
switc_root:/# mount | grep sysroot
/dev/sda1 on /sysroot type xfs (ro,relatime,attr2,inode64,noquota)
```
перемонтируем с взоможностью записи
```
switc_root:/# mount -o remount,rw /sysroot
/dev/sda1 on /sysroot type xfs (rw,relatime,attr2,inode64,noquota)
```
меняем корневой каталог и изменяем пароль root, так же не забываем создать файл autorelabel чтобы selinux не заблокировал файлы /etc/(passwd|shadow)
```
switc_root:/# chroot /sysroot
sh-4.2# passwd
sh-4.2# touch /.autorelabel
sh-4.2# exit
switc_root:/# reboot
```
### 2. Используем init=/sysroot/bin/sh  
Выполняем все тоже самое, только вместо rd.break прописываем init=/sysroot/bin/sh
```
linux16 /boot/vmlinuz-3.10.0-1127.el7.x86_64 root=UUID=1c419d6c-5064-4a2b-953c-05b2c67edb15 ro no_timer_check init=/sysroot/bin/sh net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto
```
Так же выполняем последовательность комманд для смены пароля root
```
:/# mount | grep sysroot
/dev/sda1 on /sysroot type xfs (ro,relatime,attr2,inode64,noquota)
:/# mount -o remount,rw /sysroot
/dev/sda1 on /sysroot type xfs (rw,relatime,attr2,inode64,noquota)
:/# chroot /sysroot
:/# passwd
:/# touch /.autorelabel
:/# exit
:/# reboot
```
### 3. Использование systemd.unit=rescue.target  
У меня не получилось использовать этот метод для сброса пароля. Чтобы войти в изменение системы, идет запрос пароля root. Для каких то правок этот метод подходит, но нужно знать пароль root. Либо внести предварительные настройки при загрузке этого метода чтобы система не запрашивала пароль.  
## 2. Установить систему с LVM, после чего переименовать VG
При установки системы в virtualBox в графическом интерфейсе выбрал поставить систему с использованием LVM.  
После загрузки системы выполнил команду
```
[root@vm-gkuOA1801016 ~]# lsblk
```
NAME           |MAJ:MIN|RM |SIZE  |RO |TYPE |MOUNTPOINT
:--------------|:-----:|:-:|:----:|:-:|:---:|:---------:|
sda            |   8:0 |  0|   10G|  0| disk|
├─sda1         |   8:1 |  0|  500M|  0| part| /boot
└─sda2         |   8:2 |  0|  9,5G|  0| part|
  ├─centos-root| 253:0 |  0|  8,5G|  0| lvm | /
  └─centos-swap| 253:1 |  0|    1G|  0| lvm | [SWAP]
sr0            |  11:0 |  1| 1024M|  0| rom |
```
[root@vm-gkuOA1801016 ~]# vgdisplay
  --- Volume group ---
  VG Name               centos
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               9,50 GiB
  PE Size               4,00 MiB
  Total PE              2433
  Alloc PE / Size       2433 / 9,50 GiB
  Free  PE / Size       0 / 0
  VG UUID               5mZFws-AWyZ-ARf8-JSS0-JfWu-ECes-1C2aT7
```
Перезагрузился и после правки grub зашел в emergency mode
```
linux16 /boot/vmlinuz-3.10.0-1127.el7.x86_64 root=UUID=1c419d6c-5064-4a2b-953c-05b2c67edb15 ro no_timer_check rd.break net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto
```
Отмонтируем sysroot
```
umount /sysroot
```
Отлключил блокировку изменения lvm
```
switc_root:/# grep 'locking_type =' /etc/lvm/lvm.conf
         locking_type = 4
switc_root:/# vi /etc/lvm/lvm.conf
switc_root:/# grep 'locking_type =' /etc/lvm/lvm.conf
         locking_type = 1
```
Изменяем статус vg на неактивный, переименовываем vg, и снова меняем статус
```
switc_root:/# lvm vgchange -an centos 
switc_root:/# lvm vgrename centos alex
switc_root:/# lvm vgchange -ay alex
```
Монтируем lv с корневым диском, и меняем корневой каталог
```
switc_root:/# mkdir /mnt
switc_root:/# mount /dev/alex/root /mnt/
switc_root:/# chroot /mnt/
```
Правим fstab, добавляем autorelabel и перезагружаемся
```
sh-4.2# sed -i 's/centos-/alex-/' /etc/fstab
sh-4.2# touch /.autorelabel
sh-4.2# exit
switc_root:/# reboot
```
Правим grub для загрузки системы
```
linux16 /vmlinuz-3.10.0-1160.el7.x86_64 root=/dev/mapper/alex-root ro crashkernel=auto rd.lvm.lv=alex/root rd.lvm.lv=alex/swap rhgb quiet LANG=en_US.UTF-8
```
Уже войдя в систему правим /etc/default/grub и пересоздаем /etc/grub2.conf.
```
[root@vm-gkuOA1801016 ~]# sed -i 's|centos/|alex/|g' /etc/default/grub
[root@vm-gkuOA1801016 ~]# grub2-mkconfig -o /etc/grub2.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-1160.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1160.el7.x86_64.img
Found linux image: /boot/vmlinuz-0-rescue-bba117eb1ebb45aaae14b7eb31dea8d9
Found initrd image: /boot/initramfs-0-rescue-bba117eb1ebb45aaae14b7eb31dea8d9.img
done
[root@vm-gkuOA1801016 ~]# grep alex /etc/grub2.cfg
        linux16 /vmlinuz-3.10.0-1160.el7.x86_64 root=/dev/mapper/alex-root ro crashkernel=auto rd.lvm.lv=alex/root rd.lvm.lv=alex/swap rhgb quiet
        linux16 /vmlinuz-0-rescue-bba117eb1ebb45aaae14b7eb31dea8d9 root=/dev/mapper/alex-root ro crashkernel=auto rd.lvm.lv=alex/root rd.lvm.lv=alex/swap rhgb quiet
[root@vm-gkuOA1801016 ~]# lsblk
```
NAME           |MAJ:MIN|RM |SIZE  |RO |TYPE |MOUNTPOINT
:--------------|:-----:|:-:|:----:|:-:|:---:|:---------:|
sda            |   8:0 |  0|   10G|  0| disk|
├─sda1         |   8:1 |  0|  500M|  0| part| /boot
└─sda2         |   8:2 |  0|  9,5G|  0| part|
  ├─alex-root  | 253:0 |  0|  8,5G|  0| lvm | /
  └─alex-swap  | 253:1 |  0|    1G|  0| lvm | [SWAP]
sr0            |  11:0 |  1| 1024M|  0| rom |


## 3. Добавить модуль в initrd
Создаем папку в каталоге модулей  /usr/lib/dracut/modules.d/
```
mkdir  /usr/lib/dracut/modules.d/01test
```
Создаем там два скрипта и добавляем атрибут на выполнение. И создаем заново initramfs
```
[root@vm-gkuOA1801016 ~]# vi /usr/lib/dracut/modules.d/01test/module-setup.sh
[root@vm-gkuOA1801016 ~]# vi /usr/lib/dracut/modules.d/01test/test.sh
[root@vm-gkuOA1801016 ~]# chmod +x /usr/lib/dracut/modules.d/01test/*
[root@vm-gkuOA1801016 ~]# ll /usr/lib/dracut/modules.d/01test/*
-rwxr-xr-x. 1 root root 117 Sep 16 08:18 /usr/lib/dracut/modules.d/01test/module-setup.sh
-rwxr-xr-x. 1 root root 251 Sep 16 08:20 /usr/lib/dracut/modules.d/01test/test.sh
[root@vm-gkuOA1801016 ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
[root@vm-gkuOA1801016 ~]# lsinitrd -m /boot/initramfs-3.10.0-1127.el7.x86_64.img | grep test
test
```
Удаляем из /etc/default/grub упоминания console, пересоздаем /etc/grub2.cfg и перезагружаем
```
[root@vm-gkuOA1801016 ~]# nano /etc/default/grub
[root@vm-gkuOA1801016 ~]# grub2-mkconfig -o /etc/grub2.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-1127.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1127.el7.x86_64.img
done
[root@vm-gkuOA1801016 ~]# reboot
```

### Содержание скрпитов
#### module-setup.sh
```
#!/bin/bash
check() {
  return 0
}
depends() {
  return 0
}
install() {
  inst_hook cleanup 00 "${moddir}/test.sh"
}
```
#### test.sh
```
#!/bin/bash
exec 0<>/dev/console 1<>/dev/console
2<>/dev/console
cat <<'msgend'
Hello! You are in dracut module!
___________________
< I'm dracut module >
--------------------
 \
 .-----.
 | o_o |
 | \_/ |
 / / \ \
(  | |  )
/'\_ _/`\
\  )=(  /
```
P.S. скриншот пингвина лежит тут же