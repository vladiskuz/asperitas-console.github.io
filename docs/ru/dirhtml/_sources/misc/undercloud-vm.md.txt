# Установка узла развёртывания в виртуальной машине (Centos 7,8) 

Установите необходимые пакеты на узел 
~~~shell
sudo dnf install -y bridge-utils qemu-kvm qemu-img virt-manager virt-viewer python3-libvirt libvirt libvirt-client libguestfs libguestfs-tools virt-install libgcrypt libxcrypt
~~~

Для виртуальной машины узла развёртывания необходимо создать два бриджа и указать их при создании виртуальной машины KVM. 
Внутри ВМ они будут светиться как два интерфейса: один используется для внутренней сети настройки облачной платформы, а второй используется для доступа к узлу развёртывания.  

Создать мост для интерфейса, на котором планируется запустить KVM VM при помощи _network-scripts_
~~~shell
nano /etc/sysconfig/network-scripts/ifcfg-eno1
~~~
~~~
DEVICE=eno1
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
MTU=1500
BOOTPROTO=none
BRIDGE=br-eno1
TYPE=Ethernet
~~~
~~~shell
nano /etc/sysconfig/network-scripts/ifcfg-br-eno1
~~~
~~~
DEVICE=br-eno1
ONBOOT=yes
TYPE=Bridge
DELAY=0
BOOTPROTO=static
IPADDR=192.168.24.64
NETMASK=255.255.255.0
NM_CONTROLLED=no
DEFROUTE=no
~~~

Или создайте мост при помощи Linux утилиты _ip_
~~~shell
ip link add link eno1 name eno1.100 type vlan id 100
ip link set eno1.100 up
ip link add br100 type bridge 
ip link set eno1.100 master br100
ip link set br100 up
~~~

Для проверки настроек можно использовать 
~~~shell
sudo brctl show
~~~

Для запуска ВМ выполните
~~~shell
virt-install --virt-type=kvm --name undercloud-base --ram 32768 --vcpus=16 --os-variant=centos8 --cdrom=/var/lib/libvirt/boot/undercloud.iso --network=bridge=br-eno1 --network=bridge=br-eno2 --disk path=/var/lib/libvirt../../../images/undercloud-base.qcow2,size=150,format=qcow2
~~~

KVM создаст виртуальные интерфейсы vnet0 и vnet1 в составе бриджей, если бридж падает, то интерфейсы из него исчезают. Чтобы добавить снова их в бридж можно выполнить
~~~shell
ip link set vnet1 master br-eno2
~~~
