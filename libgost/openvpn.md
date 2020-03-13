Установка поддержки ГОСТ криптографии для openssl описана в [https://github.com/gost-engine/engine](https://github.com/gost-engine/engine)  
Для опытов используется стабильная версия gost-engine 1.1.0.3  
  
Эксперименты проводились на выделенном сервере

```
~# lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 18.04.3 LTS
Release:	18.04
Codename:	bionic
~# openssl version
OpenSSL 1.1.1  11 Sep 2018
~# openvpn --version
OpenVPN 2.4.4 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on May 14 2019
library versions: OpenSSL 1.1.1  11 Sep 2018, LZO 2.08
Originally developed by James Yonan
Copyright (C) 2002-2017 OpenVPN Technologies, Inc. <sales@openvpn.net>
Compile time defines: enable_async_push=no enable_comp_stub=no enable_crypto=yes enable_crypto_ofb_cfb=yes enable_debug=yes enable_def_auth=yes enable_dependency_tracking=no enable_dlopen=unknown enable_dlopen_self=unknown enable_dlopen_self_static=unknown enable_fast_install=needless enable_fragment=yes enable_iproute2=yes enable_libtool_lock=yes enable_lz4=yes enable_lzo=yes enable_maintainer_mode=no enable_management=yes enable_multihome=yes enable_pam_dlopen=no enable_pedantic=no enable_pf=yes enable_pkcs11=yes enable_plugin_auth_pam=yes enable_plugin_down_root=yes enable_plugins=yes enable_port_share=yes enable_selinux=no enable_server=yes enable_shared=yes enable_shared_with_static_runtimes=no enable_silent_rules=no enable_small=no enable_static=yes enable_strict=no enable_strict_options=no enable_systemd=yes enable_werror=no enable_win32_dll=yes enable_x509_alt_username=yes with_aix_soname=aix with_crypto_library=openssl with_gnu_ld=yes with_mem_check=no with_sysroot=no
```

Проверим поддержку шифров gost

```
~# openssl ciphers|tr ':' '\n'|grep GOST
GOST2012-GOST8912-GOST8912
GOST2001-GOST89-GOST89
```

Создадим тестовую директорию, будем экспериментировать в ней

```
~# mkdir gost && cd gost
```

Копируем оригинальный файл конфигурации openssl

```
~/gost# cp /etc/ssl/openssl.cnf .
```

Добавим новую секцию для генерации сертификатов сервера

```
~/gost# cat <<EOF >>openssl.cnf

[ server ]
authorityKeyIdentifier = keyid,issuer
extendedKeyUsage = serverAuth
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment

EOF
```

Установим переменную окружения OPENSSL_CONF

```
~/gost# export OPENSSL_CONF=$PWD/openssl.cnf
```

Создаем корневой сертификат и ключ GOST

```
~/gost# openssl req -x509 -subj "/CN=my-ca" -newkey gost2012_256 -pkeyopt paramset:A -nodes -keyout ca.key -out ca.crt -days 3650
Generating a GOST2012_256 private key
writing new private key to 'ca.key'
-----
```

Для сервера генерируем ключ и запрос сертификата

```
~/gost# openssl req -subj "/CN=server" -newkey gost2012_256 -pkeyopt paramset:A -nodes -keyout server.key -out server.csr
Generating a GOST2012_256 private key
writing new private key to 'server.key'
-----
```

Для сервера создаем сертификат, подписанный корневым сертификатом

```
~/gost# openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 3650 -extfile $PWD/openssl.cnf -extensions server
Signature ok
subject=CN = server
Getting CA Private Key
```

Для клиентов генерируем ключи и запросы сертификатов

```
~/gost# openssl req -subj "/CN=client01" -newkey gost2012_256 -pkeyopt paramset:A -nodes -keyout client01.key -out client01.csr
Generating a GOST2012_256 private key
writing new private key to 'client01.key'
-----
~/gost# openssl req -subj "/CN=client02" -newkey gost2012_256 -pkeyopt paramset:A -nodes -keyout client02.key -out client02.csr
Generating a GOST2012_256 private key
writing new private key to 'client02.key'
-----
~/gost# openssl req -subj "/CN=client03" -newkey gost2012_256 -pkeyopt paramset:A -nodes -keyout client03.key -out client03.csr
Generating a GOST2012_256 private key
writing new private key to 'client03.key'
-----
```

Для клиентов создаем сертификаты, подписанные корневым сертификатом

```
~/gost# openssl x509 -req -in client01.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client01.crt -days 3650
Signature ok
subject=CN = client01
Getting CA Private Key
~/gost# openssl x509 -req -in client02.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client02.crt -days 3650
Signature ok
subject=CN = client02
Getting CA Private Key
~/gost# openssl x509 -req -in client03.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client03.crt -days 3650
Signature ok
subject=CN = client03
Getting CA Private Key
```

Удаляем переменную окружения OPENSSL_CONF, файл openssl.cnf и запросы сертификатов, они больше не потребуются

```
~/gost# unset OPENSSL_CONF && rm openssl.cnf *.csr
```

Создадим DH (Diffie Hellman) ключ

```
~/gost# openssl dhparam -out dh2048.pem 2048
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
.............................
.........+.............+..+..
..............++*++*++*++*
```

Строим конфигурацию сервера, тут-же в файле server.conf

```
cat <<EOF >server.conf
port 4194
proto udp
dev tun
engine gost
ncp-ciphers grasshopper-cbc
tls-cipher GOST2012-GOST8912-GOST8912
auth md_gost12_256
ca $PWD/ca.crt
cert $PWD/server.crt
key $PWD/server.key
dh $PWD/dh2048.pem
server 10.4.0.0 255.255.255.0
ifconfig-pool-persist $PWD/ipp.txt
keepalive 10 120
persist-key
persist-tun
verb 3
explicit-exit-notify 1
mute-replay-warnings
push "dhcp-option DNS 10.4.0.1"
EOF
```

Опционально создаем файл ipp.txt привязки ip адресов клиентам

```
cat <<EOF >ipp.txt
client01,10.4.0.4
client02,10.4.0.8
client03,10.4.0.12
EOF
```

Запускаем vpn сервер

```
~/gost# openvpn $PWD/server.conf
Fri Mar 13 10:59:25 2020 OpenVPN 2.4.4 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on May 14 2019
Fri Mar 13 10:59:25 2020 library versions: OpenSSL 1.1.1  11 Sep 2018, LZO 2.08
Fri Mar 13 10:59:25 2020 Note: OpenSSL hardware crypto engine functionality is not available
Fri Mar 13 10:59:25 2020 Diffie-Hellman initialized with 2048 bit key
Fri Mar 13 10:59:25 2020 ROUTE_GATEWAY 1.2.3.1/255.255.255.0 IFACE=eth0 HWADDR=00:16:3e:95:22:0c
Fri Mar 13 10:59:25 2020 TUN/TAP device tun2 opened
Fri Mar 13 10:59:25 2020 TUN/TAP TX queue length set to 100
Fri Mar 13 10:59:25 2020 do_ifconfig, tt->did_ifconfig_ipv6_setup=0
Fri Mar 13 10:59:25 2020 /sbin/ip link set dev tun2 up mtu 1500
Fri Mar 13 10:59:25 2020 /sbin/ip addr add dev tun2 local 10.4.0.1 peer 10.4.0.2
Fri Mar 13 10:59:25 2020 /sbin/ip route add 10.4.0.0/24 via 10.4.0.2
Fri Mar 13 10:59:25 2020 Could not determine IPv4/IPv6 protocol. Using AF_INET
Fri Mar 13 10:59:25 2020 Socket Buffers: R=[212992->212992] S=[212992->212992]
Fri Mar 13 10:59:25 2020 UDPv4 link local (bound): [AF_INET][undef]:4194
Fri Mar 13 10:59:25 2020 UDPv4 link remote: [AF_UNSPEC]
Fri Mar 13 10:59:25 2020 MULTI: multi_init called, r=256 v=256
Fri Mar 13 10:59:25 2020 IFCONFIG POOL: base=10.4.0.4 size=62, ipv6=0
Fri Mar 13 10:59:25 2020 ifconfig_pool_read(), in='client01,10.4.0.4', TODO: IPv6
Fri Mar 13 10:59:25 2020 succeeded -> ifconfig_pool_set()
Fri Mar 13 10:59:25 2020 ifconfig_pool_read(), in='client02,10.4.0.8', TODO: IPv6
Fri Mar 13 10:59:25 2020 succeeded -> ifconfig_pool_set()
Fri Mar 13 10:59:25 2020 ifconfig_pool_read(), in='client03,10.4.0.12', TODO: IPv6
Fri Mar 13 10:59:25 2020 succeeded -> ifconfig_pool_set()
Fri Mar 13 10:59:25 2020 IFCONFIG POOL LIST
Fri Mar 13 10:59:25 2020 client01,10.4.0.4
Fri Mar 13 10:59:25 2020 client02,10.4.0.8
Fri Mar 13 10:59:25 2020 client03,10.4.0.12
Fri Mar 13 10:59:25 2020 Initialization Sequence Completed
```

На ПК настраиваем клиента и проверяем работоспособность  
Создаем директорию, копируем сертификаты и ключ

```
~$ mkdir gost && cd gost
~/gost$ scp root@my-server.ru:/root/gost/client01.* root@my-server.ru:/root/gost/ca.crt .
client01.crt
client01.key
ca.crt
```

Строим конфигурацию клиента
  
```
~$ cat <<EOF >client.conf
client
dev tun
proto udp
remote my-server.ru 4194
nobind
persist-key
persist-tun
engine gost
ncp-ciphers grasshopper-cbc
tls-cipher GOST2012-GOST8912-GOST8912
auth md_gost12_256
remote-cert-tls server
ca $PWD/ca.crt
cert $PWD/client01.crt
key $PWD/client01.key
verb 3
EOF
```

Запускаем vpn клиента  

```
~/gost$ sudo openvpn $PWD/client.conf
[sudo] password for user: 
Fri Mar 13 15:13:53 2020 OpenVPN 2.4.4 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on May 14 2019
Fri Mar 13 15:13:53 2020 library versions: OpenSSL 1.1.1  11 Sep 2018, LZO 2.08
Fri Mar 13 15:13:53 2020 Note: OpenSSL hardware crypto engine functionality is not available
Fri Mar 13 15:13:53 2020 TCP/UDP: Preserving recently used remote address: [AF_INET]1.2.3.4:4194
Fri Mar 13 15:13:53 2020 Socket Buffers: R=[212992->212992] S=[212992->212992]
Fri Mar 13 15:13:53 2020 UDP link local: (not bound)
Fri Mar 13 15:13:53 2020 UDP link remote: [AF_INET]1.2.3.4:4194
Fri Mar 13 15:13:53 2020 TLS: Initial packet from [AF_INET]1.2.3.4:4194, sid=15b2344a 45585c3b
Fri Mar 13 15:13:53 2020 VERIFY OK: depth=1, CN=my-ca
Fri Mar 13 15:13:53 2020 VERIFY KU OK
Fri Mar 13 15:13:53 2020 Validating certificate extended key usage
Fri Mar 13 15:13:53 2020 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
Fri Mar 13 15:13:53 2020 VERIFY EKU OK
Fri Mar 13 15:13:53 2020 VERIFY OK: depth=0, CN=server
Fri Mar 13 15:13:53 2020 Control Channel: TLSv1.2, cipher TLSv1.0 GOST2012-GOST8912-GOST8912
Fri Mar 13 15:13:53 2020 [server] Peer Connection Initiated with [AF_INET]1.2.3.4:4194
Fri Mar 13 15:13:54 2020 SENT CONTROL [server]: 'PUSH_REQUEST' (status=1)
Fri Mar 13 15:13:54 2020 PUSH: Received control message: 'PUSH_REPLY,dhcp-option DNS 10.4.0.1,route 10.4.0.1,topology net30,ping 10,ping-restart 120,ifconfig 10.4.0.6 10.4.0.5,peer-id 0,cipher grasshopper-cbc'
Fri Mar 13 15:13:54 2020 OPTIONS IMPORT: timers and/or timeouts modified
Fri Mar 13 15:13:54 2020 OPTIONS IMPORT: --ifconfig/up options modified
Fri Mar 13 15:13:54 2020 OPTIONS IMPORT: route options modified
Fri Mar 13 15:13:54 2020 OPTIONS IMPORT: --ip-win32 and/or --dhcp-option options modified
Fri Mar 13 15:13:54 2020 OPTIONS IMPORT: peer-id set
Fri Mar 13 15:13:54 2020 OPTIONS IMPORT: adjusting link_mtu to 1624
Fri Mar 13 15:13:54 2020 OPTIONS IMPORT: data channel crypto options modified
Fri Mar 13 15:13:54 2020 Data Channel: using negotiated cipher 'grasshopper-cbc'
Fri Mar 13 15:13:54 2020 Outgoing Data Channel: Cipher 'grasshopper-cbc' initialized with 256 bit key
Fri Mar 13 15:13:54 2020 Outgoing Data Channel: Using 256 bit message hash 'md_gost12_256' for HMAC authentication
Fri Mar 13 15:13:54 2020 Incoming Data Channel: Cipher 'grasshopper-cbc' initialized with 256 bit key
Fri Mar 13 15:13:54 2020 Incoming Data Channel: Using 256 bit message hash 'md_gost12_256' for HMAC authentication
Fri Mar 13 15:13:54 2020 ROUTE_GATEWAY 192.168.95.5/255.255.255.0 IFACE=enp2s0 HWADDR=30:9c:23:80:e4:cd
Fri Mar 13 15:13:54 2020 TUN/TAP device tun2 opened
Fri Mar 13 15:13:54 2020 TUN/TAP TX queue length set to 100
Fri Mar 13 15:13:54 2020 do_ifconfig, tt->did_ifconfig_ipv6_setup=0
Fri Mar 13 15:13:54 2020 /sbin/ip link set dev tun2 up mtu 1500
Fri Mar 13 15:13:54 2020 /sbin/ip addr add dev tun2 local 10.4.0.6 peer 10.4.0.5
Fri Mar 13 15:13:54 2020 /sbin/ip route add 10.4.0.1/32 via 10.4.0.5
Fri Mar 13 15:13:54 2020 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
Fri Mar 13 15:13:54 2020 Initialization Sequence Completed
```

Смотрим логи на сервере

```
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 Note: OpenSSL hardware crypto engine functionality is not available
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 TLS: Initial packet from [AF_INET]94.25.172.125:2172, sid=4d0a1161 371e23ad
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 VERIFY OK: depth=1, CN=my-ca
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 VERIFY OK: depth=0, CN=client01
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 peer info: IV_VER=2.4.4
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 peer info: IV_PLAT=linux
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 peer info: IV_PROTO=2
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 peer info: IV_NCP=2
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 peer info: IV_LZ4=1
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 peer info: IV_LZ4v2=1
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 peer info: IV_LZO=1
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 peer info: IV_COMP_STUB=1
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 peer info: IV_COMP_STUBv2=1
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 peer info: IV_TCPNL=1
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 Control Channel: TLSv1.2, cipher TLSv1.0 GOST2012-GOST8912-GOST8912
Fri Mar 13 12:13:53 2020 94.25.172.125:2172 [client01] Peer Connection Initiated with [AF_INET]94.25.172.125:2172
Fri Mar 13 12:13:53 2020 client01/94.25.172.125:2172 MULTI_sva: pool returned IPv4=10.4.0.6, IPv6=(Not enabled)
Fri Mar 13 12:13:53 2020 client01/94.25.172.125:2172 MULTI: Learn: 10.4.0.6 -> client01/94.25.172.125:2172
Fri Mar 13 12:13:53 2020 client01/94.25.172.125:2172 MULTI: primary virtual IP for client01/94.25.172.125:2172: 10.4.0.6
Fri Mar 13 12:13:54 2020 client01/94.25.172.125:2172 PUSH: Received control message: 'PUSH_REQUEST'
Fri Mar 13 12:13:54 2020 client01/94.25.172.125:2172 SENT CONTROL [client01]: 'PUSH_REPLY,dhcp-option DNS 10.4.0.1,route 10.4.0.1,topology net30,ping 10,ping-restart 120,ifconfig 10.4.0.6 10.4.0.5,peer-id 0,cipher grasshopper-cbc' (status=1)
Fri Mar 13 12:13:54 2020 client01/94.25.172.125:2172 Data Channel: using negotiated cipher 'grasshopper-cbc'
Fri Mar 13 12:13:54 2020 client01/94.25.172.125:2172 Outgoing Data Channel: Cipher 'grasshopper-cbc' initialized with 256 bit key
Fri Mar 13 12:13:54 2020 client01/94.25.172.125:2172 Outgoing Data Channel: Using 256 bit message hash 'md_gost12_256' for HMAC authentication
Fri Mar 13 12:13:54 2020 client01/94.25.172.125:2172 Incoming Data Channel: Cipher 'grasshopper-cbc' initialized with 256 bit key
Fri Mar 13 12:13:54 2020 client01/94.25.172.125:2172 Incoming Data Channel: Using 256 bit message hash 'md_gost12_256' for HMAC authentication
```

На клиенте смотрим на новый сетевой интерфейс и пингуем сервер

```
~$ ifconfig tun2
tun2: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 10.4.0.6  netmask 255.255.255.255  destination 10.4.0.5
        inet6 fe80::107c:bc70:900b:fa8a  prefixlen 64  scopeid 0x20<link>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 100  (UNSPEC)
        RX packets 6  bytes 288 (288.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9  bytes 432 (432.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

~$ ping -c 3 10.4.0.1
PING 10.4.0.1 (10.4.0.1) 56(84) bytes of data.
64 bytes from 10.4.0.1: icmp_seq=1 ttl=64 time=36.6 ms
64 bytes from 10.4.0.1: icmp_seq=2 ttl=64 time=41.4 ms
64 bytes from 10.4.0.1: icmp_seq=3 ttl=64 time=71.4 ms

--- 10.4.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 36.663/49.858/71.443/15.388 ms
```

И наконец, попробуем поднять ГОСТ-ового vpn клиента на роутере.  
Для опытов используется TP-LINK tl-mr3020 v3.20  
Сначала добавим в feeds.conf новый фид для сборки библиотеки

```
src-git alex18296 https://github.com/alex18296/openwrt-feed.git^87259da556e0620e208554bad35d9618cf89bf67
```
Выполним загрузку и установку этого фида

```
~/openwrt$ ./scripts/feeds update alex18296 && ./scripts/feeds install -p alex18296 libgost
Updating feed 'alex18296' from 'https://github.com/alex18296/openwrt-feed.git^87259da556e0620e208554bad35d9618cf89bf67' ...
Cloning into './feeds/alex18296'...
remote: Enumerating objects: 13, done.
remote: Counting objects: 100% (13/13), done.
remote: Compressing objects: 100% (12/12), done.
remote: Total 13 (delta 1), reused 9 (delta 0), pack-reused 0
Unpacking objects: 100% (13/13), done.
Switched to a new branch '87259da556e0620e208554bad35d9618cf89bf67'
/home/user/openwrt
Create index file './feeds/alex18296.index' 
Collecting package info: done
Collecting target info: done
Installing package 'libgost' from alex18296
```

В конфигурации сборки подключим эту библиотеку

```
 > Libraries > SSL
   <*> libgost........... The engine support for Russian GOST crypto algorithms.
```

Не забудьте отключить этот фид для обновлений через opkg

```
 > Image configuration > Separate feed repositories
   < >   Enable feed alex18296
```

Соберем прошивку с подключенным openvpn-openssl и опционально openssl-util  
Загрузим в устройство и проверим подключилась ли поддержка ГОСТ-а в openssl

```
root@OpenWrt:~# openssl ciphers|tr ':' '\n'|grep GOST
GOST2012-GOST8912-GOST8912
GOST2001-GOST89-GOST89
```

Сохраним сертификаты и ключ на роутере в  
/etc/openvpn/ca.crt  
/etc/openvpn/client02.crt  
/etc/openvpn/client02.key  
  
На роутере строим конфигурацию клиента в /etc/openvpn/client.conf

```
root@OpenWrt:~# cat <<EOF >/etc/openvpn/client.conf
client
dev tun
proto udp
remote my-server.ru 4194
nobind
persist-key
persist-tun
engine gost
cipher grasshopper-cbc
tls-cipher GOST2012-GOST8912-GOST8912
auth md_gost12_256
remote-cert-tls server
ca /etc/openvpn/ca.crt
cert /etc/openvpn/client02.crt
key /etc/openvpn/client02.key
verb 3
EOF
```

На роутере запускаем клиента

```
root@OpenWrt:~# /etc/init.d/openvpn restart

```

Смотрим логи на роутере

```
root@OpenWrt:/tmp# logread -f
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: OpenVPN 2.4.7 mipsel-openwrt-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [MH/PKTINFO] [AEAD]
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: library versions: OpenSSL 1.1.1d  10 Sep 2019, LZO 2.10
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: Initializing OpenSSL support for engine 'gost'
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: TCP/UDP: Preserving recently used remote address: [AF_INET]1.2.3.4:4194
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: Socket Buffers: R=[163840->163840] S=[163840->163840]
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: UDP link local: (not bound)
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: UDP link remote: [AF_INET]1.2.3.4:4194
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: TLS: Initial packet from [AF_INET]1.2.3.4:4194, sid=13f978f2 209e69ce
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: VERIFY OK: depth=1, CN=my-ca
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: VERIFY KU OK
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: Validating certificate extended key usage
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: VERIFY EKU OK
Fri Mar 13 13:41:35 2020 daemon.notice openvpn(client)[1820]: VERIFY OK: depth=0, CN=server
Fri Mar 13 13:41:36 2020 daemon.warn openvpn(client)[1820]: WARNING: 'link-mtu' is used inconsistently, local='link-mtu 1569', remote='link-mtu 1553'
Fri Mar 13 13:41:36 2020 daemon.warn openvpn(client)[1820]: WARNING: 'cipher' is used inconsistently, local='cipher grasshopper-cbc', remote='cipher BF-CBC'
Fri Mar 13 13:41:36 2020 daemon.warn openvpn(client)[1820]: WARNING: 'keysize' is used inconsistently, local='keysize 256', remote='keysize 128'
Fri Mar 13 13:41:36 2020 daemon.notice openvpn(client)[1820]: Control Channel: TLSv1.2, cipher TLSv1.0 GOST2012-GOST8912-GOST8912
Fri Mar 13 13:41:36 2020 daemon.notice openvpn(client)[1820]: [server] Peer Connection Initiated with [AF_INET]1.2.3.4:4194
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: SENT CONTROL [server]: 'PUSH_REQUEST' (status=1)
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: PUSH: Received control message: 'PUSH_REPLY,dhcp-option DNS 10.4.0.1,route 10.4.0.1,topology net30,ping 10,ping-restart 120,ifconfig 10.4.0.10 10.4.0.9,peer-id 1,cipher grasshopper-cbc'
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: OPTIONS IMPORT: timers and/or timeouts modified
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: OPTIONS IMPORT: --ifconfig/up options modified
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: OPTIONS IMPORT: route options modified
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: OPTIONS IMPORT: --ip-win32 and/or --dhcp-option options modified
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: OPTIONS IMPORT: peer-id set
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: OPTIONS IMPORT: adjusting link_mtu to 1624
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: OPTIONS IMPORT: data channel crypto options modified
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: Outgoing Data Channel: Cipher 'grasshopper-cbc' initialized with 256 bit key
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: Outgoing Data Channel: Using 256 bit message hash 'md_gost12_256' for HMAC authentication
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: Incoming Data Channel: Cipher 'grasshopper-cbc' initialized with 256 bit key
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: Incoming Data Channel: Using 256 bit message hash 'md_gost12_256' for HMAC authentication
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: TUN/TAP device tun0 opened
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: TUN/TAP TX queue length set to 100
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: /sbin/ifconfig tun0 10.4.0.10 pointopoint 10.4.0.9 mtu 1500
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: /sbin/route add -net 10.4.0.1 netmask 255.255.255.255 gw 10.4.0.9
Fri Mar 13 13:41:37 2020 daemon.warn openvpn(client)[1820]: WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
Fri Mar 13 13:41:37 2020 daemon.notice openvpn(client)[1820]: Initialization Sequence Completed
```

Смотрим на новый сетевой интерфейс и пингуем сервер

```
root@OpenWrt:~# ifconfig tun0
tun0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
          inet addr:10.4.0.10  P-t-P:10.4.0.9  Mask:255.255.255.255
          inet6 addr: fe80::d21d:45fd:8711:5f9b/64 Scope:Link
          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
          RX packets:3 errors:0 dropped:0 overruns:0 frame:0
          TX packets:7 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:100 
          RX bytes:252 (252.0 B)  TX bytes:556 (556.0 B)

root@OpenWrt:~# ping -c 3 10.4.0.1
PING 10.4.0.1 (10.4.0.1): 56 data bytes
64 bytes from 10.4.0.1: seq=0 ttl=64 time=33.763 ms
64 bytes from 10.4.0.1: seq=1 ttl=64 time=32.255 ms
64 bytes from 10.4.0.1: seq=2 ttl=64 time=42.055 ms

--- 10.4.0.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 32.255/36.024/42.055 ms
```
