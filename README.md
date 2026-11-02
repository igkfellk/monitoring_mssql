# 📊 Расширенный мониторинг MSSQL в Zabbix: Настройка кастомных метрик и логов

Эта инструкция описывает процесс настройки углубленного мониторинга Microsoft SQL Server с помощью Zabbix. Мы настроим отслеживание задержек (latches), времени выполнения самых долгих запросов, а также добавим алертинг на критические ошибки в лог-файлах.

Все настройки производятся в шаблоне **`MSSQL by Zabbix agent 2`**.

---

## 1. Мониторинг времени ожидания защелок (Latches)

Защелки (latches) — это легковесные примитивы синхронизации. Высокое время ожидания может указывать на узкие места в производительности.

### 1.1. Элемент данных (Item)
Необходимо создать вычисляемый элемент данных, который будет считать среднее время ожидания на основе сырых метрик, получаемых от агента.

*   **Имя:** `Average latch wait time`
*   **Тип:** Вычисляемый (Calculated)
*   **Ключ:** `mssql.average_latch_wait_time`
*   **Тип информации:** Число с плавающей точкой (Float)
*   **Единицы измерения:** `ms`
*   **Формула:** 
    ```text
    (last(//mssql.average_latch_wait_time_raw) - last(//mssql.average_latch_wait_time_raw,#2)) / (last(//mssql.average_latch_wait_time_base) - last(//mssql.average_latch_wait_time_base,#2) + (last(//mssql.average_latch_wait_time_base) - last(//mssql.average_latch_wait_time_base,#2)=0))
    ```

### 1.2. Триггер (Trigger)
Создаем оповещение при превышении порогового значения.

*   **Имя:** `MSSQL: High average latch wait time`
*   **Важность:** Высокая (High)
*   **Выражение:**
    ```text
    last(/MSSQL by Zabbix agent 2/mssql.average_latch_wait_time)>={$MSSQL.AVG_LATCH_WAIT_TIME}
    ```

### 1.3. Макрос (Macro)
Добавьте пользовательский макрос в шаблон `MSSQL by Zabbix agent 2` для гибкого управления порогом:
*   **Макрос:** `{$MSSQL.AVG_LATCH_WAIT_TIME}`
*   **Значение:** `10` (или другое количество миллисекунд, критичное для вашей системы).

---

## 2. Мониторинг времени выполнения запросов

Настройка кастомного SQL-запроса для отслеживания запросов, которые выполняются подозрительно долго.

### 2.1. Элемент данных (Item)
*   **Имя:** `Max request execution time per seconds`
*   **Тип:** Zabbix агент (пассивный или активный, в зависимости от конфигурации)
*   **Ключ:** 
    ```text
    mssql.custom.query["{$MSSQL.URI}","{$MSSQL.USER}","{$MSSQL.PASSWORD}","max_request_seconds"]
    ```
*   **Тип информации:** Целое положительное (Integer)
*   **Единицы измерения:** `s`
*   **Интервал обновления:** `30s`
*   **Предобработка:**
    *   **Имя:** JSONPath
    *   **Параметры:** `$[0].max_elapsed_seconds`

### 2.2. Триггеры (Triggers)
Создаем трехуровневую систему оповещений о долгих запросах:

**Триггер 1 (Предупреждение):**
*   **Имя:** `MSSQL: Request execution time [{ITEM.VALUE}]`
*   **Важность:** Средняя (Average)
*   **Выражение:** 
    ```text
    last(/MSSQL by Zabbix agent 2/mssql.custom.query["{$MSSQL.URI}","{$MSSQL.USER}","{$MSSQL.PASSWORD}","max_request_seconds"])>={$MSSQL.TIME_QUERY_AVERAGE}
    ```

**Триггер 2 (Высокая):**
*   **Имя:** `MSSQL: Request execution time [{ITEM.VALUE}]`
*   **Важность:** Высокая (High)
*   **Выражение:** 
    ```text
    last(/MSSQL by Zabbix agent 2/mssql.custom.query["{$MSSQL.URI}","{$MSSQL.USER}","{$MSSQL.PASSWORD}","max_request_seconds"])>={$MSSQL.TIME_QUERY_HIGH}
    ```

**Триггер 3 (Критическая):**
*   **Имя:** `MSSQL: Request execution time [{ITEM.VALUE}]`
*   **Важность:** Чрезвычайная (Disaster)
*   **Выражение:** 
    ```text
    last(/MSSQL by Zabbix agent 2/mssql.custom.query["{$MSSQL.URI}","{$MSSQL.USER}","{$MSSQL.PASSWORD}","max_request_seconds"])>={$MSSQL.TIME_QUERY_CRITICAL}
    ```

### 2.3. Макросы (Macros)
В шаблоне `MSSQL by Zabbix agent 2` добавьте следующие макросы для настройки порогов (в секундах):
*   `{$MSSQL.TIME_QUERY_AVERAGE}` = `7200`
*   `{$MSSQL.TIME_QUERY_HIGH}` = `14400`
*   `{$MSSQL.TIME_QUERY_CRITICAL}` = `21600`

### 2.4. SQL-запрос и настройка Zabbix Agent 2
Чтобы получать эту метрику, необходимо создать кастомный запрос.

**Шаг 1. SQL-запрос**
На сервере БД создайте файл `max_request_seconds.sql` со следующим содержимым:
```sql
SELECT ISNULL(MAX(r.total_elapsed_time) / 1000, 0) AS max_elapsed_seconds
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
WHERE s.is_user_process = 1
AND r.session_id <> @@SPID
AND r.session_id > 50;
```

**Шаг 2. Подготовка директорий**
*   В папке `C:\Program Files\Zabbix Agent 2` создайте папку `Custom Queries\MSSQL`.
*   Поместите туда ранее созданный файл `max_request_seconds.sql`.

**Шаг 3. Настройка конфигурации Агента**
В папке `C:\Program Files\Zabbix Agent 2\zabbix_agent2.d\plugins.d` отредактируйте файл `mssql.conf` и добавьте (или измените) следующие строки:
```ini
Plugins.MSSQL.CustomQueriesDir=C:\Program Files\Zabbix Agent 2\Custom Queries\MSSQL
Plugins.MSSQL.CustomQueriesEnabled=true
```

**Шаг 4. Перезапуск**
Перезагрузите службу **Zabbix Agent 2**, чтобы применить настройки.

---

## 3. Мониторинг критических ошибок в лог-файле

Отслеживание специфических ошибок MSSQL, таких как взаимные блокировки (latch timeouts) или проблемы с планировщиком.

### 3.1. Элемент данных (Item)
*   **Имя:** `Error log file`
*   **Тип:** Zabbix агент (активный) *(Важно: именно активный!)*
*   **Ключ:** 
    ```text
    logrt["E:/Microsoft SQL Server/MSSQL13.MSSQLSERVER/MSSQL/Log\ERRORLOG*","Time out occured while waiting for buffer latch|appears to be non-yielding on Scheduler","UTF-16",,,]
    ```
    *Примечание: Параметр `"UTF-16"` критически важен, так как логи MSSQL в Windows по умолчанию пишутся в этой кодировке.*
*   **Тип информации:** Журнал (Log)
*   **Интервал обновления:** `5s`

### 3.2. Триггер (Trigger)
*   **Имя:** `Error log file` (или `MSSQL: Critical error in Error Log`)
*   **Важность:** Высокая (High)
*   **Выражение:**
    ```text
    find(/MSSQL by Zabbix agent 2/logrt["E:/Microsoft SQL Server/MSSQL13.MSSQLSERVER/MSSQL/Log\ERRORLOG*","Time out occured while waiting for buffer latch|appears to be non-yielding on Scheduler","UTF-16",,,])=1
    ```
    *Примечание: Функция `find` ищет совпадение по регулярному выражению в новых строках лога.*
