## **Описание/Пошаговая инструкция выполнения домашнего задания:**
создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докере</br>
далее создать инстанс виртуальной машины с дефолтными параметрами</br>
добавить свой ssh ключ в metadata ВМ</br>
зайти удаленным ssh (первая сессия), не забывайте про ssh-add</br>
поставить PostgreSQL</br>
зайти вторым ssh (вторая сессия)</br>
запустить везде psql из под пользователя postgres</br>
выключить `auto commit`</br>
сделать в первой сессии новую таблицу и наполнить ее данными 
```postgres 
create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
```
### **посмотреть текущий уровень изоляции:**
```postgres
show transaction isolation level
```
*Ответ:*  </br>
начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции</br>
в первой сессии добавить новую запись

```postgres 
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
сделать `select * from persons` во второй сессии
### **видите ли вы новую запись и если да то почему?**
*Ответ:* </br>

завершить первую транзакцию - `commit;`
сделать `select * from persons` во второй сессии

### **видите ли вы новую запись и если да то почему?**
*Ответ:* </br>

завершите транзакцию во второй сессии</br>
начать новые но уже `repeatable read` транзации - 
```postgres
set transaction isolation level repeatable read;
```
в первой сессии добавить новую запись 
```postgres
insert into persons(first_name, second_name) values('sveta', 'svetova');
```
сделать `select * from persons` во второй сессии
### **видите ли вы новую запись и если да то почему?**
*Ответ* </br>

завершить первую транзакцию - `commit;` </br>
сделать `select * from persons` во второй сессии
### **видите ли вы новую запись и если да то почему?**
*Ответ:*  </br>
завершить вторую транзакцию</br>
сделать `select * from persons` во второй сессии
### **видите ли вы новую запись и если да то почему?** 
*Ответ:*