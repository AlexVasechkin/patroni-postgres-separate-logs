# Конфигурация разделения логов Patroni и PostgreSQL 16 (production)

Документ описывает перевод production-кластера Patroni + PostgreSQL 16 со стандартного
смешанного логирования в journal на два независимых потока логов.

**Условные значения, которые нужно подставить под свой кластер:**
`scope` кластера — `pg16-cluster`, путь к конфигу — `/etc/patroni/patroni.yml`,
таймзона — `Europe/Moscow`, удержание логов PostgreSQL — 30 дней.

---

## 1. Текущее состояние

```
patroni.service (systemd)
 └── stderr ──> journal ──┬── сообщения Patroni (python logging -> stderr)
                          └── сообщения PostgreSQL (logging_collector = off -> stderr)
```

Оба потока смешаны, читаются только через `journalctl -u patroni`, ротация — только по правилам
journald (`SystemMaxUse`), раздельное удержание и раздельные уровни детализации невозможны.

## 2. Целевое состояние

```
patroni.service (systemd)
 ├── Patroni      -> /var/log/patroni/patroni.log      (ротация средствами Patroni)
 ├── PostgreSQL   -> /var/log/postgresql/*.log         (ротация средствами logging_collector)
 └── stderr       -> journal                           (только systemd + ранний старт postmaster)
```

---

## 3. Подготовка ОС (выполняется на всех узлах до изменения конфигурации)

```bash
# Каталоги логов. Владелец — системный пользователь СУБД (postgres/enterprisedb).
install -d -o postgres -g postgres -m 0750 /var/log/patroni
install -d -o postgres -g postgres -m 0750 /var/log/postgresql
```

**Файловая система.** `/var/log` (или отдельные точки монтирования под эти каталоги) должна быть
отдельным разделом/LV — не тем, где лежит `PGDATA`, и не корневым. Переполнение каталога логов при
`logging_collector = on` приводит к остановке записи логов, а в худшем случае — к падению postmaster;
если это тот же раздел, что `PGDATA`, кластер встанет целиком.

**SELinux (RHEL/CentOS/Rocky).** Каталог вне `PGDATA` требует корректной метки, иначе сборщик
логов не сможет писать:

```bash
semanage fcontext -a -t postgresql_log_t '/var/log/postgresql(/.*)?'
semanage fcontext -a -t postgresql_log_t '/var/log/patroni(/.*)?'
restorecon -Rv /var/log/postgresql /var/log/patroni
```

**Удержание логов PostgreSQL.** `logging_collector` сам старые файлы не удаляет.
`/etc/tmpfiles.d/postgresql-log.conf`:

```
d /var/log/postgresql 0750 postgres postgres 30d
d /var/log/patroni     0750 postgres postgres -
```

```bash
systemd-tmpfiles --create /etc/tmpfiles.d/postgresql-log.conf
systemctl enable --now systemd-tmpfiles-clean.timer
```

> **Анти-паттерн:** внешний `logrotate` с `copytruncate` поверх `logging_collector` или поверх
> лог-файла Patroni. Оба процесса держат файл открытым и продолжают писать по старому дескриптору —
> результат: потерянные записи и файлы-«дыры». Ротацию делают сами PostgreSQL и Patroni,
> внешний инструмент отвечает только за удаление старых файлов.

**Расчёт объёма раздела под логи:**

| Поток | Формула | При значениях из этого документа |
| --- | --- | --- |
| Patroni | `file_num × file_size` | `10 × 50 МБ` = **500 МБ** (жёсткий потолок) |
| PostgreSQL | `суточный объём × удержание` | при 200 МБ/сутки и 30 днях ≈ **6 ГБ** |

Итого под `/var/log` резервируется не менее **10 ГБ** с алертом на 80% заполнения.
Суточный объём измеряется на пилотной реплике (п. 7) до раскатки на лидер.

---

## 4. Параметры логирования Patroni (секция `log` в `patroni.yml`)

По умолчанию секция `log` в `patroni.yml` отсутствует целиком — Patroni пишет в `stderr`.
Секция читается из локального файла на узле, в DCS не хранится и применяется
по `systemctl reload patroni` **без рестарта PostgreSQL**.

| Параметр | Стандартное значение | Новое значение | Комментарий |
| --- | --- | --- | --- |
| `log.type` | `plain` | `plain` | Формат записей: `plain` (текст) или `json` (структурированный вывод для систем сбора логов) |
| `log.level` | `INFO` | `INFO` | Минимальный уровень сообщений Patroni, попадающих в лог: `DEBUG`/`INFO`/`WARNING`/`ERROR`/`CRITICAL` |
| `log.traceback_level` | `ERROR` | `ERROR` | Уровень, начиная с которого в лог пишется полный python-трейсбек исключения |
| `log.format` | `%(asctime)s %(levelname)s: %(message)s` | `%(asctime)s %(levelname)s [%(threadName)s] %(name)s: %(message)s` | Шаблон строки лога в терминах полей модуля `logging` python |
| `log.dateformat` | не задан (ISO + мс) | `%Y-%m-%d %H:%M:%S %z` | Формат метки времени в строке лога (`strftime`); без него — ISO с микросекундами |
| `log.dir` | не задан (вывод в `stderr`) | `/var/log/patroni` | Каталог для файлов лога. **Ключевой параметр разделения:** пока не задан, Patroni пишет в `stderr`, то есть в journal |
| `log.file_num` | `4` | `10` | Сколько ротированных файлов хранить (`patroni.log.1` … `.N`). Вместе с `file_size` задаёт жёсткий потолок объёма |
| `log.file_size` | `25000000` (25 МБ) | `52428800` (50 МБ) | Размер файла в байтах, при достижении которого Patroni выполняет ротацию |
| `log.mode` | не задан (права по umask) | `0640` | Права доступа на создаваемые файлы лога (нужны, если логи читает агент сбора) |
| `log.max_queue_size` | `1000` | `1000` | Размер очереди асинхронной записи. При переполнении записи отбрасываются — логирование никогда не блокирует основной цикл HA |
| `log.loggers` | `{}` (пусто) | `urllib3: WARNING` | Переопределение уровня для отдельных python-модулей; используется для подавления шума библиотек |

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
  loggers:
    urllib3: WARNING
```

Замечания для production:

- Имя файла задаёт сам Patroni: `<log.dir>/patroni.log`, ротированные — `patroni.log.1` … `patroni.log.10`.
  Объём ограничен сверху жёстко, внешняя ротация не нужна и вредна.
- `log.level` оставлен `INFO`: на `WARNING` теряется хронология выборов лидера и смены ролей —
  именно то, что нужно при разборе инцидента. Это основной diagnostic-поток кластера.
- `log.mode` и `log.type: json` поддерживаются начиная с Patroni 3.1; `log.deduplicate_heartbeat_logs` —
  с Patroni 4.0. Перед применением сверить с `patroni --version`; неизвестные ключи Patroni
  игнорирует молча, то есть настройка просто не заработает.
- `log.dir` нельзя указывать внутри `PGDATA`: каталог перезаписывается при `pg_basebackup`
  во время реинициализации реплики (`patronictl reinit`).

---

## 5. Параметры логирования PostgreSQL 16 (`postgresql.parameters` в `patroni.yml`)

| Параметр | Стандартное значение | Новое значение | Комментарий |
| --- | --- | --- | --- |
| `logging_collector` | `off` | `on` | Включает отдельный процесс-сборщик, перехватывающий `stderr` postmaster и backend-ов и пишущий его в файлы. **Ключевой параметр разделения потоков** |
| `log_destination` | `stderr` | `stderr` | Куда направлять логи: `stderr`, `csvlog`, `jsonlog`, `syslog` (можно список через запятую) |
| `log_directory` | `log` (относительно `PGDATA`) | `/var/log/postgresql` | Каталог для файлов лога; абсолютный путь или путь относительно `PGDATA` |
| `log_filename` | `postgresql-%Y-%m-%d_%H%M%S.log` | `postgresql-%Y-%m-%d_%H%M%S.log` | Шаблон имени файла (`strftime`). Определяет уникальность файлов и то, безопасна ли ротация |
| `log_file_mode` | `0600` | `0640` | Права доступа на создаваемые файлы лога |
| `log_rotation_age` | `1d` | `1d` | Период, по истечении которого начинается новый файл (ротация по времени); `0` — отключить |
| `log_rotation_size` | `10MB` | `100MB` | Объём, при достижении которого начинается новый файл (ротация по размеру); `0` — отключить |
| `log_truncate_on_rotation` | `off` | `off` | Перезаписывать существующий файл с тем же именем вместо дописывания в конец. Безопасен только при неуникальном `log_filename` и выключенной ротации по размеру |
| `log_line_prefix` | `%m [%p] ` | `%m [%p-%l] %q%u@%d %a %h %x ` | Префикс каждой строки: время, pid, номер строки, пользователь, база, приложение, хост клиента, xid |
| `log_timezone` | `GMT` | `Europe/Moscow` | Таймзона метки времени в логах; не зависит от `TimeZone` клиентских сессий |
| `log_min_messages` | `warning` | `warning` | Минимальный уровень серверных сообщений, попадающих в лог |
| `log_min_error_statement` | `error` | `error` | Уровень ошибки, начиная с которого в лог пишется текст вызвавшего её SQL-запроса |
| `log_error_verbosity` | `default` | `default` | Детальность сообщения об ошибке: `terse` / `default` / `verbose` (DETAIL, HINT, позиция, контекст) |
| `log_min_duration_statement` | `-1` (выключено) | `1000` (мс) | Порог длительности в мс, выше которого запрос логируется с полным текстом; `-1` — выключено. Основной инструмент поиска медленных запросов |
| `log_min_duration_sample` | `-1` | `-1` | Порог длительности для **выборочного** логирования; работает в паре с `log_statement_sample_rate` |
| `log_statement_sample_rate` | `1.0` | `1.0` | Доля запросов, превысивших `log_min_duration_sample`, которые попадут в лог (способ снизить объём без потери картины) |
| `log_statement` | `none` | `ddl` | Какие SQL-команды логировать всегда, независимо от длительности: `none` / `ddl` / `mod` / `all` |
| `log_duration` | `off` | `off` | Логировать длительность каждой завершённой команды — без текста запроса |
| `log_checkpoints` | `on` | `on` | Логировать контрольные и рестарт-точки со статистикой записи буферов и длительностью этапов |
| `log_connections` | `off` | `off` (включать осознанно, см. ниже) | Логировать установление соединения и успешную аутентификацию (по сообщению на каждое соединение) |
| `log_disconnections` | `off` | `off` (включать осознанно, см. ниже) | Логировать закрытие соединения с длительностью сессии |
| `log_lock_waits` | `off` | `on` | Логировать сессии, ожидавшие блокировку дольше `deadlock_timeout`; основной признак проблем с конкурентностью |
| `log_temp_files` | `-1` (выключено) | `10240` (кБ, т.е. от 10 МБ) | Логировать временные файлы размером не меньше указанного (в кБ); `0` — все, `-1` — выключено |
| `log_autovacuum_min_duration` | `10min` | `1min` | Порог длительности, выше которого операция автовакуума или автоанализа пишется в лог |
| `log_replication_commands` | `off` | `on` | Логировать команды репликации (`START_REPLICATION`, `BASE_BACKUP` и прочие) — видно работу Patroni с репликами |
| `log_recovery_conflict_waits` | `off` | `on` | Логировать на реплике ожидания, вызванные конфликтами восстановления (применение WAL против запросов) |
| `log_hostname` | `off` | `off` | Писать имя хоста клиента вместо только IP; требует обратного DNS-запроса на каждое соединение |
| `log_startup_progress_interval` | `10s` | `10s` | Интервал отчётов о прогрессе длительных операций запуска (например, восстановления после сбоя) |
| `log_parameter_max_length` | `-1` | `-1` | Сколько символов значений bind-параметров писать при логировании запросов; `-1` — полностью, `0` — не писать |
| `log_parameter_max_length_on_error` | `0` | `0` | То же для сообщений об ошибках; `0` — параметры в лог не попадают |
| `syslog_facility` | `LOCAL0` | `LOCAL0` (не используется) | Категория syslog. Действует только при `log_destination = syslog` |
| `syslog_ident` | `postgres` | `postgres` (не используется) | Имя программы в сообщениях syslog. Действует только при `log_destination = syslog` |
| `syslog_sequence_numbers` | `on` | `on` (не используется) | Нумерация сообщений syslog для защиты от подавления повторяющихся строк |
| `syslog_split_messages` | `on` | `on` (не используется) | Разбивать длинные сообщения на отдельные строки syslog |
| `cluster_name` | `''` (пусто) | `pg16-cluster` | Имя кластера: подставляется в заголовок процессов и различает узлы при централизованном сборе логов |

### Обоснование значений, выбранных иначе, чем «включить всё»

- **`log_truncate_on_rotation: off` + уникальный `log_filename`.** Это единственная безопасная
  комбинация при одновременной ротации по времени (`1d`) и по размеру (`100MB`). С «красивым»
  именем вида `postgresql-%Y-%m-%d.log` и `log_truncate_on_rotation: on` вторая за сутки ротация
  по размеру **обрезает тот же файл** и уничтожает уже записанные логи за день — для production
  это прямая потеря диагностики. Чистка старых файлов вынесена в `tmpfiles.d` (п. 3).
- **`log_connections` / `log_disconnections`: оставлены `off`.** На нагруженном кластере без
  пулера (или с пулером в режиме `session`) это два лишних сообщения на каждое соединение —
  кратный рост объёма логов и заметная нагрузка на запись. Включать, только если это требование
  аудита и после замера суточного объёма на пилотной реплике.
- **`log_temp_files: 10240` вместо `0`.** `0` логирует *каждый* временный файл, включая мелкие
  сортировки; на OLTP-нагрузке это шум, который вытесняет полезные записи. Порог 10 МБ ловит
  реально проблемные запросы.
- **`log_min_duration_statement: 1000`.** Безопасный старт. Если объём логов окажется высоким,
  снижать детализацию не уменьшением порога, а через `log_min_duration_sample` +
  `log_statement_sample_rate`.
- **`log_statement: ddl`.** Даёт историю изменений схемы практически без объёма.
  `all`/`mod` в production не включать без отдельного обоснования.
- **`log_min_messages: warning`.** Понижать до `info`/`debug1` на лидере нельзя: рост объёма
  на порядки.
- **`log_line_prefix`.** Содержит время, pid, номер строки в сессии, пользователя, базу,
  приложение, хост и xid — минимум для корреляции. Если логи разбираются pgBadger,
  префикс менять нельзя без сверки с его требованиями.
- **`log_destination: stderr`.** Оставлен как есть. PostgreSQL 16 поддерживает `jsonlog` —
  переходить на него стоит только вместе с готовым парсером в системе сбора логов,
  отдельной задачей.

```yaml
postgresql:
  parameters:
    logging_collector: 'on'
    log_destination: 'stderr'
    log_directory: '/var/log/postgresql'
    log_filename: 'postgresql-%Y-%m-%d_%H%M%S.log'
    log_file_mode: '0640'
    log_rotation_age: '1d'
    log_rotation_size: '100MB'
    log_truncate_on_rotation: 'off'
    log_line_prefix: '%m [%p-%l] %q%u@%d %a %h %x '
    log_timezone: 'Europe/Moscow'
    log_min_messages: 'warning'
    log_min_error_statement: 'error'
    log_min_duration_statement: 1000
    log_statement: 'ddl'
    log_checkpoints: 'on'
    log_lock_waits: 'on'
    log_temp_files: 10240
    log_autovacuum_min_duration: '1min'
    log_replication_commands: 'on'
    log_recovery_conflict_waits: 'on'
    cluster_name: 'pg16-cluster'
```

### Применение изменений

Параметры PostgreSQL на работающем кластере хранятся в DCS. Секция `bootstrap.dcs` после
инициализации кластера **не перечитывается** — правка `patroni.yml` на узлах ничего не изменит.

```bash
# Просмотр и правка с показом diff и подтверждением (штатный путь для production):
patronictl -c /etc/patroni/patroni.yml edit-config pg16-cluster
```

`--force` в production не использовать: он применяет изменения без показа diff.

| Группа | Параметры | Способ применения |
| --- | --- | --- |
| `postmaster` (нужен restart) | `logging_collector`, `cluster_name` | `patronictl restart ... --pending` в порядке из п. 6 |
| `sighup` / `superuser` (достаточно reload) | все остальные параметры из таблицы | `patronictl reload pg16-cluster` |

> Это важно для планирования окна: `log_directory`, `log_filename`, `log_rotation_*`,
> `log_file_mode`, `log_truncate_on_rotation`, `log_line_prefix`, `log_timezone` имеют контекст
> `sighup` и применяются **без рестарта**. Рестарт нужен только из-за `logging_collector`
> (и `cluster_name`, если он задаётся) — поэтому оба эти параметра меняются в один проход,
> чтобы обойтись одним рестартом на узел.

Проверить контекст на своей сборке:

```bash
psql -Atc "SELECT name, context FROM pg_settings
           WHERE name LIKE 'log%' OR name IN ('cluster_name','logging_collector')
           ORDER BY context, name;"
```

---

## 6. Порядок раскатки на production-кластере

Рестарт нужен на каждом узле, поэтому последовательность обязательна.

**Шаг 0. Фиксация текущего состояния (для отката)**

```bash
patronictl -c /etc/patroni/patroni.yml show-config > /root/dcs-config-$(date +%F).yml
cp /etc/patroni/patroni.yml /root/patroni.yml-$(date +%F)
patronictl -c /etc/patroni/patroni.yml list
```

**Шаг 1. Подготовка ОС на всех узлах** — п. 3 (каталоги, SELinux, `tmpfiles.d`).
Изменений в работе кластера нет.

**Шаг 2. Логи Patroni на всех узлах, по одному**

```bash
# добавить секцию log в /etc/patroni/patroni.yml
systemctl reload patroni        # НЕ restart: reload = SIGHUP, PostgreSQL не трогается
tail -n 20 /var/log/patroni/patroni.log
```

Начинать с реплики, лидер — последним. Проверять `patronictl list` после каждого узла.

**Шаг 3. Параметры PostgreSQL, применяемые по reload**

```bash
patronictl -c /etc/patroni/patroni.yml edit-config pg16-cluster   # все параметры из п. 5
patronictl -c /etc/patroni/patroni.yml reload pg16-cluster
```

После этого `patronictl list` покажет `pending_restart` на всех узлах — это ожидаемо,
из-за `logging_collector` и `cluster_name`.

**Шаг 4. Рестарт реплик (по одной)**

```bash
patronictl -c /etc/patroni/patroni.yml restart pg16-cluster <replica-1> --pending
# дождаться streaming, проверить лаг, только затем следующая реплика
patronictl -c /etc/patroni/patroni.yml list
psql -h <replica-1> -c "SELECT pg_current_logfile();"
```

Пауза между репликами — не меньше времени, нужного на прогрев и возврат в `streaming`.

**Шаг 5. Пилотное наблюдение.** Оставить кластер на сутки с перенастроенными репликами.
Замерить суточный объём логов и сверить с расчётом из п. 3:

```bash
du -sh /var/log/postgresql /var/log/patroni
df -h /var/log
```

**Шаг 6. Переключение роли и рестарт бывшего лидера**

```bash
patronictl -c /etc/patroni/patroni.yml switchover pg16-cluster \
  --leader <old-leader> --candidate <replica-1>
# после подтверждения новой роли:
patronictl -c /etc/patroni/patroni.yml restart pg16-cluster <old-leader> --pending
```

`systemctl restart patroni` на действующем лидере не применять: это незапланированный failover
с разрывом клиентских сессий вне контролируемого окна.

**Шаг 7. Приёмка** — проверки из п. 8.

### Откат

| Шаг | Как откатить | Простой |
| --- | --- | --- |
| Логи Patroni (шаг 2) | убрать секцию `log` из `patroni.yml`, `systemctl reload patroni` | нет |
| Параметры reload (шаг 3) | `patronictl edit-config` на значения из `/root/dcs-config-*.yml`, затем `reload` | нет |
| `logging_collector` (шаги 4, 6) | вернуть `off` через `edit-config`, затем рестарт узлов в том же порядке | рестарт на узел |

До выполнения шага 4 вся конфигурация откатывается без единого рестарта.

---

## 7. Параметры systemd-юнита `patroni.service`

Дополнение (drop-in), а не правка юнита из пакета:
`/etc/systemd/system/patroni.service.d/logging.conf`

| Параметр | Стандартное значение | Новое значение | Комментарий |
| --- | --- | --- | --- |
| `StandardOutput` | `journal` (из `DefaultStandardOutput`) | `journal` | Куда systemd направляет `stdout` процесса Patroni |
| `StandardError` | `inherit` (= `journal`) | `journal` | Куда systemd направляет `stderr` процесса Patroni. Именно сюда попадают строки postmaster, напечатанные до старта сборщика логов |
| `SyslogIdentifier` | имя юнита (`patroni`) | `patroni` | Тег, под которым записи попадают в journal (`journalctl -t patroni`) |

```ini
[Service]
StandardOutput=journal
StandardError=journal
SyslogIdentifier=patroni
```

```bash
systemctl daemon-reload      # не перезапускает процесс
systemctl status patroni
```

Замечания для production:

- `KillMode`, `TimeoutStopSec`, `Restart` **намеренно не меняются**. Они определяют поведение
  при останове кластера: заниженный `TimeoutStopSec` приводит к `SIGKILL` по Patroni до
  завершения shutdown checkpoint — то есть к восстановлению после сбоя при следующем старте.
  Это отдельная тема, не относящаяся к логированию.
- `StandardOutput` оставлен `journal`, а не `null`. После включения `log.dir` и
  `logging_collector` в journal остаются только сообщения systemd и те несколько строк
  postmaster, которые печатаются **до** старта сборщика логов. Именно там видны ошибки
  вида «could not open log file» — при `null` диагностика неудачного старта теряется.
  Это осознанное отступление от «ни одной строки PostgreSQL в journal»: цена — несколько
  строк на старте, выгода — работающий разбор инцидента.
- Ограничить объём journal отдельно, в `/etc/systemd/journald.conf`:
  `SystemMaxUse=2G`, `MaxRetentionSec=30day` (после правки — `systemctl restart systemd-journald`).

---

## 8. Проверка и приёмка

```bash
# 1. В journal — только Patroni и systemd, без строк postmaster
journalctl -u patroni --since "1 hour ago" --no-pager | grep -ci 'checkpoint\|autovacuum\|duration:'   # ожидается 0

# 2. Логи Patroni: файл есть, растёт, права 0640, ротация работает
ls -l /var/log/patroni/
tail -n 20 /var/log/patroni/patroni.log

# 3. Логи PostgreSQL: файл есть, им владеет postgres, права 0640
ls -l /var/log/postgresql/
psql -Atc "SELECT pg_current_logfile();"
tail -n 20 "$(psql -Atc 'SELECT pg_current_logfile()')"

# 4. Фактические значения в СУБД совпадают с таблицей, pending_restart пуст
psql -c "SELECT name, setting, pending_restart FROM pg_settings
         WHERE name LIKE 'log%' OR name IN ('cluster_name','logging_collector')
         ORDER BY name;"

# 5. Состояние кластера: все узлы running/streaming, pending_restart отсутствует
patronictl -c /etc/patroni/patroni.yml list

# 6. Конфигурация в DCS соответствует согласованной
patronictl -c /etc/patroni/patroni.yml show-config

# 7. Удержание и место
systemd-tmpfiles --dry-run --clean /etc/tmpfiles.d/postgresql-log.conf
df -h /var/log && du -sh /var/log/postgresql /var/log/patroni
```

### Мониторинг после внедрения

| Что контролировать | Порог / условие |
| --- | --- |
| Свободное место на разделе с логами | алерт при заполнении 80% |
| Сообщения `could not open log file` / `logging collector` в journal | любое вхождение |
| Суточный прирост `/var/log/postgresql` | рост более чем вдвое от замеренного базового |
| Срабатывание `systemd-tmpfiles-clean.timer` | отсутствие запуска более 48 ч |
| Наличие свежих записей в `patroni.log` | отсутствие записей более 10 мин при живом кластере |
