# Сопровождение облака и управление ресурсами


## Флейворы 

Для создания флейворов используется команда
~~~shell
openstack flavor create cpu<cpu_num>.ram<ram_gb>.disk<disk_gb> \
    --vcpus <cpu_num> \
    --ram <ram_mb> \
    --disk <disk_gb>
~~~

По умолчанию флейворы создаются публичные. Необходимо явно ставить флаг `--private`, 
чтобы можно было управлять доступом к флейвору.  

Чтобы вместе запустить виртуальную машину с ГПУ или другим PCI устройством - 
настройте его использование на вычислительном узле в nova.conf и указанный `alias` 
используйте для создания флейвора  
~~~shell
openstack flavor create --private \
    gpu-<gpu_alias>-x<gpu_num>.cpu<cpu_num>.ram<ram_gb>.disk<disk_gb> \
    --property pci_passthrough:alias='<gpu_alias>:<gpu_num>'
~~~

Чтобы поделить узлы по признаку SSD/HDD или с/без ГПУ - существует механизм **trait**'ов.

Посмотреть их можно для конкретного узла через Placement API
~~~shell
openstack resource provider trait list <resource_provider_uuid>\
~~~

Полный список можно посмотреть следующим образом 
~~~shell
openstack trait list --sort-column name
~~~

Из интересных существующих trait'ов есть `STORAGE_DISK_HDD`/`STORAGE_DISK_SSD` - 
но они не назначаются узлам автоматически. Для этого используется команда 

~~~shell 
openstack resource provider trait set --trait <trait_name> <resource_provider_uuid>
~~~

Чтобы использовать свой trait, можно использовать команды 
~~~shell
openstack trait create CUSTOM_HAS_GPU
openstack resource provider trait set --trait CUSTOM_HAS_GPU <resource_provider_with_gpu>
~~~

Далее запрещаем флейвору без ГПУ занимать узлы с ГПУ
~~~shell
openstack flavor set cpu1.ram1.hdd10 --property trait:CUSTOM_HAS_GPU=forbidden
~~~
Или наоборот 
~~~shell
openstack flavor set cpu1.ram1.hdd10 --property trait:STORAGE_DISK_HDD=required
~~~
