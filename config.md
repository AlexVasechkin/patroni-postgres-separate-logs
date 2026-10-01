# Конфигурация разделения логов Patroni и PostgreSQL 16

## 1. Текущее состояние

```
patroni.service (systemd)
 └── stderr ──> journal ──┬── сообщения Patroni (python logging -> stderr)
                          └── сообщения PostgreSQL (logging_collector = off -> stderr)
```

Оба потока смешаны, читаются только через `journalctl -u patroni`, ротация — только по правилам journald.

## 2. Целевое состояние

```
patroni.service (systemd)
 ├── Patroni      -> /var/log/patroni/patroni.log      (ротация средствами Patroni)
 └── PostgreSQL   -> /var/log/postgresql/*.log         (ротация средствами logging_collector)
```

Каталоги создаются заранее, владелец — пользователь СУБД:

```bash
install -d -o postgres -g postgres -m 0750 /var/log/patroni
install -d -o postgres -g postgres -m 0750 /var/log/postgresql
```

---

## 3. Параметры логирования Patroni (секция `log` в `patroni.yml`)

По умолчанию секция `log` в `patroni.yml` отсутствует целиком — Patroni пишет в `stderr`.

| Параметр | Стандартное значение | Новое значение |
| --- | --- | --- |
| `log.type` | `plain` | `plain` |
| `log.level` | `INFO` | `INFO` |
| `log.traceback_level` | `ERROR` | `ERROR` |
| `log.format` | `%(asctime)s %(levelname)s: %(message)s` | `%(asctime)s %(levelname)s [%(threadName)s] %(name)s: %(message)s` |
| `log.dateformat` | не задан (ISO + мс) | `%Y-%m-%d %H:%M:%S %z` |
| `log.dir` | не задан (вывод в `stderr`) | `/var/log/patroni` |
| `log.file_num` | `4` | `10` |
| `log.file_size` | `25000000` (25 МБ) | `52428800` (50 МБ) |
| `log.mode` | не задан (права по umask) | `0640` |
| `log.max_queue_size` | `1000` | `1000` |
| `log.loggers` | `{}` (пусто) | `urllib3: WARNING`, `patroni.postgresql.config: WARNING` |
| `log.deduplicate_heartbeat_logs` | `false` | `true` |

Фрагмент `patroni.yml`:

```yaml
log:
  type: plain
  level: INFO
  traceback_level: ERROR
  format: '%(asctime)s %(levelname)s [%(threadName)s] %(name)s: %(message)s'
  dateformat: '%Y-%m-%d %H:%M:%S %z'
  dir: /var/log/patroni
  file_num: 10
  file_size: 52428800
  mode: 0640
  max_queue_size: 1000
  deduplicate_heartbeat_logs: true
  loggers:
    urllib3: WARNING
    patroni.postgresql.config: WARNING
```

> Имя файла задаётся самим Patroni: `<log.dir>/patroni.log`, ротированные — `patroni.log.1` … `patroni.log.N`.
> Секция `log` читается из локального `patroni.yml`, в DCS не хранится; применяется по `systemctl reload patroni` без рестарта PostgreSQL.

---

## 4. Параметры логирования PostgreSQL 16 (`postgresql.parameters` в `patroni.yml`)

| Параметр | Стандартное значение | Новое значение |
| --- | --- | --- |
| `logging_collector` | `off` | `on` |
| `log_destination` | `stderr` | `stderr` |
| `log_directory` | `log` (относительно `PGDATA`) | `/var/log/postgresql` |
| `log_filename` | `postgresql-%Y-%m-%d_%H%M%S.log` | `postgresql-%Y-%m-%d.log` |
| `log_file_mode` | `0600` | `0640` |
| `log_rotation_age` | `1d` | `1d` |
| `log_rotation_size` | `10MB` | `100MB` |
| `log_truncate_on_rotation` | `off` | `on` |
| `log_line_prefix` | `%m [%p] ` | `%m [%p] %q%u@%d %a %h %x ` |
| `log_timezone` | `GMT` | `Europe/Moscow` |
| `log_min_messages` | `warning` | `warning` |
| `log_min_error_statement` | `error` | `error` |
| `log_error_verbosity` | `default` | `default` |
| `log_min_duration_statement` | `-1` (выключено) | `1000` (мс) |
| `log_min_duration_sample` | `-1` | `-1` |
| `log_statement_sample_rate` | `1.0` | `1.0` |
| `log_statement` | `none` | `ddl` |
| `log_duration` | `off` | `off` |
| `log_checkpoints` | `on` | `on` |
| `log_connections` | `off` | `on` |
| `log_disconnections` | `off` | `on` |
| `log_lock_waits` | `off` | `on` |
| `log_temp_files` | `-1` (выключено) | `0` (все) |
| `log_autovacuum_min_duration` | `10min` | `1min` |
| `log_replication_commands` | `off` | `on` |
| `log_recovery_conflict_waits` | `off` | `on` |
| `log_hostname` | `off` | `off` |
| `log_startup_progress_interval` | `10s` | `10s` |
| `log_parameter_max_length` | `-1` | `-1` |
| `log_parameter_max_length_on_error` | `0` | `0` |
| `syslog_facility` | `LOCAL0` | `LOCAL0` (не используется) |
| `syslog_ident` | `postgres` | `postgres` (не используется) |
| `syslog_sequence_numbers` | `on` | `on` (не используется) |
| `syslog_split_messages` | `on` | `on` (не используется) |
| `cluster_name` | `''` (пусто) | `pg16-cluster` (= `scope` кластера) |

Фрагмент `patroni.yml`:

```yaml
bootstrap:
  dcs:
    postgresql:
      parameters:
        logging_collector: 'on'
        log_destination: 'stderr'
        log_directory: '/var/log/postgresql'
        log_filename: 'postgresql-%Y-%m-%d.log'
        log_file_mode: '0640'
        log_rotation_age: '1d'
        log_rotation_size: '100MB'
        log_truncate_on_rotation: 'on'
        log_line_prefix: '%m [%p] %q%u@%d %a %h %x '
        log_timezone: 'Europe/Moscow'
        log_min_messages: 'warning'
        log_min_error_statement: 'error'
        log_min_duration_statement: 1000
        log_statement: 'ddl'
        log_checkpoints: 'on'
        log_connections: 'on'
        log_disconnections: 'on'
        log_lock_waits: 'on'
        log_temp_files: 0
        log_autovacuum_min_duration: '1min'
        log_replication_commands: 'on'
        log_recovery_conflict_waits: 'on'
        cluster_name: 'pg16-cluster'
```

Для уже работающего кластера секция `bootstrap` не перечитывается — параметры меняются через DCS:

```bash
patronictl -c /etc/patroni/patroni.yml edit-config
# или по одному параметру:
patronictl -c /etc/patroni/patroni.yml edit-config \
  -p logging_collector=on -p log_directory=/var/log/postgresql --force
```

### Применение изменений

- **Требуют `restart` PostgreSQL** (`pending_restart` в `patronictl list`): `logging_collector`, `log_directory`, `log_filename`, `log_file_mode`, `log_rotation_age`, `log_rotation_size`, `log_truncate_on_rotation`, `cluster_name`.
  ```bash
  patronictl -c /etc/patroni/patroni.yml restart pg16-cluster --pending
  ```
- **Применяются по `reload`** (SIGHUP): все остальные параметры из таблицы.
  ```bash
  patronictl -c /etc/patroni/patroni.yml reload pg16-cluster
  ```

> `log_directory` вне `PGDATA` должен существовать и быть доступен на запись пользователю `postgres`,
> иначе `logging_collector` не стартует и PostgreSQL не поднимется.

---

## 5. Параметры systemd-юнита `patroni.service`

`/etc/systemd/system/patroni.service.d/logging.conf`:

| Параметр | Стандартное значение | Новое значение |
| --- | --- | --- |
| `StandardOutput` | `journal` (наследуется из `DefaultStandardOutput`) | `journal` |
| `StandardError` | `inherit` (= `journal`) | `journal` |
| `SyslogIdentifier` | имя юнита (`patroni`) | `patroni` |
| `ExecReload` | `/bin/kill -s HUP $MAINPID` | `/bin/kill -s HUP $MAINPID` |
| `KillMode` | `control-group` | `process` |
| `TimeoutStopSec` | `90` | `30` |

```ini
[Service]
StandardOutput=journal
StandardError=journal
SyslogIdentifier=patroni
KillMode=process
TimeoutStopSec=30
```

```bash
systemctl daemon-reload && systemctl restart patroni
```

> После включения `log.dir` и `logging_collector` в journal остаются только сообщения самого
> systemd и несколько строк PostgreSQL, напечатанных до старта сборщика логов.
> Если нужно полное отсутствие вывода в journal — `StandardOutput=null`, но тогда теряются
> ранние ошибки запуска; без крайней необходимости так делать не стоит.

---

## 6. Проверка

```bash
# 1. В journal — только Patroni
journalctl -u patroni -n 50 --no-pager

# 2. Логи Patroni
ls -l /var/log/patroni/
tail -f /var/log/patroni/patroni.log

# 3. Логи PostgreSQL
ls -l /var/log/postgresql/
tail -f /var/log/postgresql/postgresql-$(date +%F).log

# 4. Фактические значения в СУБД
psql -c "SELECT name, setting, pending_restart FROM pg_settings
         WHERE name LIKE 'log%' OR name = 'cluster_name' ORDER BY name;"

# 5. Куда пишет сборщик логов
psql -c "SELECT pg_current_logfile();"

# 6. Состояние кластера (не должно быть pending_restart)
patronictl -c /etc/patroni/patroni.yml list
```
