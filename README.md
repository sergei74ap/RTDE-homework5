# RTDE-homework5: Data quality assurance with Great Expectations framework

## 1. Источники данных

В качестве источника данных используются таблицы в Greenplum,
соответствующие слою ODS хранилища данных, подготовленного в ДЗ №4.

Сервер `greenplum.rt.dadadata`, схема `sperfilyev`.

Перечень проверяемых таблиц:

* `ods_t_billing` - данные биллинговой системы (начисления абонентов);
* `ods_t_issue` - данные о жалобах абонентов;
* `ods_t_payment` - данные о платежах абонентов;
* `ods_t_traffic` - данные о трафике абонентских устройств.

## 1. Общий подход к проверке для всех источников

Для всех таблиц применены следующие проверки:

### Количество записей в таблице:

Исходим из гипотезы, что ETL-процедура забирает из каждой таблицы от 1 тыс. до 20 тыс. записей.

### Количество колонок в таблице:

Фиксируем количество столбцов согласно структуре ранее созданного слоя ODS:

* billing - 6
* issue - 6
* payment - 8 
* traffic - 6

Других столбцов в этих источниках быть не должно. Если возникнут изменения в системах-источниках
или в ETL-процедуре и это приведёт к изменению количества столбцов - процедура DQ должна это сразу выявить.

### Список колонок в таблице:

Фиксируем состав и порядок столбцов согласно структуре ранее созданного слоя ODS:

* billing - user_id, billing_period, service, tariff, charged_sum, created_at
* issue - user_id, start_time, end_time, title, description, service
* payment - user_id, pay_doc_type, pay_doc_num, account, phone, billing_period, pay_date, pay_sum
* traffic - user_id, traffic_time, device_id, device_ip_addr, bytes_sent, bytes_received

Других столбцов в этих источниках быть не должно. Если возникнут изменения в системах-источниках
или в ETL-процедуре и это приведёт к изменению состава столбцов - процедура DQ должна это сразу выявить.

### Поле `user_id`

Данное поле является общим для всех источников, проверки для него единообразные:

* Не должно быть пропусков (NULL)
* Поле должно иметь целочисленный тип (INTEGER)
* Из истории систем-источников известно, что идентификаторы всех абонентов начинаются с значения 10000
(для исторически первого абонента) и далее инкрементируются для новых абонентов. Поэтому проверяем,
что в данном поле число не менее 10000
* При профилировании источников установлено, что отношение кол-ва уникальных значений к общему кол-ву значений
составляет 0.123. Исходим из гипотезы, что в продакшене данное соотношение будет <= 200 записей на 1 абонента
и не должно приходить батчей, содержащих данные только для одного абонента, поэтому отношение должно быть
>= 1/200. 

### Поля, содержащие дату/время

Поля, содержащие дату/время, должны удовлетворять следующим требованиям:

* Не должно быть пропусков (NULL)
* Тип поля должен соответствовать типу, заданному в структуре ODS DWH (DATETIME/TIMESTAMP/DATE)
* Известно, что системы-источники начали фиксировать операции с начала 2012 года. Также исходим из гипотезы, 
что процедуры ETL и DQ должны будут корректно работать до начала 2030 года. Поэтому все даты ограничиваем
диапазоном от 01.01.2012 до 01.01.2030.

### Поля, содержащие денежные суммы

* Не должно быть пропусков (NULL)
* Тип поля должен соответствовать типу, заданному в структуре ODS DWH, корректно отрабатывающиему
операции с денежными суммами (NUMERIC/DECIMAL)
* Исходим из гипотезы, что для 99% абонентов объёмы услуг нашей компании за любой период составляют
не более 300 тыс.руб. Также считаем, что в биллинговых и платёжных системах не должно быть нулевых сумм.
Поэтому все денежные суммы ограничиваем диапазоном от 0.01 (1 копейка) до 300000.00 (рублей). 

### Поля, содержащие строки (текст)
* Во всех полях не должно быть пропусков (NULL)
* Поле должно иметь тип, определённый в структуре ODS DWH (в ДЗ №4 был применён тип TEXT)


## 1. Проверки, внедрённые для отдельных источников

### 1.1. Таблица `billing`

### 1.1. Таблица `issues`

### 1.1. Таблица `payment`

### 1.1. Таблица `traffic`