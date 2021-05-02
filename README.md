# RTDE-homework5: Data quality assurance with Great Expectations framework

## 1. Источники данных

В качестве источников данных используются таблицы в Greenplum,
соответствующие слою ODS хранилища данных, подготовленного в ДЗ №4.

Сервер `greenplum.rt.dadadata.ru`, схема `sperfilyev`.

Перечень проверяемых таблиц:

* `ods_t_billing` - данные биллинговой системы (начисления абонентов);
* `ods_t_issue` - данные о жалобах абонентов;
* `ods_t_payment` - данные о платежах абонентов;
* `ods_t_traffic` - данные о трафике абонентских устройств.

## 2. Общий подход к проверке для всех источников

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
* Из истории систем-источников известно, что идентификаторы всех абонентов начинаются с значений от 10000
(для исторически первых абонентов) и далее инкрементируются для новых абонентов. Поэтому проверяем,
что в данном поле число не менее 10000
* При профилировании источников установлено, что отношение кол-ва уникальных значений к общему кол-ву значений
составляет 0.123. Исходим из гипотезы, что в продакшене данное соотношение будет **не более 200 записей на 1 абонента**
и не должно приходить батчей, содержащих данные только для одного абонента. 
Поэтому устанавливаем требование, что **отношение кол-ва уникальных значений должно быть не менее 1/200**. 

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


## 3. Дополнительные проверки, внедрённые для отдельных источников

### 3.1. Таблица `billing`

* Уникальность должна обеспечиваться для группы полей (compound columns): `user_id, billing_period, service`.
Исходим из гипотезы, что каждому абоненту в каждом расчётном периоде по каждой услуге должно формироваться
не более одного начисления и это требование должно соблюдаться в операционной системе-источнике.
* `billing_period` - текстовое поле, должно быть не пустым и соответствовать шаблону *'ГГГГ-ММ'* (год-месяц).
Значения должны быть в диапазоне от января 2012 года до января 2030 года.
Исходим из гипотезы, что источники должны порождать от 50 до 200 записей в каждом расчётном периоде,
поэтому ограничиваем отношение кол-ва уникальных значений к общему кол-ву записей диапазоном от 1/200 до 1/50.
* `service` - текстовое поле, не пустое. Длина строки должна быть не менее 3 символов (для самых коротких
наименований/идентификаторов услуг, например "ШПД", "VPN")

### 3.2. Таблица `issues`

* Уникальность должна обеспечиваться для группы полей (compound columns): `user_id, start_time`.
Исходим из гипотезы, что один абонент в каждый момент времени может подать не более одной жалобы
и это требование должно соблюдаться в операционной системе-источнике.
* `service` - текстовое поле, не пустое.
Каждый батч должен содержать данные по трём услугам: `Connect, Disconnect, Setup Environment`.
Исходим из гипотезы, что размер батча достаточно велик для того, чтобы в одном батче обязательно присутствовали
жалобы на все услуги. Распределение количества жалоб по услугам должно быть близко к равномерному, 
т.е. веса составляют [1/3, 1/3, 1/3]
* `title` - текстовое поле, не пустое. Исходим из гипотезы, что большинство абонентов укажут осмысленную тему
жалобы длиной не менее 8 символов, чтобы это поле в дальнейшем можно было категоризировать и анализировать.
Допускаем, что 2% абонентов поспешат или поленятся и тема будет слишком короткой, эти записи придётся дополнительно 
обработать вручную.

### 3.3. Таблица `payment`

* Уникальность должна обеспечиваться для группы полей (compound columns): `user_id, pay_doc_type, pay_doc_num`.
Исходим из гипотезы, что каждая запись однозначно идентифицируется идентификатором абонента и реквизитами платежа
и это требование должно соблюдаться в операционной системе-источнике (первичный ключ).
* `account` - текстовое поле, не пустое, должно соответствовать шаблону *'FL-xxxxx'* (где xxxxx - от одной до пяти цифр).
Исходим из гипотезы, что в каждом батче будет содержаться не более 100 платежей с данного счёта, поэтому 
ограничиваем отношение кол-ва уникальных значений к общему кол-ву записей - не менее 1/100
* `billing_period` - проверки аналогично таблице `billing`.
* `pay_doc_num` - целочисленное поле, не пустое, значение должно быть положительным.
* `pay_doc_type` - текстовое поле, не пустое.
Каждый батч должен содержать данные по трём платёжным системам: `MASTER, MIR, VISA`.
Исходим из гипотезы, что размер батча достаточно велик для того, чтобы в одном батче обязательно присутствовали
платежи от всех систем. 
* `pay_sum` - параметры распределения значений (медиана, 1-й и 3-й квартили, а также 5% и 95%-квантили) должны
соответствовать эмпирически подобранным по результатам профилирования источника.
* `phone` - текстовое поле, не пустое, должно соответствовать шаблону *'+79xxxxxxxxx'*

### 3.4. Таблица `traffic`

* Уникальность должна обеспечиваться для группы полей (compound columns): `user_id, device_id, traffic_time`.
Исходим из гипотезы, что один абонент в каждый момент времени может сгенерировать с каждого устройства
только одну тарификационную запись и это требование должно соблюдаться в операционной системе-источнике.
* `bytes_received, bytes_sent` - целочисленные поля, не пустые. Минимальные и максимальные значения ограничены диапазонами:

    минимальное - от 0 до гипотетического предела передачи данных для абонентского устройства;

	максимальное - от 1 Кбайт до гипотетического предела передачи данных для абонентского устройства.

Гипотетический предел передачи данных для устройства за 1 сессию = максимальная полоса пропускания * длительность сессии.

Исходные предположения: максимальная длительность непрерывной сессии для одного устройства = 1 сутки = 24 * 3600 секунд;
максимальная полоса пропускания для подключенного к сети устройства = 10 Гбит/с = 10 * (2^10)^3 бит/с.
* `device_id` - текстовое поле, не пустое, должно соответствовать шаблону *'dNNN'*, где NNN - три цифры.

Для наших сферических целей в вакууме предполагаем, что в сети работают всего 9 устройств с идентификаторами
d001...d009 и каждое из них генерит примерно одинаковое количество тарификационных записей,
поэтому значения в данном поле распределены равномерно (9 значений с весами по 1/9).
* `device_ip_addr` - текстовое поле, не пустое, должно соответствовать классическому жаблону для представления ip-адреса:
*`xxx.xxx.xxx.xxx`* (где xxx - от одной до трёх цифр).

### Для выполнения работы использована библиотека Great Expectations

Исходный код здесь: https://github.com/great-expectations/great_expectations

Документация здесь: https://greatexpectations.io/

Зависимости: https://github.com/sergei74ap/RTDE-homework5/blob/main/requirements.txt
