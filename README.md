# วิธีตั้งค่า PHP-FPM + PGSQL และ NGINX ให้รองรับ Concurrent

## ปรับ PHP-FPM ให้รองรับ Concurrent Connections มากขึ้น
- แก้ไขไฟล์ `www.conf`
```
sudo nano /etc/php/[PHP Version]/fpm/pool.d/www.conf
```

- ตั้งค่าตามต่อไปนี้

```
; ใช้ Dynamic process management
pm = dynamic

; จำนวน process ที่จะเปิดเริ่มต้น
pm.start_servers = 10

; จำนวน process ขั้นต่ำ (idle)
pm.min_spare_servers = 5

; จำนวน process สูงสุด (idle)
pm.max_spare_servers = 30

; จำนวน process สูงสุดทั้งหมด
pm.max_children = 20

; จำนวน request ต่อ worker ก่อน restart
pm.max_requests = 500

; เพิ่ม timeout ป้องกันการ hang
request_terminate_timeout = 60s
request_slowlog_timeout = 5s

; เพิ่ม limit
rlimit_files = 4096
rlimit_core = unlimited

; log file
slowlog = /var/log/[PHP Version]-fpm.slow.log
```

- RAM	ค่าแนะนำ (โดยประมาณ)
```
1 GB	pm.max_children = 10–15
2 GB	pm.max_children = 30–40
4 GB	pm.max_children = 60–80
8 GB+	ปรับตามโหลดจริง (100+ ได้)
```

## ปรับค่าใน php.ini
- แก้ไขไฟล์ตามตัวอย่างต่อไปนี้
```
sudo nano /etc/php/[PHP Version]/fpm/php.ini
```

- แก้ไขตามนี้
```
max_execution_time = 60
memory_limit = 512M
post_max_size = 50M
upload_max_filesize = 50M
max_input_vars = 5000
```

## ติดตั้ง PgBouncer + PHP-FPM
สำหรับ production หรือเว็บที่มี concurrent เยอะแนะนำใช้ PgBouncer ทำเป็น connection pool ระหว่าง PHP-FPM กับ PostgreSQL
- ติดตั้งผ่าน apt:
```
sudo apt install pgbouncer
```
-ไฟล์หลักจะอยู่ที่: 
```
/etc/pgbouncer/pgbouncer.ini
```
- และไฟล์ environment/credential อื่น ๆ เช่น:
```
/etc/pgbouncer/userlist.txt
```
- ตั้งค่า
```
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb user=myuser password=mypass # ตั้งค่า Host, Port, DB Name และ Password ที่ต้องใช้งาน

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid
admin_users = postgres
pool_mode = session
max_client_conn = 1000
default_pool_size = 50
```

- ทำการ Restart PgBouncer
```
sudo systemctl restart pgbouncer
```

- ตรวจสอบสถานะการทำงานของ PGBouncer
```
sudo systemctl status pgbouncer
```


## ปรับ Nginx ให้สอดคล้องกับจำนวน PHP workers
- แก้ไขไฟล์ต่อไปนี้
```
sudo nano /etc/nginx/nginx.conf
```

- แก้ไขตามตัวอย่างนี้
```
worker_processes auto;

events {
    worker_connections 4096;
    multi_accept on;
}

http {
    keepalive_timeout 65;
    client_max_body_size 50M;

    upstream php-handler {
        server unix:/run/php/php5.6-fpm.sock;
        keepalive 256;
    }

    server {
        listen 80;
        server_name yourdomain.com;

        root /var/www/yourapp;
        index index.php;

        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass php-handler;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }
}
```

## ตรวจสอบ Performance
- รันคำสั่งดูสถานะ PHP-FPM:
```
sudo systemctl status php5.6-fpm
sudo service php5.6-fpm reload
```

- ตรวจสอบจำนวน process ที่เปิดอยู่:
```
ps -ef | grep php-fpm | wc -l
```

- ดู real-time usage:
```
top -c | grep php-fpm
```

## Monitoring (Optional)
### สร้าง status page:
- ใน /etc/php/[PHP Version]/fpm/pool.d/www.conf:
```
pm.status_path = /status
```

- ในไฟล์ nginx
```
location ~ ^/(status)$ {
    fastcgi_pass php-handler;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

- เปิดดูที่:
👉 http://yourdomain.com/status


