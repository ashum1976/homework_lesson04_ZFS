
#                                                               1. Работа с ZFS

#                                           1.1 Настройка запущенного стенда (Vikentsi Lapa - https://github.com/nixuser/virtlab/tree/main/zfs)




* Автодополнение в в bash для zfs

mkdir -p /usr/local/share/bash-completion/completions && cd /usr/local/share/bash-completion/completions
curl -O https://raw.githubusercontent.com/openzfs/zfs/master/contrib/bash_completion.d/zfs
chmod +x /usr/local/share/bash-completion/completions/zfs
source /usr/local/share/bash-completion/completions/zfs <-----  активировали автодополнения zfs


#                                           1.2 Создание, управление, свойства пула zfs 


  Создаём простой пул из 3-х доступных дисков по 1G
        
1. Проверяем какие есть диски
        
            [root@server completions]# lsblk 
            
            NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
            sda      8:0    0  10G  0 disk 
            └─sda1   8:1    0  10G  0 part /
            sdb      8:16   0   1G  0 disk 
            sdc      8:32   0   1G  0 disk 
            sdd      8:48   0   1G  0 disk 
            sde      8:64   0   1G  0 disk 
            sdf      8:80   0   1G  0 disk 
            sdg      8:96   0   1G  0 disk

2. Из свободных 3-х дисков создадим простой пул raidz1:

            [root@server completions]# zpool create  hwpool raidz1 /dev/sd{b,c,d}
            
            root@server completions]# lsblk 
            
            NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
            sda      8:0    0   10G  0 disk 
            └─sda1   8:1    0   10G  0 part /
            sdb      8:16   0    1G  0 disk 
            ├─sdb1   8:17   0 1014M  0 part 
            └─sdb9   8:25   0    8M  0 part 
            sdc      8:32   0    1G  0 disk 
            ├─sdc1   8:33   0 1014M  0 part 
            └─sdc9   8:41   0    8M  0 part 
            sdd      8:48   0    1G  0 disk 
            ├─sdd1   8:49   0 1014M  0 part 
            └─sdd9   8:57   0    8M  0 part 
            sde      8:64   0    1G  0 disk 
            sdf      8:80   0    1G  0 disk 
            sdg      8:96   0    1G  0 disk
            
            
    Проверяем средствами zfs, успешно ли создан пул:

            [root@server completions]# zpool list 
            
            NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
            hwpool  2.81G   108K  2.81G        -         -     0%     0%  1.00x    ONLINE  -
            
    
#                                           1.3 Создание, управление, свойства файловой системы zfs



Файловые системы ZFS представляют собой центральную точку администрирования. Наиболее эффективной моделью является использование одной файловой системы для каждого пользователя или проекта, поскольку это обеспечивает возможность управления свойствами, снимками и резервным копированием для конкретного пользователя или проекта.

*           Монтирование файловых систем ZFS

            Технология ZFS разработана в целях снижения сложности систем и упрощения администрирования. Например, существующие файловые системы требуют редактирования файла /etc/fstab при каждом добавлении новой файловой системы. Для ZFS эта потребность не актуальна вследствие автоматического монтирования и размонтирования файловых систем в соответствии со свойствами набора данных. Нет необходимости управлять записями ZFS в файле /etc/fstab.

ZFS позволяет организовать файловые системы в иерархии для группирования схожих систем. Эта модель обеспечивает центральную точку администрирования для
управления свойствами и администрирования файловых систем. Аналогичные файловые системы должны создаваться под общим именем.
При создании ФС, если не указана точка монтирования, то используется та, что задана в и

            hwpool/home/user1  <--- файловые системы для разнх пользователей, но с одинаковыми параметрами, можно сгруппирвать под одной ФС (home)
                        |-->/user2
                        |-->/user3
           
-    Пример создания основной  ФС с заданными параметрами и дочерних с отдельными параметрами
    
            [root@server completions]# zfs create hwpool/home -o quota=500M

            [root@server completions]# zfs list  <------- Просмотр информации о доступных файловых системах с помощью команды zfs list.

            
            NAME          USED  AVAIL     REFER  MOUNTPOINT
            hwpool        129K  2.69G     25.5K  /hwpool
            hwpool/home    24K   500M       24K  /hwpool/home
    
            [root@server completions]# zfs create hwpool/home/user1   <------- Создаём ФС выделенные под каждого пользователя, например
            [root@server completions]# zfs create hwpool/home/user2
            [root@server completions]# zfs create hwpool/home/user3
            
            [root@server completions]# zfs list 
            
            NAME                USED  AVAIL     REFER  MOUNTPOINT
            hwpool              214K  2.69G     25.5K  /hwpool
            hwpool/home          98K   500M       26K  /hwpool/home
            hwpool/home/user1    24K   500M       24K  /hwpool/home/user1
            hwpool/home/user2    24K   500M       24K  /hwpool/home/user2
            hwpool/home/user3    24K   500M       24K  /hwpool/home/user3
 
            
Дочернии ФС ( user1, user2, user3 ) будут наследовать параметры родительской ФС ( home ), можно задать общие параметры для всех, но и можно задавать отдельные параметры для выбранной ФС ( zfs set quota=200M  hwpool/home/user1 - установлена квота для ФС user1  в размере 200М ) :


            root@server completions]# zfs set quota=200M hwpool/home/user1    <------- установлена квота для ФС user1  в размере 200М
            
            [root@server completions]# zfs list 
            
            NAME                USED  AVAIL     REFER  MOUNTPOINT
            hwpool              216K  2.69G     25.5K  /hwpool
            hwpool/home        98.5K   500M     26.5K  /hwpool/home
            hwpool/home/user1    24K   200M       24K  /hwpool/home/user1
            hwpool/home/user2    24K   500M       24K  /hwpool/home/user2
            hwpool/home/user3    24K   500M       24K  /hwpool/home/user3

Параметры можно как задавать при создании ФС, так и  переопределять в процессе работы.

Получить всё параметры файловой системы, можно с помощью команды get all:

            [root@server completions]# zfs get all hwpool/home/user1
            
            
            NAME               PROPERTY              VALUE                  SOURCE
            hwpool/home/user1  type                  filesystem             -
            hwpool/home/user1  creation              Tue Feb  9 20:29 2021  -
            hwpool/home/user1  used                  24K                    -
            hwpool/home/user1  available             200M                   -
            hwpool/home/user1  referenced            24K                    -
            hwpool/home/user1  compressratio         1.00x                  -
            hwpool/home/user1  mounted               yes                    -
            hwpool/home/user1  quota                 200M                   local
            hwpool/home/user1  reservation           none                   default
            hwpool/home/user1  recordsize            128K                   default
            hwpool/home/user1  mountpoint            /hwpool/home/user1     default
            hwpool/home/user1  sharenfs              off                    default
            hwpool/home/user1  checksum              on                     default
            hwpool/home/user1  compression           off                    default  <------ значение параметра по умолчаню
            hwpool/home/user1  atime                 on                     default
            hwpool/home/user1  devices               on                     default
            hwpool/home/user1  exec                  on                     default
            hwpool/home/user1  setuid                on                     default
            hwpool/home/user1  readonly              off                    default
            hwpool/home/user1  zoned                 off                    default
            hwpool/home/user1  snapdir               hidden                 default
            hwpool/home/user1  aclinherit            restricted             default
            hwpool/home/user1  createtxg             309                    -
            hwpool/home/user1  canmount              on                     default
            hwpool/home/user1  xattr                 on                     default
            hwpool/home/user1  copies                1                      default
            hwpool/home/user1  version               5                      -
            hwpool/home/user1  utf8only              off                    -
            hwpool/home/user1  normalization         none                   -
            hwpool/home/user1  casesensitivity       sensitive              -
            hwpool/home/user1  vscan                 off                    default
            hwpool/home/user1  nbmand                off                    default
            hwpool/home/user1  sharesmb              off                    default
            hwpool/home/user1  refquota              none                   default
            hwpool/home/user1  refreservation        none                   default
            hwpool/home/user1  guid                  10355423699829523799   -
            hwpool/home/user1  primarycache          all                    default
            hwpool/home/user1  secondarycache        all                    default
            hwpool/home/user1  usedbysnapshots       0B                     -
            hwpool/home/user1  usedbydataset         24K                    -
            hwpool/home/user1  usedbychildren        0B                     -
            hwpool/home/user1  usedbyrefreservation  0B                     -
            hwpool/home/user1  logbias               latency                default
            hwpool/home/user1  objsetid              96                     -
            hwpool/home/user1  dedup                 off                    default
            hwpool/home/user1  mlslabel              none                   default
            hwpool/home/user1  sync                  standard               default
            hwpool/home/user1  dnodesize             legacy                 default
            hwpool/home/user1  refcompressratio      1.00x                  -
            hwpool/home/user1  written               24K                    -
            hwpool/home/user1  logicalused           12K                    -
            hwpool/home/user1  logicalreferenced     12K                    -
            hwpool/home/user1  volmode               default                default
            hwpool/home/user1  filesystem_limit      none                   default
            hwpool/home/user1  snapshot_limit        none                   default
            hwpool/home/user1  filesystem_count      none                   default
            hwpool/home/user1  snapshot_count        none                   default
            hwpool/home/user1  snapdev               hidden                 default
            hwpool/home/user1  acltype               off                    default
            hwpool/home/user1  context               none                   default
            hwpool/home/user1  fscontext             none                   default
            hwpool/home/user1  defcontext            none                   default
            hwpool/home/user1  rootcontext           none                   default
            hwpool/home/user1  relatime              off                    default
            hwpool/home/user1  redundant_metadata    all                    default
            hwpool/home/user1  overlay               off                    default
            hwpool/home/user1  encryption            off                    default
            hwpool/home/user1  keylocation           none                   default
            hwpool/home/user1  keyformat             none                   default
            hwpool/home/user1  pbkdf2iters           0                      default
            hwpool/home/user1  special_small_blocks  0                      default

Отдельные значения параметров ФС, можно узнать с помощью команды get "имя параметра" :

            [root@server completions]# zfs get compression hwpool/home/user1
            
            NAME               PROPERTY     VALUE     SOURCE
            hwpool/home/user1  compression  off       default
            
            [root@server completions]# zfs get compressratio hwpool/home/user1
            
            NAME               PROPERTY       VALUE  SOURCE
            hwpool/home/user1  compressratio  1.00x  -

#                                                               2. Домашнее задание
#                                           2.1 Установка параметров сжатия ФС

 Включение или выключение сжатия для ФС определяется параметром "compression=on/off"  Доступные значения: on , off, lzjb, gzip и gzip-N, zle, lz4.  По умолчанию установлено значение off. Активация сжатия в файловой системе с существующими данными приводит к сжатию только новых данных. Существующие данные не сжимаются.

            compression     YES      YES   on | off | lzjb | gzip | gzip-[1-9] | zle | lz4 <----- варианты сжатия
            
            [root@server completions]# zfs set compression=lzjb hwpool/home  <----- установили значение сжатия lzjb, которое переопределим в дочерних ФС ( user 1,2,3), user4 на значении от родительской ФС
            
            [root@server completions]# zfs set compression=gzip-9 hwpool/home/user1
            [root@server completions]# zfs set compression=lz4 hwpool/home/user2
            [root@server completions]# zfs set compression=zle hwpool/home/user3
            [root@server completions]# zfs create hwpool/home/user4
            
Был скопирован файл " 3,2M war-and-peace.txt " в разные ФС с раными параметрами сжатия.

            [root@server ~]# df -Th /hwpool/home/user{1,2,3,4}
            
            Filesystem        Type  Size  Used Avail Use% Mounted on
            hwpool/home/user1 zfs   200M  1.3M  199M   1% /hwpool/home/user1   <----- gzip-9 наибольшее сжатие
            hwpool/home/user2 zfs   494M  2.0M  492M   1% /hwpool/home/user2   <----- lz4
            hwpool/home/user3 zfs   495M  3.3M  492M   1% /hwpool/home/user3   <----- zle вообще нет как такового сжатия
            hwpool/home/user4 zfs   494M  2.5M  492M   1% /hwpool/home/user4   <----- lzjb
            [root@server ~]# zfs get compressratio hwpool/home/user{1,2,3,4}
            NAME               PROPERTY       VALUE  SOURCE
            hwpool/home/user1  compressratio  2.70x  - <----- gzip-9 наибольшее сжатие
            hwpool/home/user2  compressratio  1.64x  - <----- lz4
            hwpool/home/user3  compressratio  1.03x  - zle вообще нет как такового сжатия
            hwpool/home/user4  compressratio  1.37x  - lzjb
            
#                                           2.2 Перенос пула устройств хранения данных с одного компьютера на другой

В некоторых случаях может потребоваться перенос пула устройств хранения данных с одного компьютера на другой. При этом устройства хранения требуется отключить от исходного компьютера и подключить к целевому компьютеру. . ZFS позволяет экспортировать пул из одного компьютера и импортировать его в целевой компьютер, даже если они имеют различный порядок следования байтов (endianness).
Пулы устройств хранения данных должны быть явно экспортированы, что будет указывать на их готовность к переходу. Эта операция приводит к загрузке на диск всех незаписанных данных, записи данных на диск с указанием о том, что был выполнен экспорт, и удалению всех данных пула из системы.

*       Экспорт пула:

            [root@server ~]# zpool list 
            NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT 
            hwpool  2.75G  13.5M  2.74G        -         -     0%     0%  1.00x    ONLINE  -                                    <---------- наш созданный  pool 
            
            [root@server ~]# zpool export hwpool                         <-------- Экспортируем pool, он перестаёт быть доступен 
            
            [root@server ~]# zpool list 
            no pools available
            
При наличии устройств в другом каталоге или файловых пулов для их поиска следует использовать параметр -d. Пример:

1.   Для теста, создадим виртуальные файлы(диски):

            [root@server ~]# for i in {1..3}; do truncate -s 2G /zfs_test/$i.img; done  <----- создадим виртуальные файлы (диски) 
            [root@server ~]# ls -la /zfs_test/
            total 0                                                                                                                                                                                             
            drwxr-xr-x.  2 root root         45 Feb 10 20:43 .                                                                                                                                                  
            dr-xr-xr-x. 20 root root        285 Feb 10 20:34 ..                                                                                                                                                 
            -rw-r--r--.  1 root root 2147483648 Feb 10 20:43 1.img
            -rw-r--r--.  1 root root 2147483648 Feb 10 20:43 2.img                                                                                                                                              
            -rw-r--r--.  1 root root 2147483648 Feb 10 20:43 3.img                                                                                                                                              

2.  Создаём тестовый пул "test"

            [root@server ~]# zpool create test /zfs_test/{1,2,3}.img                                                                                                                                            
            [root@server ~]# zpool status
            [root@server ~]# zpool status                                                                                                                                                                        
            pool: hwpool                                                                                                                                                                                       
            state: ONLINE                                                                                                                                                                                       
            scan: none requested                                                                                                                                                                               
            config:                                                                                                                                                                                              
                                                                                                                                                                                                                
                    NAME        STATE     READ WRITE CKSUM                                                                                                                                                       
                    hwpool      ONLINE       0     0     0                                                                                                                                                       
                    raidz1-0  ONLINE       0     0     0                                                                                                                                                       
                        sdb     ONLINE       0     0     0                                                                                                                                                       
                        sdc     ONLINE       0     0     0                                                                                                                                                       
                        sdd     ONLINE       0     0     0                                                                                                                                                       
                                                                                                                                                                                                                
            errors: No known data errors                                                                                                                                                                         
                                                                                                                                                                                                                
            pool: test                                    <------------ Тестовый пул, на нём делаем экспорт, и переносим диски {1.2.3}.img на другую машину.                                                                                                                                         
            state: ONLINE                                                                                                                                                                                       
            scan: none requested                                                                                                                                                                               
            config:                                                                                                                                                                                              
                                                                                                                                                                                                                
                    NAME               STATE     READ WRITE CKSUM                                                                                                                                                
                    test               ONLINE       0     0     0                                                                                                                                                
                    /zfs_test/1.img  ONLINE       0     0     0                                                                                                                                                
                    /zfs_test/2.img  ONLINE       0     0     0                                                                                                                                                
                    /zfs_test/3.img  ONLINE       0     0     0

                    
             [root@server ~]# zpool export test  <------ Экспортируем пул        
                    
                    
            
3.      Для получения списка доступных пулов используется команда zpool import без каких-либо параметров, но с указанием каталога где смотреть наш пул. Пример:

            [root@server ~]# zpool import  -d /zfs_test/
            pool: test
                id: 12513217291922710169
            state: ONLINE
            action: The pool can be imported using its name or numeric identifier.
            config:

                    test               ONLINE
                    /zfs_test/1.img  ONLINE
                    /zfs_test/2.img  ONLINE
                    /zfs_test/3.img  ONLINE
            
            
            

#                                           2.3 Импорт пула            

#                                           Для домашнего задания:   
           
           
            [root@server ~]# tar -xzf zfs_task1.tar.gz -C zfs_pool/   <----- Разархивируем полученный файл с экспортированным пулом
                        
           
            [root@server ~]# zpool import -d /root/zfs_pool/zpoolexport/   <----- Для просмотра доступных пулов, берём директорию, где лежит экспортированный пул (указываем именно директорию)
                pool: otus
                id: 6554193320433390805
                state: ONLINE
                action: The pool can be imported using its name or numeric identifier.
                config:

                otus                                  ONLINE
                mirror-0                            ONLINE
                /root/zfs_pool/zpoolexport/filea  ONLINE
                /root/zfs_pool/zpoolexport/fileb  ONLINE

           
           
                [root@server ~]# zpool import -d /root/zfs_pool/zpoolexport/ 6554193320433390805         <------- Импортируем по  id: 6554193320433390805
                
                
                [root@server ~]# zpool list
                
                NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
                hwpool  2.75G  13.6M  2.74G        -         -     0%     0%  1.00x    ONLINE  -
                otus     480M  2.09M   478M        -         -     0%     0%  1.00x    ONLINE  -
            
           
           
           
           
           
#                                           2.3 Получение/отправка снимков файловой системы
            
            [root@server ~]# zfs receive hwpool/otus@task2 < ./otus_task2.file  <---- Полученный снимок 
            
            
            [root@server ~]# zfs list 
            NAME                USED  AVAIL     REFER  MOUNTPOINT
            hwpool             12.7M  1.69G     33.3K  /hwpool
            hwpool/home        8.84M   491M     36.0K  /hwpool/home
            hwpool/home/user1  1.23M   199M     1.23M  /hwpool/home/user1
            hwpool/home/user2  2.01M   491M     2.01M  /hwpool/home/user2
            hwpool/home/user3  3.17M   491M     3.17M  /hwpool/home/user3
            hwpool/home/user4  2.39M   491M     2.39M  /hwpool/home/user4
            hwpool/otus        3.72M  1.69G     3.72M  /hwpool/otus
            [root@server ~]# zfs list  -t snapshot                              <-------- Смотрим наличие снапшотов (снимков ФС)
            NAME                USED  AVAIL     REFER  MOUNTPOINT
            hwpool/otus@task2     0B      -     3.72M  -

            
            [root@server file_mess]# zfs rollback hwpool/otus@task2         <------ Делаем откат ФС на имеющийся снапшот 
            
            [root@server file_mess]# cat secret_message   <------ Смотрим сообщение
            https://github.com/sindresorhus/awesome
            

            
           
                       
                     
               
