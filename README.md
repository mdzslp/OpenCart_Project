# OpenCart_Project
Отчёт о выполнении проектной работы по настройке мониторинга инфраструктуры OpenCart

В рамках проектной работы была развёрнута и настроена система мониторинга для веб-приложения OpenCart, работающего на стеке Nginx + PHP-FPM + MySQL. Инфраструктура развёрнута на виртуальной машине VMBox (Ubuntu 22.04), мониторинг и логгирование реализовано с использованием Prometheus, Alertmanager, Grafana и Loki. OpenCart было развернуто на VM.

2. Развёртывание инфраструктуры
2.1. Настройка виртуальной машины.

ОС: Ubuntu 22.04 LTS.


2.2. Установка OpenCart
Установлены зависимости:

bash
sudo apt update && sudo apt install -y nginx mysql-server php-fpm php-mysql php-curl php-gd php-mbstring php-xml php-zip unzip
Настроен Nginx для OpenCart (конфиг).

Развёрнута база данных MySQL:

sql
CREATE DATABASE opencart;
CREATE USER 'opencart'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON opencart.* TO 'opencart'@'localhost';
FLUSH PRIVILEGES;
Установлен OpenCart из официального архива.

3. Настройка мониторинга
3.1. Развёртывание Prometheus + Alertmanager
Prometheus запущен в Docker с кастомным конфигом:

yaml
scrape_configs:
  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
  - job_name: "opencart"
    metrics_path: "/metrics"
    static_configs:
      - targets: ["opencart:80"]
Alertmanager настроен на отправку уведомлений в Telegram (конфиг):

yaml
route:
  receiver: 'telegram'
receivers:
  - name: 'telegram'
    telegram_configs:
      - bot_token: "123456:ABC-DEF..."
        chat_id: 78901234
3.2. Настройка Grafana + Loki
Grafana развёрнута в Docker и подключена к:

Prometheus (источник данных: http://prometheus:9090).

Loki (источник данных: http://loki:3100).

Loki настроен для сбора логов Nginx и приложения (конфиг):

yaml
server:
  http_listen_port: 3100
storage_config:
  filesystem:
    directory: /tmp/loki/chunks
3.3. Сбор логов через Promtail
Настроен конфиг для отправки логов в Loki:

yaml
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: nginx
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          __path__: /var/log/nginx/*.log
4. Визуализация в Grafana
4.1. Дашборды
Мониторинг сервера:

CPU, RAM, Disk I/O (метрики Node Exporter).

Запросы к Nginx (rate(nginx_http_requests_total[5m])).

Логи OpenCart:

Фильтрация ошибок в Loki ({job="nginx"} |~ "error").

Алерты:

Уведомления о высокой загрузке CPU (node_load1 > 2).

Ошибки 5xx в Nginx (probe_http_status_code{status=~"5.."}).

4.2. Примеры графиков
Использование CPU:

promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
Свободная память:

promql
node_memory_MemFree_bytes / node_memory_MemTotal_bytes * 100
5. Алертинг
Настроены уведомления в Telegram:

При высокой загрузке сервера.

При недоступности OpenCart (HTTP-проверка через Blackbox Exporter).

При ошибках в логах (через Loki Alerting).

Пример алерта:

🚨 CRITICAL: High CPU Load (95%) on opencart-server
📊 Current: 95.2% | Threshold: 90%
🕒 Time: 2024-03-20 14:30:00
6. Проблемы и их решение
Проблема	Решение
Loki не собирал логи	Проверка прав доступа к /var/log/nginx/ в Promtail.
Grafana не видела Prometheus	Указан неверный URL (http://prometheus:9090 вместо localhost).
Ошибки 502 в Nginx	Настройка корректного fastcgi_pass в PHP-FPM.
7. Итоги
Развёрнута отказоустойчивая система мониторинга.

Настроены дашборды в Grafana для анализа метрик и логов.

Реализован алертинг в Telegram при проблемах.

Приложения
Конфиги Nginx, Prometheus, Alertmanager, Loki

Скриншоты Grafana

Пример алертов в Telegram
