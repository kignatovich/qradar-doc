Чтобы настроить мониторинг изменений двух полей в таблице PostgreSQL с последующей передачей данных в IBM QRadar, необходимо выполнить следующие шаги:

---

### 1. **Настроить аудит в PostgreSQL**

Для отслеживания изменений полей используйте функции PostgreSQL, такие как триггеры или расширение `pgAudit`.

#### Использование триггеров
Создайте триггер, который будет отслеживать изменения в нужных полях и записывать их в лог-таблицу.

```sql
-- Создаем таблицу для логов изменений
CREATE TABLE field_change_logs (
    id SERIAL PRIMARY KEY,
    table_name TEXT,
    column_name TEXT,
    old_value TEXT,
    new_value TEXT,
    change_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    user_name TEXT
);

-- Функция для записи изменений
CREATE OR REPLACE FUNCTION log_field_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.field1 IS DISTINCT FROM OLD.field1 THEN
        INSERT INTO field_change_logs(table_name, column_name, old_value, new_value, user_name)
        VALUES (TG_TABLE_NAME, 'field1', OLD.field1::TEXT, NEW.field1::TEXT, SESSION_USER);
    END IF;

    IF NEW.field2 IS DISTINCT FROM OLD.field2 THEN
        INSERT INTO field_change_logs(table_name, column_name, old_value, new_value, user_name)
        VALUES (TG_TABLE_NAME, 'field2', OLD.field2::TEXT, NEW.field2::TEXT, SESSION_USER);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Добавляем триггер на нужную таблицу
CREATE TRIGGER monitor_field_changes
AFTER UPDATE ON target_table
FOR EACH ROW
EXECUTE FUNCTION log_field_changes();
```

#### Использование pgAudit (опционально)
`pgAudit` позволяет вести аудит на уровне SQL-запросов, включая изменения данных. Убедитесь, что расширение включено и настроено.

---

### 2. **Подготовить интеграцию с QRadar**

Для передачи данных из PostgreSQL в QRadar используется Syslog или API. Подготовьте PostgreSQL к отправке логов:

#### Вариант 1: Использование `syslog`
1. Установите и настройте syslog-агент (например, `rsyslog` или `syslog-ng`) на сервере PostgreSQL.
2. Настройте PostgreSQL для отправки логов в syslog:
    ```conf
    # postgresql.conf
    logging_collector = on
    log_destination = 'syslog'
    syslog_facility = 'LOCAL0'
    syslog_ident = 'postgres'
    ```
3. Настройте syslog для отправки данных на QRadar.

#### Вариант 2: Использование внешнего скрипта
Напишите скрипт, который извлекает данные из таблицы `field_change_logs` и отправляет их в QRadar через API или в формате syslog.

**Пример скрипта на Python для отправки данных:**

```python
import psycopg2
import socket

# Параметры подключения к PostgreSQL
db_config = {
    'dbname': 'your_db',
    'user': 'your_user',
    'password': 'your_password',
    'host': 'localhost',
    'port': 5432
}

# Параметры Syslog
syslog_server = 'qradar_ip'
syslog_port = 514

def fetch_logs():
    conn = psycopg2.connect(**db_config)
    cur = conn.cursor()
    cur.execute("SELECT * FROM field_change_logs WHERE sent_to_qradar = FALSE")
    logs = cur.fetchall()
    conn.close()
    return logs

def send_to_syslog(log):
    message = f"Table: {log[1]}, Column: {log[2]}, Old: {log[3]}, New: {log[4]}, User: {log[6]}"
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.sendto(message.encode(), (syslog_server, syslog_port))

def update_log_status(log_id):
    conn = psycopg2.connect(**db_config)
    cur = conn.cursor()
    cur.execute("UPDATE field_change_logs SET sent_to_qradar = TRUE WHERE id = %s", (log_id,))
    conn.commit()
    conn.close()

logs = fetch_logs()
for log in logs:
    send_to_syslog(log)
    update_log_status(log[0])
```

Добавьте задачу в планировщик (`cron`, `systemd`) для регулярного выполнения.

---

### 3. **Настроить QRadar для обработки событий**
1. В QRadar настройте источник событий (Log Source) для PostgreSQL:
    - Укажите имя источника (например, PostgreSQL Logs).
    - Настройте тип источника (например, Syslog).
2. Создайте или настройте DSM (Device Support Module), чтобы QRadar корректно интерпретировал события.

---

После выполнения этих шагов вы сможете мониторить изменения в таблицах PostgreSQL и получать соответствующие события в QRadar.
