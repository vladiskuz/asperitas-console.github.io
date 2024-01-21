# Описание узла развёртывания 

Основной пользователь, от имени которого выполняется управление всеми системами узла развёртывания - **stack**. 
Подключитесь к узлу развёртывания по SSH. 
Далее все команды и действия будут указываться с узла развёртывания от имени пользователя **stack**
~~~shell
ssh stack@$ip
~~~

## TripleO

При успешной установке узла развёртывания на нём будет развёрнут полноценный **однонодовый OpenStack**. 
В том числе будут установлены дополнительные пакеты TripleO и overcloud CLI. 

В домашней директории /home/stack находится файл **stackrc**.
~~~
[stack@undercloud ~]$ cat stackrc 
# Clear any old environment that may conflict.
for key in $( set | awk -F= '/^OS_/ {print $1}' ); do unset "${key}" ; done

export OS_AUTH_TYPE=password
export OS_PASSWORD=*****
export OS_AUTH_URL=https://192.168.24.2:13000
export OS_USERNAME=admin
export OS_PROJECT_NAME=admin
export COMPUTE_API_VERSION=1.1
export NOVA_VERSION=1.1
export OS_NO_CACHE=True
export OS_CLOUDNAME=undercloud
export OS_IDENTITY_API_VERSION='3'
export OS_PROJECT_DOMAIN_NAME='Default'
export OS_USER_DOMAIN_NAME='Default'
export OS_CACERT="/etc/pki/ca-trust/source/anchors/cm-local-ca.pem"
# Add OS_CLOUDNAME to PS1
if [ -z "${CLOUDPROMPT_ENABLED:-}" ]; then
    export PS1=${PS1:-""}
    export PS1=\${OS_CLOUDNAME:+"(\$OS_CLOUDNAME)"}\ $PS1
    export CLOUDPROMPT_ENABLED=1
fi
~~~

Этот файл используется для обращения к OpenStack API узла развёртывания. 
Все интерфейсы этого API при этом доступны только в сети **Ctlplane**. 

~~~
[stack@undercloud ~]$ source stackrc 
(undercloud) [stack@undercloud ~]$ openstack endpoint list --interface public -c "Service Name" -c "Service Type" -c URL
+------------------+-------------------------+--------------------------------------------------+
| Service Name     | Service Type            | URL                                              |
+------------------+-------------------------+--------------------------------------------------+
| glance           | image                   | https://192.168.24.2:13292                       |
| zaqar-websocket  | messaging-websocket     | wss://192.168.24.2:9000                          |
| ironic           | baremetal               | https://192.168.24.2:13385                       |
| keystone         | identity                | https://192.168.24.2:13000                       |
| swift            | object-store            | https://192.168.24.2:13808/v1/AUTH_%(tenant_id)s |
| zaqar            | messaging               | https://192.168.24.2:13888                       |
| placement        | placement               | https://192.168.24.2:13778/placement             |
| heat             | orchestration           | https://192.168.24.2:13004/v1/%(tenant_id)s      |
| ironic-inspector | baremetal-introspection | https://192.168.24.2:13050                       |
| mistral          | workflowv2              | https://192.168.24.2:13989/v2                    |
| neutron          | network                 | https://192.168.24.2:13696                       |
| nova             | compute                 | https://192.168.24.2:13774/v2.1                  |
+------------------+-------------------------+--------------------------------------------------+
~~~

Среди созданных сетей `openstack network list` вы увидите сеть **ctlplane**.

**Все сервисы OpenStack запущены в контейнерах**. Для проверки этого выполните
~~~
(undercloud) [stack@undercloud ~]$ sudo podman ps
CONTAINER ID  IMAGE                                                           COMMAND      CREATED      STATUS                NAMES
91124d20d2d9  undercloud.ctlplane:13787/asperitos/openstack-memcached:latest  kolla_start  3 weeks ago  Up 3 weeks (healthy)  memcached
b7333c04c4c2  undercloud.ctlplane:13787/asperitos/openstack-haproxy:latest    kolla_start  3 weeks ago  Up 10 days            haproxy
4698f4d2425b  undercloud.ctlplane:13787/asperitos/openstack-rabbitmq:latest   kolla_start  3 weeks ago  Up 3 weeks (healthy)  rabbitmq
489fddc0f48f  undercloud.ctlplane:13787/asperitos/openstack-mariadb:latest    kolla_start  3 weeks ago  Up 3 weeks (healthy)  mysql
b399d357390f  undercloud.ctlplane:13787/asperitos/openstack-keystone:latest   kolla_start  3 weeks ago  Up 3 weeks (healthy)  keystone
69d2f12b398b  undercloud.ctlplane:13787/asperitos/openstack-iscsid:latest     kolla_start  3 weeks ago  Up 3 weeks (healthy)  iscsid
<...>
~~~

Каждому рабочему контейнеру также соответствует **сервис systemd**, который перезапускает контейнер при падении. 
Выполните: 
~~~
(undercloud) [stack@undercloud ~]$ sudo systemctl | grep tripleo_
  tripleo_glance_api.service                loaded active running   glance_api container
  tripleo_haproxy.service                   loaded active running   haproxy container
  tripleo_heat_api.service                  loaded active running   heat_api container
  tripleo_heat_api_cron.service             loaded active running   heat_api_cron container
  tripleo_heat_engine.service               loaded active running   heat_engine container
  tripleo_ironic_api.service                loaded active running   ironic_api container
  tripleo_ironic_conductor.service          loaded active running   ironic_conductor container
  tripleo_ironic_inspector.service          loaded active running   ironic_inspector container
  tripleo_ironic_inspector_dnsmasq.service  loaded active running   ironic_inspector_dnsmasq container
<...>
~~~

Поэтому для того, чтобы **остановить** какой-либо **контейнер**, используйте команду 
~~~shell
sudo systemctl stop tripleo_$service_name
~~~

Все **изменяемые файлы и директории** в сервисах как правило выносятся в хост систему из контейнера.

~~~shell
sudo podman inspect $container_name | jq .[0].Mounts
~~~

Поэтому при необходимости можно контейнеры удалять и **пересоздавать**, не боясь потерять данные. 
Для этого тоже можно использовать `podman inspect`, чтобы узнать команду, которой был создан контейнер.

~~~shell
sudo podman inspect $container_name | jq .[0].Config.CreateCommand
~~~

Обратите внимание, что все контейнеры _(кроме одного)_ запускаются с командой `kolla_start`. 
Эта команда использует файлы конфигурации _kolla_config_ для запуска процессов внутри контейнера. 
Например:

~~~
(undercloud) [stack@undercloud ~]$ sudo cat /var/lib/kolla/config_files/keystone.json 
{
    "command": "/usr/sbin/httpd",
    "config_files": [
        {
            "dest": "/etc/keystone/fernet-keys",
            "merge": false,
            "preserve_properties": true,
            "source": "/var/lib/kolla/config_files/src/etc/keystone/fernet-keys"
        },
        {
            "dest": "/etc/httpd/conf.d",
            "merge": false,
            "preserve_properties": true,
            "source": "/var/lib/kolla/config_files/src/etc/httpd/conf.d"
        },
        {
            "dest": "/",
            "merge": true,
            "preserve_properties": true,
            "source": "/var/lib/kolla/config_files/src/*"
        }
    ]
}
~~~

Рассмотрим подробнее: 
* `command` - команда, которая будет запущена в контейнере
* `config_files` - список файлов, которые будут скопированы в контейнер из хост системы. 
  * `dest` - путь внутри контейнера, куда будет скопирован файл
  * `merge` - если `true`, то файлы будут скопированы в директорию `dest`, если `false` - то весь каталог `source` будет скопирован в `dest`
  * `preserve_properties` - если `true`, то будут сохранены права доступа к файлу и владелец
  * `source` - путь внутри контейнера, который будет скопирован в `dest`. 

Отдельно хочу выделить путь `/var/lib/kolla/config_files/src/*`.  Для контейнера `keystone` - это директория `/var/lib/config-data/puppet-generated/keystone`. 
Для других контейнеров это путь `/var/lib/config-data/puppet-generated/$service_name`. 
Это директория, в которой хранятся все файлы конфигурации необходимые для сервиса. 
Например: 

~~~
(undercloud) [stack@undercloud ~]$ sudo tree /var/lib/config-data/puppet-generated/keystone
/var/lib/config-data/puppet-generated/keystone
|-- etc
|   |-- httpd
|   |   |-- conf
|   |   |   |-- httpd.conf
|   |   |   `-- ports.conf
|   |   `-- conf.d
|   |       |-- 10-keystone_wsgi.conf
<...>
|   |-- keystone
|   |   |-- credential-keys
|   |   |   |-- 0
|   |   |   `-- 1
|   |   |-- fernet-keys
|   |   |   |-- 0
|   |   |   `-- 1
|   |   `-- keystone.conf
|   `-- my.cnf.d
|       `-- tripleo.cnf
`-- var
    `-- www
        `-- cgi-bin
            `-- keystone
                `-- keystone
~~~

После запуска контейнера команда kolla_start копирует все вышеуказанные файлы в `/` системы контейнера и запускает команду `/usr/sbin/httpd`. 

При этом сами файлы в папке `/var/lib/config-data/puppet-generated/` генерируются **puppet** манифестами также запускаемыми в контейнерах.
Посмотреть на эти контейнеры можно через 
~~~shell 
sudo podman ps -a | grep container-puppet
~~~

Обратите внимание, что некоторые сервисы запускаются через папку `/var/lib/config-data/`, например, _barbican_, но это редкий случай.

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

Бридж `br-ctlplane` имеет физический интерфейс `enp1s0` и связывает через него с внешней сетью бридж `br-int`.

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
