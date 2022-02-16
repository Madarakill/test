ISP, RTR-L, RTR-R, WEB-L, WEB-R, SRV

Настройка сети /etc/network/interfaces. В конце post-up /etc/gre.sh (RTR)
hostname
nano /etc/sysctl.conf , net.ipv4.ip_forward=1 (ISP,RTR). sysctl -p
apt install frr, iptables, ssh (RTR,ISP)
apt install curl, ssh, cifs-utils (WEB-L, WEB-R) + mdadm, samba (SRV) 
Настройка туннеля
для RTR-L nano gre.sh
#!/bin/bash
ip tunnel add tun1 mode gre local 4.4.4.100 remote 5.5.5.100 ttl 255
ip link set tun1 up
ip addr add 10.10.10.1/30 dev tun1
для RTR-R  nano gre.sh
#!/bin/bash
ip tunnel add tun1 mode gre local 5.5.5.100 remote 4.4.4.100 ttl 255
ip link set tun1 up
ip addr add 10.10.10.2/30 dev tun1
chmod +x /etc/gre.sh
nano /etc/crontab
@reboot root /etc/gre.sh
Настройка frr (RTR)
nano /etc/frr/daemons раскомментировать ospfd=yes
systemctl restart, enable frr
vtysh
en
conf t
router eigrp 6500
ДЛЯ RTR-L
network 192.168.100.0/24
network 10.10.10.0/30
ДЛЯ RTR-R
network 172.16.100.0/24
network 10.10.10.0/30
do wr
Трансляция трафика в интернет 
RTR-L: iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o ensISP -j MASQUERADE
RTR-R: iptables -t nat -A POSTROUTING -s 172.16.100.0/24 -o ensISP -j MASQUERADE
Разрешение порта 
iptables -t filter -A INPUT -p tcp -dport 5000 -j ACCEPT
iptables -t filter -A OUTPUT -p tcp -dport 5000 -j ACCEPT
Добавить 2 ж. диска на SRV. 
mdadm --create --verbose -l 1 /dev/md127 --raid-devices=2 /dev/sd{b,c}
echo DEVICE partitions >> /etc/mdadm/mdadm.conf
mdadm --detail --verbose --scan | head -1 >> /etc/mdadm/mdadm.conf
nano /etc/fstab 
/dev/md127	/mnt/storage	ext4	default	0	0
Создание расшаренной папки на SRV
nano /etc/samba/smb.conf
[share]
path = /opt/share
read only = no
writeable = yes
browseable = yes
guest ok = yes
systemctl restart smbd.service 
Монтирование папки на WEB-L WEB-R
mount.cifs //192.168.100.200/share /opt/share
nano /etc/crontab
@reboot mount.cifs //192.168.100.200/share /opt/share
//192.168.100.200/share /opt/share cifs	rw
Установка Docker Web на WEB-L WEB-R
Вставить диск docker. mount /dev/sr0 /media/cdrom
mkdir /root/inst
mkdir /root/app
cp /media/cdrom/*.deb /root/inst
cp /media/cdrom/app…zip /root/app
apt install /root/inst/*
docker image load -i /root/app/app…zip
docker run -dp 5000:5000 --name app -v db:/var/lib/postgresql appdocker0
curl localhost:5000/	Вбить инфу: /add?message=…. Посмотреть /get
