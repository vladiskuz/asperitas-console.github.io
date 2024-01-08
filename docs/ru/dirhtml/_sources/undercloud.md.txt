# Описание узла развёртывания 

Основной пользователь, от имени которого выполняется управление всеми системами узла развёртывания - **stack**. 
Подключитесь к узлу развёртывания по SSH. 
Далее все команды и действия будут указываться с узла развёртывания от имени пользователя **stack**
~~~shell
ssh stack@$ip
~~~

## TripleO

## Сеть

Введите в консоли команду 
~~~shell
sudo ovs-vsctl show 
~~~

Результат выполнения будет выглядеть следующим образом 
~~~
59a6160b-9679-46ba-a583-63a64d0c7f4d
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-ctlplane
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port "enp1s0"
            Interface "enp1s0"
        Port phy-br-ctlplane
            Interface phy-br-ctlplane
                type: patch
                options: {peer=int-br-ctlplane}
        Port br-ctlplane
            Interface br-ctlplane
                type: internal
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port int-br-ctlplane
            Interface int-br-ctlplane
                type: patch
                options: {peer=phy-br-ctlplane}
        Port "tapde8cbd01-25"
            tag: 1
            Interface "tapde8cbd01-25"
                type: internal
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "2.12.0"
~~~

Вывод указывает, что существует два связанных между собой бриджа OVS: `br-ctlplane` и `br-int`. 
Их также можно увидеть при выводе команды `ip a`. 

Бридж `br-int` не имеет в своём основании никакого физического интерфейса, зато имеет виртуальный интерфейс `tapde8cbd01-25`. 
Расскажем о нём позднее. 

Бридж `br-ctlplane` имеет физический интерфейс enp1s0 и связывает через него с внешней сетью бридж `br-int`.

При выводе `ip a` не указан никакой интерфейс tap. Попробуем выполнить следующее 
~~~
[stack@undercloud ~]$ sudo ip netns
qdhcp-a4d98b54-1bc4-402a-92c1-51cbeee94aa3 (id: 0)
~~~

Значит, что в системе существует выделенное сетевое пространство имён. Подробнее про них можно почитать [здесь](https://habr.com/ru/articles/549414/)
~~~
[stack@undercloud ~]$ sudo ip netns exec qdhcp-a4d98b54-1bc4-402a-92c1-51cbeee94aa3 ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
8: tapde8cbd01-25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:1e:c1:8b brd ff:ff:ff:ff:ff:ff
    inet 192.168.24.36/24 brd 192.168.24.255 scope global tapde8cbd01-25
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe1e:c18b/64 scope link 
       valid_lft forever preferred_lft forever
~~~

Мы видим знакомый нам tap интерфейс с адресом 192.168.24.36/24. Назначение этого адреса становится понятно, если поискать этот порт среди портов openstack
~~~
(undercloud) [stack@undercloud ~]$ openstack port list | grep 192.168.24.36
| de8cbd01-25f3-4ebd-8965-74c98eca45bd |      | fa:16:3e:1e:c1:8b | ip_address='192.168.24.36', subnet_id='ab41b2bc-f27c-43a6-977e-4bd93d0c1cc7' | ACTIVE |
(undercloud) [stack@undercloud ~]$ openstack port show de8cbd01-25f3-4ebd-8965-74c98eca45bd -c binding_vif_details -c binding_vif_type -c device_owner
+---------------------+-------------------------------------------------------------------------------------------------------------+
| Field               | Value                                                                                                       |
+---------------------+-------------------------------------------------------------------------------------------------------------+
| binding_vif_details | bridge_name='br-int', connectivity='l2', datapath_type='system', ovs_hybrid_plug='True', port_filter='True' |
| binding_vif_type    | ovs                                                                                                         |
| device_owner        | network:dhcp                                                                                                |
+---------------------+-------------------------------------------------------------------------------------------------------------+
~~~

Мы видим, что этот порт является IP адресом DHCP сервера в сети ctlplane, поднятый на узле развёртывания в отдельном сетевом пространстве имён. 

Если использовать имя сетевого пространства имён и выполнить 
~~~
[stack@undercloud ~]$ sudo podman ps | grep  qdhcp-a4d98b54-1bc4-402a-92c1-51cbeee94aa3

~~~

То вы увидите контейнер, который отвечает за DHCP сервис в сети Ctlplane. 
