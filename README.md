# MikroTik Logs Collection Pipeline

Конвейер сбора и анализа логов с роутеров MikroTik: **syslog-ng → Promtail → Loki → Grafana**

## Описание

Данная сборка предоставляет готовую конфигурацию для централизованного сбора, обработки и визуализации логов с устройств MikroTik. Система обеспечивает:

- 📥 **Прием логов** от MikroTik по UDP/TCP на порту 5140
- 🔄 **Обработка** в единый JSON-формат с нормализацией полей
- 📊 **Хранение** в Loki с быстрым поиском по меткам
- 📈 **Визуализация** в Grafana с фильтрацией по устройствам, уровням и пользователям

## Архитектура развертывания

**Рекомендуемая схема:**
- **Сервер 1** (`YOUR_SERVER_IP`): syslog-ng + Promtail + Loki
- **Сервер 2** (`YOUR_GRAFANA_IP`): Grafana (опционально, может быть на том же сервере)

## Архитектура

```
MikroTik → UDP:5140 → syslog-ng → JSON файл → Promtail → Loki → Grafana
```

## Структура
```
├── syslog-ng/                    # Конфигурация syslog-ng
│   ├── conf.d/
│   │   ├── options.conf          # Общие настройки syslog-ng
│   │   └── mikrotik-files.conf   # Источники и назначения для MikroTik
│   └── syslog-ng.conf           # Основной конфиг (базовый)
├── promtail/                     # Конфигурация Promtail
│   └── config.yml               # Настройки для чтения JSON и отправки в Loki
├── mikrotik-logs-howto.md       # Подробная инструкция по развертыванию
└── README.md                    
```

### 1. Требования

**На сервере сбора логов (`YOUR_SERVER_IP`):**
- Linux-хост с доступом к MikroTik по сети
- Установленный **syslog-ng**
- Установленный **Docker** (для Promtail)
- Запущен **Loki** (порт 3100)

**На сервере визуализации (опционально отдельный):**
- Запущен **Grafana** (порт 3000)
- Доступ к Loki серверу по сети

### 2. Установка конфигураций

```bash
# Копируем конфигурации syslog-ng
sudo cp syslog-ng/conf.d/*.conf /etc/syslog-ng/conf.d/

# Копируем конфигурацию Promtail
sudo cp promtail/config.yml /etc/promtail/

# Создаем каталог для логов
sudo mkdir -p /var/log/mikrotik
sudo chown root:adm /var/log/mikrotik
sudo chmod 0755 /var/log/mikrotik
```

### 3. Настройка MikroTik

В веб-интерфейсе MikroTik:

**System → Logging → Actions → Add:**
- Type: `remote`
- Remote Address: `YOUR_SERVER_IP` (замените на IP вашего сервера)
- Remote Port: `5140`
- Protocol: `udp`
- Enable: `BSD Syslog` ✅
- Syslog Facility: `local7`

**System → Logging → Rules → Add:**
- Topics: `info` (или нужные вам)
- Action: созданный `remote`

### 4. Запуск сервисов

```bash
# Перезапуск syslog-ng
sudo systemctl restart syslog-ng
sudo systemctl status syslog-ng

# Запуск Promtail в Docker
docker run -d --name promtail --restart=unless-stopped \
  -p 9081:9081 \
  -v /etc/promtail:/etc/promtail:ro \
  -v /var/lib/promtail:/var/lib/promtail \
  -v /var/log/mikrotik:/var/log/mikrotik:ro \
  grafana/promtail:2.9.3 \
  -config.file=/etc/promtail/config.yml
```

### 5. Проверка работы

```bash
# Проверка приема логов
sudo ss -lunp | grep ':5140'
tail -f /var/log/mikrotik/complex_mikrotik.json

# Проверка Promtail
docker logs promtail --tail=30
```

## Настройка Grafana

1. **Добавить источник данных Loki:**
   - URL: `http://YOUR_SERVER_IP:3100` (замените на IP сервера с Loki)

2. **Создать переменные:**
   - `identity` - устройства MikroTik
   - `level` - уровни логов
   - `actor` - пользователи

3. **Запрос для панели логов:**
```logql
{job="mikrotik", identity=~"$identity", level=~"$level", actor=~"$actor"} |~ "(?i)${search:regex}"
| json
| label_format level="{{ if or (eq .level `err`) (eq .level `error`) }}error{{ else if or (eq .level `crit`) (eq .level `critical`) }}critical{{ else }}{{ .level }}{{ end }}"
| line_format "{{ if eq .level `critical` }}🟥{{ else if eq .level `error` }}🔴{{ else }}  {{ end }} {{.identity}} [{{.level}}] {{.program}} — {{.message}}"
```

![Dashboard](https://github.com/user-attachments/assets/66daa3c3-3ec0-4ddc-bb40-f3237048397a)

## Конфигурация

### syslog-ng

- **options.conf** - общие настройки (DNS, hostname, размер сообщений)
- **mikrotik-files.conf** - источники UDP/TCP 5140 и JSON-вывод

### Promtail

- Читает JSON-файл `/var/log/mikrotik/complex_mikrotik.json`
- Извлекает метки: `identity`, `program`, `level`, `actor`, `srcip`
- Нормализует уровни логов
- Отправляет в Loki

## Мониторинг и отладка

```bash
# Проверка приема UDP
sudo tcpdump -ni any udp and port 5140 -vv -c 5

# Логи Promtail
docker logs promtail --tail=100

# Метрики Promtail
curl -s 127.0.0.1:9081/metrics | grep promtail_sent_bytes_total

# API Loki
curl -Gs "http://YOUR_SERVER_IP:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="mikrotik"}' \
  --data-urlencode 'limit=20'
```

## Ротация логов

Создайте `/etc/logrotate.d/syslog-ng-mikrotik`:

```conf
/var/log/mikrotik/complex_mikrotik.json {
  daily
  rotate 14
  missingok
  notifempty
  compress
  delaycompress
  copytruncate
}
```

## Troubleshooting

### Ошибка "timestamp too new"

Проблема с синхронизацией времени между MikroTik и сервером:

```bash
# На сервере
sudo timedatectl set-ntp true

# На MikroTik
/system ntp client set enabled=yes
/system ntp client servers add address=pool.ntp.org
```

### Пустые переменные в Grafana

- Увеличьте временной диапазон
- Проверьте наличие логов в Loki
- Проверьте правильность выбора Data Source (должно быть Loki)

## Поддержка

Подробная документация доступна в [mikrotik-logs-howto.md](mikrotik-logs-howto.md)
