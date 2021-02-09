
#                                                               Работа с ZFS

#                                           1.1 Настройка запущенного стенда (Vikentsi Lapa - https://github.com/nixuser/virtlab/tree/main/zfs)




* Автодополнение в в bash для zfs

mkdir -p /usr/local/share/bash-completion/completions && cd /usr/local/share/bash-completion/completions
curl -O https://raw.githubusercontent.com/openzfs/zfs/master/contrib/bash_completion.d/zfs
chmod +x /usr/local/share/bash-completion/completions/zfs
source /usr/local/share/bash-completion/completions/zfs <----- без перелогина активировали автодополнения zfs


#                                           1.2 Срздание, управление, свойства пула zfs 


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

            [root@server completions]# zpool create hwpool /dev/sd{b,c,d}
            
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

Получить всё параметры файловой системы, или отдельные значения, можно с помощью команды get:

-        [root@server completions]# zfs get all hwpool/home/user1
            
            
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
            hwpool/home/user1  compression           off                    default
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



 
            
 
