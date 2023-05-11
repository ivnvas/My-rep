## **Установка и настройка PostgreSQL**
* создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере</br>
Создал ВМ с Ubuntu на Virtual Box
* поставьте на нее PostgreSQL 15 через sudo apt<br>
Установил PostgreSQL 15
* проверьте что кластер запущен через sudo -u postgres pg_lsclusters<br>
Кроме 15 версии была у меня еще установлена 14 версия.
![Inst](scrin_6/claster.png) 
* зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
postgres=# create table test(c1 text);
postgres=# insert into test values('1');
\q
![Inst](scrin_6/tab_test.png) 
* остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
![Inst](scrin_6/stop_claster.png) 
* создайте новый диск к ВМ размером 10GB</br>
Диск создан
* добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
* проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux </br>
Выполнил согласно инструкции
* перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
![Inst](scrin_6/disc.png) 
* сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/ </br>
Сделал
* перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
![Inst](scrin_6/stop_claster.png) 
* попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
![Inst](scrin_6/mv.png) 
* напишите получилось или нет и почему</br>
Запуск не удался, так как в системном файле указан путь к data_directory, которого нет (мы перенесли на другое место)
* задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его
data_directory = 'mnt/data/15/main'  
![Inst](scrin_6/conf.png) 
* напишите что и почему поменяли</br>
Поменял путь к данным, которые мы перенесли.
* попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
![Inst](scrin_6/pusk15.png) 
* напишите получилось или нет и почему<br>
Получилось, так как указан корректный путь в файле postgresql.conf
* зайдите через через psql и проверьте содержимое ранее созданной таблицы
![Inst](scrin_6/tab_test_povtor.png) 
