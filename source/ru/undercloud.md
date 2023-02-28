# Установка узла развёртывания

Дистрибутив распространяется на переносимом цифровом носителе, на котором располагается файл undercloud.iso или установщик для образа узла развёртывания. 

* При установке узла на физический узел - вставьте флешку с установщиком в компьютер и загрузитесь в неё с физического узла. 
* При установке на физический узел через BMC по сети Интернет - примонтируйте файл _undercloud.iso_ через виртуальную консоль BMC и загрузитесь в Virtual CD узла. 
* При установке узла развёртывания в виртуальной машине следуйте инструкциям ниже 

## Установка в виртуальной машине (Centos 7,8) 

Установите необходимые пакеты на узел 
~~~shell
sudo dnf install -y bridge-utils
sudo dnf install -y qemu-kvm qemu-img virt-manager virt-viewer python3-libvirt libvirt libvirt-client libguestfs libguestfs-tools virt-install libgcrypt libxcrypt
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
virt-install --virt-type=kvm --name undercloud-base --ram 32768 --vcpus=16 --os-variant=centos8 --cdrom=/var/lib/libvirt/boot/undercloud.iso --network=bridge=br-eno1 --network=bridge=br-eno2 --disk path=/var/lib/libvirt/images/undercloud-base.qcow2,size=150,format=qcow2
~~~

KVM создаст виртуальные интерфейсы vnet0 и vnet1 в составе бриджей, если бридж падает, то интерфейсы из него исчезают. Чтобы добавить снова их в бридж можно выполнить
~~~shell
ip link set vnet1 master br-eno2
~~~

## Процесс установки 

После загрузки с диска установки вы увидите на экране сервера выбор диска для установки операционной системы AsperitOS. 

![](../../images/undercloud-1.png)
![](../../images/undercloud-2.png)

Дождитесь загрузки и проверки _undercloud.raw_. 

![](../../images/undercloud-3.png)
![](../../images/undercloud-4.png)
![](../../images/undercloud-5.png)
![](../../images/undercloud-6.png)
![](../../images/undercloud-7.png)
![](../../images/undercloud-8.png)
![](../../images/undercloud-9.png)
![](../../images/undercloud-10.png)
![](../../images/undercloud-11.png)
![](../../images/undercloud-12.png)
![](../../images/undercloud-13.png)
![](../../images/undercloud-14.png)
![](../../images/undercloud-15.png)
