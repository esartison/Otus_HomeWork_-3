# Домашнее Задание Евгения Сартисона #

Описание/Пошаговая инструкция выполнения домашнего задания:
*Однажды утром Алексей получил срочное сообщение от отдела логистики: "Мы не можем обновить данные о последней партии бананов, отправленной в Европу!" 
Оказалось, что диск виртуальной машины, где работала база данных, был заполнен на 95%. Алексей понял, что это не просто временная проблема: если данные не будут обновлены вовремя, 
это приведёт к задержкам поставок и штрафам от клиентов.

Причиной нехватки места стало не только увеличение объёмов данных, но и ошибка в настройках: логи транзакций (WAL-файлы) занимали слишком много места, 
так как никто не настроил их автоматическую очистку.
Алексей решил, что нужно не просто освободить место, но и обеспечить стабильность системы на будущее.
Он решил добавить внешний диск к виртуальной машине и перенести туда базу данных. Однако на пути к успеху его ждали новые вызовы:

***Как правильно проинициализировать и подключить внешний диск?***
***Как перенести данные, не потеряв их?***
***Как настроить PostgreSQL, чтобы он работал с новым диском?***


## (1) Создайте виртуальную машину с Ubuntu 20.04 и установите PostgreSQL 15 или выше. ##
взял уже существующую VM в YC с лабами для Docker, решил не создавать новую для экономии времени
![image](https://github.com/user-attachments/assets/6fd3e968-e27d-455c-bb03-2ffa716eb21d)

добавил новый диск в YC
![image](https://github.com/user-attachments/assets/3ff17eb5-f41c-4f32-abd5-770203991c40)


Установил Posrgres 17 с настройками по умолчанию
![image](https://github.com/user-attachments/assets/95acc8bd-8096-4696-9624-f0876d509192)


## (2) Создайте таблицу с данными о перевозках. ## 
Создать базу данных CARGO
![image](https://github.com/user-attachments/assets/9c59bcd2-39b3-48f4-9d8f-efbd9b5b0b1a)

Создал совсем простую таблицу

>CREATE TABLE all_shipments (
>  shipment_id SERIAL UNIQUE NOT NULL,
>  country_code VARCHAR(10) NOT NULL
>);
>
>insert into all_shipments (shipment_id, country_code)
>values (1,'BY');
![image](https://github.com/user-attachments/assets/9f227098-491f-4d7c-a4d7-8342f1bf71c6)





## (3) Добавьте внешний диск к виртуальной машине и перенесите туда базу данных. ## 
Добавил директорию /pgdataext, прописал в fstab, перезапустил VM чтобы убедиться что диск монтируется корректно
![image](https://github.com/user-attachments/assets/3364f313-f548-4207-a74f-35bc25726371)
Использовал документ [Разметить и смонтировать пустой диск](https://yandex.cloud/ru/docs/compute/operations/vm-control/vm-attach-disk?from=int-console-help-center-or-nav)

Для того что перенести pg_data на новый диск нужно остановить процессы, сделать копию, поправить конфиг и заново запустить экземпляр Postgres

data_directory ДО всех изменений

![image](https://github.com/user-attachments/assets/85104b33-8720-488d-9432-f8541bb941a0)

Нужно остановить сервис Postgres на сервере
![image](https://github.com/user-attachments/assets/29178b6c-7ad4-4525-a17f-ff7b729897eb)

синхронизовать через rsync старую и новую директорию
>sudo rsync -av /var/lib/postgresql /pgdataext
![image](https://github.com/user-attachments/assets/b0de9b99-f5e7-458c-9b51-0a1c4994cea3)

переименовываем старую директорию для каталога main
>mv /var/lib/postgresql/17/main /var/lib/postgresql/17/main_bkp_18May2025
![image](https://github.com/user-attachments/assets/172aea73-aa9c-4c42-9e12-8450c4987f5c)





## (4) Настройте PostgreSQL для работы с новым диском. ##

Правим локацию data_directory  в postgresql.conf
![image](https://github.com/user-attachments/assets/444d8522-528f-4a47-a315-8caebba1759f)

Перезапускаем postgres
![image](https://github.com/user-attachments/assets/a14b0f69-1cdf-48fb-b8e8-12f2a921a268)

Проверка новой локации ПОСЛЕ

![image](https://github.com/user-attachments/assets/9b0a7ced-3fb9-4737-8b0e-b347d5c276f3)

удяляем старую директорию

![image](https://github.com/user-attachments/assets/71d7a8cf-b593-4ba4-bcfc-498805a2ed4d)


## (5) Проверьте, что данные сохранились и доступны. ##

Данные после переноса на /pgdataext сохранились
![image](https://github.com/user-attachments/assets/cb41df5d-c40b-4a75-a359-acc722f6ac53)

Так-как копирование данных через rsync было консистентным(во время остановленного экземпляра) данные видны и все хорошо.


## Задание со звездочкой: перенос базы на другой сервер(VM) ##
Добавил новую VM(esartison-vm2) и остановил страую(esartison-vm1), подклюил pgdata в новую VM
![image](https://github.com/user-attachments/assets/7b3eb737-d298-4475-94a8-44e7f8165c4a)

установил postgres на esartison-vm2
![image](https://github.com/user-attachments/assets/0ede3fa8-02f8-45dc-8f37-eaade5499da3)

Остановил postgres
>sudo systemctl stop postgresql

создал новый pgdata
>sudo mkdir -p /postgres/pgdata
>sudo chown postgres:postgres /postgres/pgdata
>sudo chmod 700 /postgres/pgdata

Добавить существующий диск с помощью [Смонтировать диск, созданный из снимка или образа](https://yandex.cloud/ru/docs/compute/operations/vm-control/vm-attach-disk?from=int-console-help-center-or-nav)
>sudo mkdir /postgres/pgdata
>sudo mount /dev/vdb1 /postgres/pgdata
>sudo chown -R postgres:postgres /postgres/pgdata
>sudo chmod -R 700 /postgres/pgdata
![image](https://github.com/user-attachments/assets/e34711ac-e8df-4f0a-a086-209cdea26b83)

Поправить локацию для data_directory
![image](https://github.com/user-attachments/assets/93b59a84-6951-4a30-b213-079414d8dc54)

запустить Postgres

![image](https://github.com/user-attachments/assets/79d1b879-1fdc-47ef-8f8b-1e18c41e9c79)

Проверка базы CARGO в esartison-vm2 - база на месте
![image](https://github.com/user-attachments/assets/5f1e6c90-cb2b-4ac3-b3f7-1d1d7e5d71c8)

проверка данных в таблице
![image](https://github.com/user-attachments/assets/20a40e7c-b1dd-4383-9cb7-5e2fbe76467e)

Все сработало, успешно перенесли кластер с таблицей на другой сервер. 




