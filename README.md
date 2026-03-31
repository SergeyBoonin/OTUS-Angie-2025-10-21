# Домашняя работа OTUS-Angie-2025-10-21 OTUS_Angie_21_Защита_от_DoS-атак,_ограничение_доступа

#### 1. Для запущенного ранее приложения найдите наиболее подверженные атаке location.
Это будет **location ~ \.php$** которая отправляет запросы в AppTier для динамической обработки. 

#### 2. Настройте ограничение частоты запросов.

В файле [angie/http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-21/blob/main/angie/http.d/default.conf) в контексте **http**:
```
    # Allocating shared memory for the 'limit_conn' module
    limit_conn_zone $binary_remote_addr   zone=limit_conn_binary_remote_addr:10m;

    # Allocating shared memory for the 'limit_req' module
    limit_req_zone  $binary_remote_addr   zone=limit_req_binary_remote_addr:10m    rate=1r/s;
```
В файле [angie/http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-21/blob/main/angie/http.d/default.conf) в соответствующем контексте **location**:
```
        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass wordpress:9000;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;

            limit_conn limit_conn_binary_remote_addr 5;
            limit_req zone=limit_req_binary_remote_addr burst=5 nodelay;

            status_zone https_wordpress_.php;
        }
```

#### 3. Запустите сервис fail2ban для автоматической блокировки атакующих на основе частоты запросов.

В файле [docker-compose.yml](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-21/blob/main/docker-compose.yml)
```
  fail2ban:
    image: crazymax/fail2ban:latest
    container_name: fail2ban
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - /var/log:/var/log:ro  # Mount the host's log directory (read-only)
      - fail2ban:/data # Persist Fail2Ban data
    environment: # Or use an env_file
      TZ: "UTC" # Set your timezone
      F2B_BANTIME:  60 #86400 # Ban for 24h
      F2B_MAXRETRY:  3
      F2B_FINDTIME: 10 #600 # 10 minutes
      # Add other jails here
      F2B_JAILS: "nginx-http-auth"
      F2B_NGINX_HTTP_AUTH_ENABLED: "true"
      F2B_NGINX_HTTP_AUTH_LOGPATH: "/var/log/angie/error.log"
```
По-хорошему следовало бы поглубже проработать эту часть и вытащить сюда и другие jails.

Вектор ясен, пусть это будет TODO! ;)

#### 4. Настройте защищенный доступ в любой выбранный location с HTTP-авторизацией и ограничением доступа по IP.

Пользовательские данные в файле [angie/http.d/htpasswd](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-21/blob/main/angie/http.d/htpasswd)
```
user:$apr1$hNDTTHI5$NnUscxV7FqeaWxR35qENb.
```

В файле [angie/http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-21/blob/main/angie/http.d/default.conf) в соответствующем контексте **location**:
```
    location /console/ {

        satisfy any;

        # Только локальный анонимный доступ
        allow 127.0.0.1;
        allow ::1;
        deny all;

        # или, если не локальный, то по auth_basic
        auth_basic           "Identify yourself!";
        auth_basic_user_file /etc/angie/http.d/htpasswd;
```
