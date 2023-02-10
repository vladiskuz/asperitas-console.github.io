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

