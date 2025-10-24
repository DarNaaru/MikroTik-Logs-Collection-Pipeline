
# Сборка логов MikroTik → syslog-ng → Promtail → Loki → Grafana

Конвейер сбора и анализа логов с роутеров MikroTik:
- syslog-ng принимает syslog по UDP/TCP 5140 и пишет строки в единый JSON-файл с единым форматом полей.
- Promtail читает файл, структурирует записи (identity/host/program/level/actor), нормализует уровни и отправляет в Loki.
- Loki хранит логи и даёт быстрый поиск по меткам, а Grafana визуализирует и фильтрует события по устройствам, уровням и пользователям.
Назначение: централизованный аудит изменений, диагностика инцидентов и удобная наблюдаемость за сетью на базе MikroTik.

Пошаговая инструкция **с нуля до дашборда**. Описаны команды, конфиги и проверки.  

- **MikroTik →** UDP **5140** → **syslog-ng** (на `YOUR_SERVER_IP`)
- syslog-ng пишет **один общий JSON-файл**: `/var/log/mikrotik/complex_mikrotik.json`
- **Promtail (Docker)** читает этот файл и отправляет в **Loki**: `http://YOUR_SERVER_IP:3100`
- **Grafana** (на `YOUR_GRAFANA_IP:3000` или том же сервере) показывает логи и даёт фильтры

> **Архитектура:** syslog-ng, Promtail и Loki на одном сервере (`YOUR_SERVER_IP`). Grafana может быть на отдельном сервере (`YOUR_GRAFANA_IP`).

---

## Требования

**На сервере сбора логов (`YOUR_SERVER_IP`):**
- Linux-хост с доступом к MikroTik'ам по сети
- Установлен **syslog-ng**
- Установлен **Docker** (для promtail)
- Запущен **Loki** (порт `3100`)

**На сервере визуализации (опционально отдельный):**
- Запущен **Grafana** (порт `3000`)
- Доступ к Loki серверу по сети

> **Важно:** syslog-ng, Promtail и Loki должны быть на одном сервере для оптимальной производительности. Grafana может быть на отдельном сервере.

---

## 1) Настраиваем syslog-ng (приём логов MikroTik по UDP/5140 и запись в JSON)

1. Каталог для логов:
   ```bash
   sudo mkdir -p /var/log/mikrotik
   sudo chown root:adm /var/log/mikrotik
   sudo chmod 0755 /var/log/mikrotik
   ```

2. Файл `/etc/syslog-ng/conf.d/options.conf` (общие опции):

   ```conf

   options {
     keep-hostname(yes);   # не переписывать hostname
     use-dns(no);          # не делать DNS-резолв
     chain-hostnames(no);
     frac-digits(3);       # миллисекунды в ISODATE
     log-msg-size(65536);  # запас по длине сообщения
   };

   ```

3. Файл `/etc/syslog-ng/conf.d/mikrotik-files.conf` (источник + вывод в 1 файл JSON):
  ```conf

  # Источники логов - приём syslog от MikroTik
  # UDP источник: слушает на всех интерфейсах (0.0.0.0) порт 5140
  # so-rcvbuf(10485760) - буфер 10MB для обработки больших объёмов логов
  source s_mikrotik_udp { syslog(transport(udp) ip(0.0.0.0) port(5140) so-rcvbuf(10485760)); };
  
  # TCP источник: альтернативный протокол для надёжной доставки
  source s_mikrotik_tcp { syslog(transport(tcp) ip(0.0.0.0) port(5140)); };

  # JSON шаблон для структурированного вывода логов
  # format-json автоматически экранирует специальные символы
  template t_json_safe {
    template("$(format-json \
                timestamp=$ISODATE \      # Время в формате ISO 8601 (2024-12-19T10:30:00.123Z)
                identity=${HOST} \        # Имя устройства из заголовка syslog (hostname)
                host=$SOURCEIP \          # IP-адрес отправителя (MikroTik)
                program=${PROGRAM} \      # Программа/сервис (system, user, etc.)
                level=${LEVEL} \          # Уровень важности (info, warn, error, crit)
                message=${MESSAGE} \      # Текст сообщения
              )\n");                      # Перенос строки в конце
  };

# Назначение - запись в единый JSON файл
destination d_mikrotik_json {
  file("/var/log/mikrotik/complex_mikrotik.json"    # Путь к файлу логов
      template(t_json_safe)                         # Используем JSON шаблон
      create-dirs(yes)                              # Создавать каталоги если не существуют
      owner("root") group("adm") perm(0640));       # Права: root:adm, 640 (rw-r-----)
};

# Маршруты логов - связываем источники с назначениями
# UDP трафик → JSON файл
log { source(s_mikrotik_udp); destination(d_mikrotik_json); };
# TCP трафик → JSON файл  
log { source(s_mikrotik_tcp); destination(d_mikrotik_json); };

  ```

4. Проверка и перезапуск:
   ```bash
   sudo syslog-ng -s
   sudo systemctl restart syslog-ng
   sudo systemctl status syslog-ng --no-pager
   sudo ss -lunp | grep ':5140'    # должен слушать UDP 5140
   ```

5. Локальный тест:
   ```bash
   printf '<14>%s TEST testproc[123]: UDP 5140 test\n' "$(date '+%b %e %T')" | nc -u -w1 127.0.0.1 5140
   tail -n 1 /var/log/mikrotik/complex_mikrotik.json
   ```

---

## 2) MikroTik (syslog на YOUR_SERVER_IP:5140/udp)

**/system logging → Actions → Add**  
- Type: `remote`, Remote Address: `YOUR_SERVER_IP`, Remote Port: `5140`, Protocol: `udp`
- Enable: `BSD Syslog` (включает заголовок RFC3164 с hostname → корректная `identity`)
- Syslog Facility: `local7` (можно любой `local0`…`local7`, но держите постоянным)
- Syslog Severity: (по умолчанию пусто — берётся из события)

**/system logging → Rules → Add**  
- Topics: `info` (и/или другие нужные), Action: созданный `remote`

Проверка на сервере:
```bash
sudo tcpdump -ni any udp and port 5140 -vv -c 5
```

> Примечание про BSD Syslog и NAT
>
> - Галочка “BSD Syslog” добавляет RFC3164‑заголовок (hostname, timestamp, facility/severity). С ней syslog-ng берёт имя устройства в поле `${HOST}` (в JSON это `identity`).
> - Если NAT на пути от MikroTik до сервера НЕ настроен (источник виден как реальный IP устройства), галочку можно не ставить — `identity` тогда можно получать из `${SOURCEIP}`. Однако тогда hostname в заголовке отсутствует и часть функций (например, идентификация по имени устройства) будет недоступна.
> - Если NAT есть и исходный адрес подменяется (или трафик идёт через несколько хопов), без BSD Syslog в логах окажется адрес последнего хопа, и все устройства будут выглядеть одинаково. В этом случае обязательно включайте “BSD Syslog”, чтобы hostname из заголовка однозначно идентифицировал устройство.
>
> - Syslog Facility (например, `local7`): это категория (0–23), которая входит в PRI вместе с severity. Выбирайте любую из `local0`…`local7` и держите её постоянной. Так syslog-ng стабильно парсит PRI, корректно извлекает severity, а вы при желании сможете фильтровать по facility на стороне приёмника. По умолчанию удобно использовать `local7`.

---

## 3) Promtail (Docker) — читает JSON и пушит в Loki

1. Каталоги:
   ```bash
   sudo mkdir -p /etc/promtail /var/lib/promtail
   ```

2. Конфиг `/etc/promtail/config.yml`:

   ```yaml

   # Настройки сервера Promtail
   server:
     http_listen_port: 9081    # HTTP API для метрик и статуса
     grpc_listen_port: 0       # gRPC отключен (не используется)

   # Файл позиций - отслеживает где остановился Promtail при чтении логов
   positions:
     filename: /var/lib/promtail/positions.yaml

   # Клиенты - куда отправлять логи (Loki сервер)
   clients:
     - url: http://YOUR_SERVER_IP:3100/loki/api/v1/push

   # Конфигурация сбора логов
   scrape_configs:
     - job_name: mikrotik                    # Имя задачи для идентификации
       static_configs:
         - targets: [localhost]              # Целевой хост (локальный)
           labels:
             job: mikrotik                   # Метка для группировки в Loki
             __path__: /var/log/mikrotik/complex_mikrotik.json  # Путь к лог-файлу

       # Этапы обработки логов (pipeline)
       pipeline_stages:
         # 1. Парсинг JSON - извлекаем поля из JSON логов
         - json:
             expressions:
               timestamp: timestamp          # Время события
               identity: identity            # Имя устройства (hostname)
               program: program              # Программа/сервис
               level: level                  # Уровень важности
               message: message              # Текст сообщения
         
         # 2. Нормализация уровней - приводим к единому формату
         - template:
             source: level                   # Источник - поле level
             # Шаблон: err/error → error, crit/critical → critical, остальное как есть
             template: '{{- $l := ToLower .level -}}{{- if or (eq $l "err") (eq $l "error") -}}error{{- else if or (eq $l "crit") (eq $l "critical") -}}critical{{- else -}}{{$l}}{{- end -}}'
         
         # 3. Создание меток - делаем поля доступными для фильтрации в Loki
         - labels:
             identity:                       # Метка устройства
             program:                        # Метка программы
             level:                          # Метка уровня

         # 4. Извлечение пользователей из сообщений MikroTik
         # Ищем паттерн: "... changed/added by ... :User@IP (CMD)"
         - match:
             selector: '{job="mikrotik"}'    # Применяется только к логам MikroTik
             stages:
               # Регулярное выражение для извлечения пользователя, IP и команды
               - regex:
                   expression: '(?:changed|added) by [^:]+:(?P<actor>[^@[:space:]]+)@(?P<srcip>[0-9.]+) \((?P<cmd>[^)]+)'
               # Создаём метки из извлечённых групп
               - labels:
                   actor:                    # Пользователь (admin, user1, etc.)
                   srcip:                    # IP-адрес пользователя

         # 5. Обработка времени - используем время из лога, а не время получения
         - timestamp:
             source: timestamp               # Поле с временем
             format: RFC3339                 # Формат ISO 8601 (2024-12-19T10:30:00Z)
             
   ```

3. Запуск:

Перезапуск promtail-контейнера

   ```bash
   docker rm -f promtail 2>/dev/null || true
   ```
Смонтирование лог файла внутрь контейнера
   
   ```bash
   docker run -d --name promtail --restart=unless-stopped \
   -p 9081:9081 \
   -v /etc/promtail:/etc/promtail:ro \
   -v /var/lib/promtail:/var/lib/promtail \
   -v /var/log/mikrotik:/var/log/mikrotik:ro \
   grafana/promtail:2.9.3 \
   -config.file=/etc/promtail/config.yml
   ```

4. Проверки:
   ```bash
   docker logs promtail --tail=30
   curl -s http://127.0.0.1:9081/metrics | grep promtail_sent_bytes_total
   ```

---

## 4) Loki — проверка API
```bash
curl -Gs "http://127.0.0.1:3100/loki/api/v1/query_range"   --data-urlencode 'query={job="mikrotik"}'   --data-urlencode 'limit=20'   --data-urlencode "start=$(($(date +%s%N)-5*60*1000000000))"   --data-urlencode "end=$(date +%s%N)"
```

---

## 5) Grafana: источник, переменные, запрос и подсветка

### Источник Loki
Connections → Data sources → Loki → URL: `http://YOUR_SERVER_IP:3100` → Save & Test.

> **Примечание:** Если Grafana на отдельном сервере, замените `YOUR_SERVER_IP` на IP сервера с Loki.

### Переменные (Type: Query, DS: Loki, Multi, Include All, All value = `.*`):
- `identity`: Label values, **Label**=`identity`, Selector `{job="mikrotik"}`
- `level`:    Label values, **Label**=`level`,    Selector `{job="mikrotik"}`
- `actor`:    Label values, **Label**=`actor`,    Selector `{job="mikrotik"}`

*Show on dashboard*: **Label and value**.

#### Подробная настройка переменных (identity / level / actor)

1) Откройте Dashboard → Settings → Variables → Add variable → Type: Query.

2) Поля (пример на `identity`; для `level` и `actor` аналогично, меняется только Label):
- Name: `identity`
- Label (optional): `Хост` (любой отображаемый заголовок)
- Data source: ваш источник Loki (например, `loki-complex-mkt`)
- Query type: `Label values`
- Label/Metric: `identity`
- Stream selector: `{job="mikrotik"}`
- Multi-value: On
- Include All: On
- Custom all value: `.*`
- Show on dashboard: `Label and value`
- Refresh: `On dashboard load` (или `On time range change`, если нужно под диапазон)
- Regex: оставьте пустым (используйте только при необходимости отфильтровать редкие значения)

  Поле поиска Search:
- Select variable type: `Text box`
- Nane: `search`
- Label: `Поиск`

![Variables](https://github.com/user-attachments/assets/3b68ee4c-6a82-41df-9dc8-fdbcba6355fd)

(Пока не вставите query запрос ниже, галочек справа от variables не будет)

3) Кнопка “Preview of values” должна показать реальные значения. Если список пустой:

![Preview of values](https://github.com/user-attachments/assets/e8a6bbf9-5c66-466d-a2b6-b27f38ba077c)

- Проверьте, что в Loki есть логи со стримом `{job="mikrotik"}` и меткой `identity`
- Увеличьте Time range (например, Last 24 hours)
- Убедитесь, что переменная написана точно `identity`

4) Использование в запросах панелей (важно):
- При Multi-value и Include All используйте оператор `=~` (regex):
  `{job="mikrotik", identity=~"$identity"}`
- Если переменная Single-value (Multi выключен) — можно `identity="$identity"`
- Ошибка 404 (пусто) связана с использованием `=` при Multi-value

5) Переменная `$level` — причины пустых данных:
- В окне времени нет нужных уровней (например, только `info`) — увеличьте диапазон
- Значения нормализуются в promtail до `info|error|critical`. В Preview могут отображаться и старые `err/crit`, если они уже в индексе (то уйдут со временем)
- При необходимости добавьте Regex: `^(info|error|critical)$`

6) Переменная `$actor` — появится после того, как promtail извлечёт метку `actor` из сообщений вида
   “changed/added by … :User@IP (CMD)”. Проверьте в Explore:
   `{job="mikrotik"} | json | line_format "{{.actor}}"`

7) Быстрые проверки наличия меток в Loki:
```bash
curl -s 'http://YOUR_SERVER_IP:3100/loki/api/v1/label/identity/values'
curl -s 'http://YOUR_SERVER_IP:3100/loki/api/v1/label/level/values'
curl -s 'http://YOUR_SERVER_IP:3100/loki/api/v1/label/actor/values'
```


### Панель Logs — запрос (LogQL)
```logql
{job="mikrotik", identity=~"$identity", level=~"$level", actor=~"$actor"} |~ "(?i)${search:regex}"
| json
| label_format level="{{ if or (eq .level `err`) (eq .level `error`) }}error{{ else if or (eq .level `crit`) (eq .level `critical`) }}critical{{ else }}{{ .level }}{{ end }}"
| line_format "{{ if eq .level `critical` }}🟥{{ else if eq .level `error` }}🔴{{ else }}  {{ end }} {{.identity}} [{{.level}}] {{.program}} — {{.message}}"
```
![Test](https://github.com/user-attachments/assets/b101ac6b-9595-478c-b118-612224e3d126)

## 6) Эксплуатация и отладка

- Приём UDP: `sudo ss -lunp | grep 5140`, `sudo tcpdump -ni any udp and port 5140 -vv -c 10`
- Promtail: `docker logs promtail --tail=100`, `curl -s 127.0.0.1:9081/metrics | head`
- Файл логов: `tail -f /var/log/mikrotik/complex_mikrotik.json`
- Перезапустить promtail: `docker restart promtail`

### Ротация лога `/var/log/mikrotik/complex_mikrotik.json`

Создайте `/etc/logrotate.d/syslog-ng-mikrotik`:
```conf

# Конфигурация ротации логов MikroTik
/var/log/mikrotik/complex_mikrotik.json {
  daily                    # Ротация каждый день в 00:00
  rotate 14                # Хранить 14 архивных файлов (2 недели)
  missingok                # Не выдавать ошибку если файл отсутствует
  notifempty               # Не ротировать пустые файлы
  compress                 # Сжимать старые файлы (gzip)
  delaycompress            # Сжимать не сразу, а на следующий день
  copytruncate             # Копировать файл и обрезать оригинал (без перезапуска syslog-ng)
}

```
Проверка: `sudo logrotate -d /etc/logrotate.conf`, принудительно: `sudo logrotate -f /etc/logrotate.d/syslog-ng-mikrotik`.

---

Готово. Теперь у вас стабильный поток логов MikroTik → Loki с фильтрацией по **Хост/Уровень/Пользователь** в Grafana.

---

## Troubleshooting

### 1) Ошибка в promtail: final error sending batch … HTTP 400 … timestamp too new

Симптом: в логах promtail видите `status=400 … entry … has timestamp too new`. Это значит, что метка времени из лога (берётся из поля `timestamp` в JSON) опережает текущее время Loki.

Причина: рассинхрон часов (чаще всего на MikroTik) или неверный Time Zone. Loki по умолчанию отвергает «будущие» записи.

Что сделать (рекомендуется):
- На сервере (YOUR_SERVER_IP):
  ```bash
  timedatectl
  date -u
  sudo apt-get install -y chrony
  sudo timedatectl set-ntp true
  chronyc sources -v
  ```
- На MikroTik:
  ```
  /system clock print
  /system clock set time-zone-name=<ваш_TZ>
  /system ntp client set enabled=yes
  /system ntp client servers add address=pool.ntp.org
  ```
- После синхронизации перезапустить promtail:
  ```bash
  docker restart promtail
  ```

Временные обходные пути (не вместо NTP, а на время):
- Убрать стадию `timestamp` в promtail (тогда будет использоваться время приёма):
  ```yaml
  # - timestamp:
  #     source: timestamp
  #     format: RFC3339
  ```
  Перезапустите promtail.
- Увеличить допустимый «забег вперёд» в Loki (если контролируете конфиг Loki):
  ```yaml
  limits_config:
    max_future_time: 10m
  ```
  Перезапустить Loki. Использовать с осторожностью — только для небольшого дрейфа.
  
---

## 7) Автозапуск и постоянный приём syslog

1) Включить автозагрузку и запустить сейчас:
```bash
sudo systemctl enable --now syslog-ng
```

2) Открыть порт 5140 на фаерволе (выберите ваш вариант):
- UFW (Ubuntu/Debian):
```bash
sudo ufw allow 5140/udp
sudo ufw allow 5140/tcp
```
4) Проверка после перезагрузки хоста:
```bash
sudo systemctl reboot
# затем
sudo ss -lunp | grep ':5140'
sudo systemctl status syslog-ng --no-pager
```

5) Promtail (Docker) запускается с `--restart=unless-stopped` и поднимется автоматически. Проверка:
```bash
docker ps | grep promtail
docker logs promtail --tail=50
```
