# Soft-RAID во FreeBSD

Список команд для создания soft-RAID во FreeBSD на примере, когда имеем 2 жестких диска /dev/ada0 и /dev/ada1:

```
sysctl kern.geom.debugflags=16
gmirror label -v -b round-robin gm0 /dev/ada0
echo 'geom_mirror_load=YES' >> /boot/loader.conf
sed -i 's/ada0/mirror\/gm0/' /etc/fstab
reboot
gmirror insert gm0 /dev/ada1
```

А после можно сидеть и наблюдать за процессом синхронизации `gmirror status`.
