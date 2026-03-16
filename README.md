Вот полная документация по настройке отказоустойчивого балансировщика нагрузки на базе Nginx для работы с двумя и более виртуальными машинами.
------------------------------
1. Подготовка бэкенд-серверов (ВМ 1 и ВМ 2)
На каждой «рабочей» машине должен быть запущен веб-сервер. В нашем примере они слушают порт 8080.

* Установка: sudo apt install nginx -y
* Идентификация: Чтобы различать серверы, создайте на них разные файлы:
* ВМ 1: echo "Server 1" > /var/www/html/index.html
   * ВМ 2: echo "Server 2" > /var/www/html/index.html
* Порт: Убедитесь, что приложения слушают нужный порт (в конфиге ниже используется 8080).

------------------------------
2. Глобальная настройка балансировщика (/etc/nginx/nginx.conf)
Этот файл отвечает за общие лимиты и формат системных логов.
Конфигурация блока http:

http {
    # 1. Лимиты запросов (Rate Limiting)
    # 10m — память для хранения IP, 30r/s — разрешенная скорость
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=30r/s;

    # 2. Формат логов (чтобы видеть, какой бэкенд ответил)
    log_format upstream_log '$remote_addr - $remote_user [$time_local] '
                           '"$request" $status $body_bytes_sent '
                           'to: $upstream_addr | time: $upstream_response_time';

    # 3. Применяем этот формат ко всем сайтам
    access_log /var/log/nginx/access.log upstream_log;

    include /etc/nginx/mime.types;
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

------------------------------
3. Настройка логики балансировки (/etc/nginx/sites-available/default)
Здесь описывается распределение трафика и обработка ошибок.
Конфигурация:

# Группа серверов (Кластер)
upstream backend_cluster {
    # ip_hash; # Раскомментируйте для закрепления сессий за одной ВМ
    server 192.168.114.146:8080 max_fails=3 fail_timeout=30s;
    server 192.168.114.147:8080 max_fails=3 fail_timeout=30s;
    server 192.168.114.148:8080 backup; # Резервный сервер
}

server {
    listen 80;

    # Страницы ошибок
    error_page 502 503 504 /error.html;
    error_page 429 /too_many_requests.html;

    location / {
        # Применение лимита (burst — размер очереди, nodelay — мгновенный ответ)
        limit_req zone=mylimit burst=50 nodelay;

        # Проброс запроса
        proxy_pass http://backend_cluster;
        
        # Передача реальных данных клиента бэкендам
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Перехват ошибок от бэкендов для показа своих страниц
        proxy_intercept_errors on;
    }

    # Расположение файлов с ошибками
    location = /error.html { root /usr/share/nginx/html; internal; }
    location = /too_many_requests.html { root /usr/share/nginx/html; internal; }
}

------------------------------
4. Создание страниц-заглушек
Создайте HTML-файлы на балансировщике:

sudo echo "<h1>Серверы перегружены или недоступны</h1>" > /usr/share/nginx/html/error.html
sudo echo "<h1>Слишком много запросов. Подождите.</h1>" > /usr/share/nginx/html/too_many_requests.html

------------------------------
5. Применение и проверка
После любых изменений выполняйте команды:

   1. Проверка синтаксиса: sudo nginx -t (должно быть syntax is ok).
   2. Перезапуск: sudo systemctl restart nginx.
   3. Проверка логов в реальном времени: tail -f /var/log/nginx/access.log.
   * Ожидаемый вид: ... to: 192.168.114.146:8080 | time: 0.005.
   
------------------------------
6. Устранение проблем (Траблшутинг)

* Ошибка 502: Проверьте фаервол на бэкендах (sudo ufw allow 8080) и запущен ли там процесс.
* Ошибка 429: Вы слишком часто обновляете страницу (сработал Rate Limit).
* Тормоза: Увеличьте параметр rate в nginx.conf или проверьте параметр time в логах (если он большой — тормозит сама ВМ бэкенда).

Нужно ли добавить в эту документацию раздел по автоматическому выпуску SSL-сертификата (HTTPS) через Certbot?

