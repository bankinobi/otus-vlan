# Строим бонды и вланы

## Bonding

```
между centralRouter и inetRouter
"пробросить" 2 линка (общая inernal сеть) и объединить их в бонд 
проверить работу c отключением интерфейсов
```

В Vagrantfile зададим серверам два интерфейса, объединенных одной внутренней сетью.

С помощью Ansible создадим дополнительный bond-интерфейс:

Конфиг bond-master

```
DEVICE=bond0
ONBOOT=yes
TYPE=Bond
BONDING_MASTER=yes
IPADDR="{{ ipaddr }}"
PREFIX=30
BOOTPROTO=static
BONDING_OPTS="mode=1 miimon=100 fail_over_mac=1"
```

И объединим под ним интерфейсы eth1 и eth2, указав bond0 как мастер:

Конфиг bond-slave

```
BOOTPROTO=none
ONBOOT=yes
DEVICE="{{ item }}"
MASTER=bond0
SLAVE=yes
```


<details>
<summary>Убедимся, что сервера друг друга видят, и проверим статус нашего bond-а</summary>

    [vagrant@inetRouter ~]$ ping 192.168.255.2
    PING 192.168.255.2 (192.168.255.2) 56(84) bytes of data.
    64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.572 ms
    64 bytes from 192.168.255.2: icmp_seq=2 ttl=64 time=0.684 ms

    [vagrant@centralRouter ~]$ ping 192.168.255.1
    PING 192.168.255.1 (192.168.255.1) 56(84) bytes of data.
    64 bytes from 192.168.255.1: icmp_seq=1 ttl=64 time=0.543 ms
    64 bytes from 192.168.255.1: icmp_seq=2 ttl=64 time=0.672 ms

    [vagrant@inetRouter ~]$ cat /proc/net/bonding/bond0
    Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

    Bonding Mode: fault-tolerance (active-backup) (fail_over_mac active)
    Primary Slave: None
    Currently Active Slave: eth2
    MII Status: up
    MII Polling Interval (ms): 100
    Up Delay (ms): 0
    Down Delay (ms): 0

    Slave Interface: eth2
    MII Status: up
    Speed: 1000 Mbps
    Duplex: full
    Link Failure Count: 0
    Permanent HW addr: 08:00:27:0b:8b:7a
    Slave queue ID: 0

    Slave Interface: eth1
    MII Status: up
    Speed: 1000 Mbps
    Duplex: full
    Link Failure Count: 0
    Permanent HW addr: 08:00:27:6b:50:71
    Slave queue ID: 0
</details>


<details>
<summary>Для проверки настроек "погасим" один из интерфейсов</summary>

    [vagrant@inetRouter ~]$ sudo ifdown eth1
    Device 'eth1' successfully disconnected.
    [vagrant@inetRouter ~]$ ip a
    ...
    3: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
        link/ether 08:00:27:0b:8b:7a brd ff:ff:ff:ff:ff:ff
        inet 192.168.255.1/30 brd 192.168.255.3 scope global noprefixroute bond0
          valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fe0b:8b7a/64 scope link
          valid_lft forever preferred_lft forever
    4: eth1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
        link/ether 08:00:27:6b:50:71 brd ff:ff:ff:ff:ff:ff
    5: eth2: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
        link/ether 08:00:27:0b:8b:7a brd ff:ff:ff:ff:ff:ff
    [vagrant@inetRouter ~]$ cat /proc/net/bonding/bond0
    Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

    Bonding Mode: fault-tolerance (active-backup) (fail_over_mac active)
    Primary Slave: None
    Currently Active Slave: eth2
    MII Status: up
    MII Polling Interval (ms): 100
    Up Delay (ms): 0
    Down Delay (ms): 0

    Slave Interface: eth2
    MII Status: up
    Speed: 1000 Mbps
    Duplex: full
    Link Failure Count: 0
    Permanent HW addr: 08:00:27:0b:8b:7a
    Slave queue ID: 0
    [vagrant@inetRouter ~]$ ping 192.168.255.2
    PING 192.168.255.2 (192.168.255.2) 56(84) bytes of data.
    64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.538 ms
    64 bytes from 192.168.255.2: icmp_seq=2 ttl=64 time=0.584 ms
</details>

## Vlan

```
Создать сервера во внутренней сети
testClient1 - 10.10.10.254
testClient2 - 10.10.10.254
testServer1- 10.10.10.1
testServer2- 10.10.10.1
равести вланами
testClient1 <-> testServer1
testClient2 <-> testServer2
```

В Vagrantfile всем серверам добавим по интерфейсу в общей внутренней сети test-net.

С помощью Ansible создадим vlan-интерфейсы:

```
BOOTPROTO=static
VLAN=yes
ONBOOT=yes
IPADDR="{{ vlan.ipaddr }}"
NETMASK=255.255.255.0
DEVICE="{{ vlan.device }}"
```

Для проверки воспользуемся PING:

<details>
<summary>testClient1 <-> testServer1</summary>

    [vagrant@testServer1 ~]$ ping 10.10.10.254
    PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
    64 bytes from 10.10.10.254: icmp_seq=1 ttl=64 time=0.553 ms
    64 bytes from 10.10.10.254: icmp_seq=2 ttl=64 time=0.565 ms
    [vagrant@testClient1 ~]# ping 10.10.10.1
    PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
    64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.461 ms
    64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.642 ms
</details>

<details>
<summary>testClient2 <-> testServer2</summary>

    [vagrant@testServer2 ~]$ ping 10.10.10.254
    PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
    64 bytes from 10.10.10.254: icmp_seq=1 ttl=64 time=1.51 ms
    64 bytes from 10.10.10.254: icmp_seq=2 ttl=64 time=0.641 ms
    [vagrant@testClient2 ~]$ ping 10.10.10.1
    PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
    64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.497 ms
    64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.639 ms
</details>
