# Устранение неполадок

## Физический узел в статусе _enroll_

Если после добавления физического узла его статус изменился на _enroll_, то возможны следующие варианты:
* Указанный IP-адрес недоступен с узла развёртывания. Проверьте доступность адреса с узла развёртывания: надо выйти из asperitas консоли и попробовать команду
~~~shell
ping -c 1 2.2.2.3
~~~
* Логин/пароль от BMC неверные 

Чтобы изменить параметры узла, можно удалить узел и создать его заново или изменить параметры текущего узла. 
Для этого выберите узел в списке физических узлов и нажмите _Enter_ или дважды кликните мышкой:

![](../../images/baremetal-node-edit.png)

Измените необходимые параметры, а также **снова введите пароль**. Нажмите кнопку _Save_ и ожидайте изменение статуса на manageable.

## Ошибка настройка сети 

Если при развёртывании облака вы видете ошибку,
~~~
    "failed_when_result": true
}
2023-03-02 17:02:49.089226 | 08002710-c16f-44d5-9117-00000000029d |     TIMING | NetworkConfig stdout | monitoring-ceph-0 | 0:05:58.946554 | 0.06s
~~~
зайдите на упавший хост по ssh и выполните 
~~~shell
sudo cat /etc/os-net-config/config.json
~~~
В выводе будет сгенерённый для вашего узла сетевой план. 
Проверьте, что план соответствует ожидаемым настройкам .
Если план не соответствует, то перейдите к настройке сетевого плана снова и запустите обновление развёртывания. 

Если план соответствует ожидаемому, то выполните 
~~~shell
sudo os-net-config -c /etc/os-net-config/config.json --verbose
~~~

И посмотрите на вывод команды и ошибки, которые команда выведет. 
Дальнейшая работа с проблемой зависит от ошибок в выводе команды.

## Wait for puppet host configuration to finish

Если при развёртывании сервисов Ansible падает на задаче _Wait for puppet host configuration to finish_, то сначала необходимо определить узел, на котором произошла ошибка 

~~~
YYYY-MM-DD HH:MM:SS | 00000000-0000-0000-0000-000000000000 | WAITING | Wait for puppet host configuration to finish | asperitas-controller-0
~~~

В примере выше ошибка произошла на узле _asperitas-controller-0_. 
Для поиска причины ошибки необходимо зайти по ssh на этот узел и запустить puppet самостоятельно 
~~~shell
sudo puppet apply --modulepath=/etc/puppet/modules:/opt/stack/puppet-modules:/usr/share/openstack-puppet/modules --detailed-exitcodes --summarize --color=true --verbose /var/lib/tripleo-config/puppet_step_config.pp 
~~~

Дальнейший поиск проблемы зависит от вывода команды. Вывод команды может быть долгим, поэтому дождитесь, пожалуйста, пока команда не завершится. 

### Pacemaker

Ошибка выглядит следующим образом:
~~~
Error: 'pcs status | grep -q -e "asperitas-novacomputeiha-30[[:blank:]].*Started"' returned 1 instead of one of [0]
~~~

Это значит, что pacemaker не смог запустить ресурс _asperitas-novacomputeiha-30_. 

С любого из контроллеров выполните:
~~~shell
sudo pcs status
~~~

Вы увидите статус _FAILED_ для ресурса _asperitas-novacomputeiha-30_. Проверьте правильность его настройки:
~~~shell
sudo pcs resource config asperitas-novacomputeiha-30
~~~

В выводе будет указан адрес ресурса в поле _Attributes_. 
1. Проверьте доступность адреса с узла контроллера, на котором вы сейчас находитесь.
~~~shell
ping -c 1 <ip>
~~~
2. Проверьте, что на этот адрес ходят большие пакеты
~~~shell
ping -c 1 -s 10000 <ip>
~~~
3. Проверьте, что доступен порт 3121. Запрос не должен виснуть и не должен выводить ошибку _Failed to connect_
~~~shell 
curl <ip>:3121 
~~~
Если порт не доступен, то зайдите на проблемный узел по ssh и проверьте статус сервиса _pacemaker_remote_
~~~shell
sudo systemctl status pacemaker_remote
~~~
Этот сервис должен быть активен только на узлах виртуализации. На узлах контроллерах должен быть активен сервис _pacemaker_.

Если сервис активен, но имеет ошибки вида 
~~~
error: TLS handshake with remote client failed: An illegal parameter has been received.
~~~
то скопируйте файл _/etc/pacemaker/auth_ с рабочего узла. Команды с узла развёртывания 
~~~
ssh heat-admin@<working-host>.ctlplane 'sudo cp /etc/pacemaker/auth ./'
rsync -azP heat-admin@<working-host>.ctlplane:authkey ./
rsync -azP authkey heat-admin@<failed-host>.ctlplane:
ssh heat-admin@<failed-host>.ctlplane 'sudo cp authkey /etc/pacemaker/'
~~~

После решения проблем зайдите на любой из контроллеров и выполните 
~~~shell
sudo pcs resouce refresh <failed-host>
~~~
В указанном выше примере _failed-host_ == _asperitas-novacomputeiha-30_.

Запустите повторно puppet на узле, на котором puppet упал изначально. Если puppet выполнит работу без ошибок, то продолжите работу Ansible далее без повторного выполнения задачи Ansible.

## Не найден хост 

### После обновления развёртывания 

1. Переведите nova-scheduler в режим debug. Для этого поменяйте параметр _debug=False_ в _True_ в файле _/var/lib/config-data/puppet-generated/nova/etc/nova/nova.conf_. Затем перезапустите nova-scheduler
~~~shell
sudo systemctl restart tripleo_nova_scheduler
~~~
2. Попробуйте создать виртуальную машину и проверьте в логах ошибку 
~~~shell
less /var/log/containers/nova/nova-scheduler.log
~~~
Если ошибка выглядит следующим образом, 
~~~
Filter AllHostsFilter returned 0 host(s)
~~~
то зайдите на любой контроллер (только один!) и выполните
~~~shell 
sudo podman exec -ti -u root nova_compute nova-manage placement heal_allocations
sudo podman exec -ti -u root nova_compute nova-manage placement sync_aggregates
~~~

## Число используемых ресурсов на гипервизорах обнулилось 

Такое может произойти, если вы попытались поменять основной домен для имен хостов. В таком случае вы увидете следующее.

Например, вы поменяли хостнейм с _localdomain_ на _novalocal_, тогда 
~~~shell 
openstack server list --all-projects --host asperitas-novacomputeiha-12.localdomain -f value | wc -l
20

openstack hypervisor show asperitas-novacomputeiha-12.novalocal -c service_host -c running_vms

+--------------+-----------------------------------------+
| Field        | Value                                   |
+--------------+-----------------------------------------+
| running_vms  | 0                                       |
| service_host | asperitas-novacomputeiha-12.localdomain |
+--------------+-----------------------------------------+
~~~

Причина проблемы в том, что у виртуалок запущенных на хосте _asperitas-novacomputeiha-12.localdomain_ есть параметр гипервизора, который остался в значении _asperitas-novacomputeiha-12.localdomain_. 

Чтобы решить проблему, необходимо на любом из контроллеров зайти в базу данных Mariadb и поменять значение для виртуалки 
~~~shell
sudo podman ps | grep galera 

ab09fec32c4b  undercloud.ctlplane:13787/image/docker/images/ispras/minos/openstack-mariadb:pcmklatest                 /bin/bash /usr/lo...  9 days ago    Up 8 days ago                                       galera-bundle-podman-0

sudo podman exec -ti -u root galera-bundle-podman-0 mysql
> use nova_api
> select display_name from instances where node='asperitas-novacomputeiha-12.localdomain' and deleted_at is null;
> begin;
> update instances set node='asperitas-novacomputeiha-12.novalocal' where node='asperitas-novacomputeiha-12.localdomain' and deleted_at is null;
> commit ;
~~~
