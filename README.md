## สรุปโปรเจ็ค

โปรเจ็คนี้เป็น Laravel (เวอร์ชัน 12 ตามชื่อโฟลเดอร์) ที่รันด้วย Docker Compose เพื่อให้สามารถพัฒนาแบบมีสภาพแวดล้อมใกล้เคียงการใช้งานจริงได้ง่าย ๆ บริการหลักได้แก่ PHP-FPM (app), Nginx, MySQL, Redis, phpMyAdmin และ MailHog

## โครงสร้างโปรเจ็ค (สำคัญ)

- `docker-compose.yaml` - คอนฟิกบริการ Docker ทั้งหมด
- `Dockerfile` - สร้าง image ของบริการ `app` (PHP / PHP-FPM)
- `nginx/` - การตั้งค่า Nginx (เช่น `nginx/conf/app.conf`)
- `mysql/` - โฟลเดอร์เก็บข้อมูลของ MySQL (volume)
- `redis/` - โฟลเดอร์เก็บข้อมูลของ Redis (volume)
- `src/` - โค้ดของ Laravel (โฟลเดอร์โปรเจ็คจริง)
  - `artisan` - CLI ของ Laravel
  - `composer.json`, `package.json`, `vite.config.js` - ไฟล์จัดการ dependency และ build
  - `.env.example` - ไฟล์ตัวอย่าง environment (คัดลอกเป็น `.env` ก่อนใช้งาน)
- `public/` - entrypoint ของเว็บ
- `resources/`, `routes/`, `app/`, `config/` - โครงสร้างมาตรฐานของ Laravel
- `tests/` - unit/integration tests

> หมายเหตุ: โครงสร้างข้างต้นสรุปเฉพาะส่วนที่สำคัญสำหรับการตั้งค่าและการพัฒนา

## บริการที่รัน (จาก `docker-compose.yaml`)

- app (PHP-FPM) - รัน PHP/Laravel
- nginx - เว็บเซิร์ฟเวอร์ (mapping พอร์ตภายนอก 8100 -> 80 ภายในคอนเทนเนอร์)
- db (MySQL 8.0) - พอร์ตแมป 3308 -> 3306 (host:3308)
- phpmyadmin - พอร์ต 8200 (สำหรับเข้าจัดการฐานข้อมูล)
- redis - caching/session store
- mailhog - สำหรับดูเมลในสภาพแวดล้อมพัฒนา (Web UI: 8025)


พอร์ตที่เปิดโดยค่าเริ่มต้น

- เว็บแอพ (Nginx): http://localhost:8100
- phpMyAdmin: http://localhost:8200 (user: `admin`, password: `1234` ตาม `docker-compose.yaml`)
- MySQL (จาก host): 127.0.0.1:3308 (user: `admin`, password: `1234`)
- MailHog UI: http://localhost:8025 (SMTP: 1025)

## ความสัมพันธ์การทำงาน (สั้นๆ)

- `nginx` ทำหน้าที่เป็นเว็บเซิร์ฟเวอร์และส่งคำขอไปยัง `app` ที่รันด้วย PHP-FPM
- `app` ติดต่อกับ `db` (MySQL) เพื่อเก็บข้อมูล และ `redis` สำหรับ caching/session
- `phpmyadmin` ใช้สำหรับดู/แก้ไขฐานข้อมูลเมื่อพัฒนา
- `mailhog` แคปเจอร์อีเมลจากแอพเพื่อดูใน UI แทนการส่งจริง

## การติดตั้ง (บนเครื่องพัฒนาที่มี macOS)

สิ่งที่ต้องติดตั้งล่วงหน้า

- Docker Desktop (มี Docker Engine และ Docker Compose)
- Git

ขั้นตอนการตั้งค่า (แบบรวดเร็ว)

1. โคลนโปรเจ็ค

```bash
git clone <repository-url> laravel12-docker
cd laravel12-docker
```

2. คัดลอกไฟล์ environment (ภายในโฟลเดอร์ `src`)

```bash
cp src/.env.example src/.env
# แก้ไขค่าใน src/.env ให้ตรงกับความต้องการ (เช่น DB_HOST, DB_PORT ถ้าต้องการเปลี่ยน)
```

3. สตาร์ท Docker Compose (สร้าง image และรันคอนเทนเนอร์)

```bash
docker compose up -d --build
```

4. ติดตั้ง dependencies ของ PHP (ใน container `app`) และตั้งค่าพื้นฐาน

```bash
# รันคำสั่ง composer ภายใน container app
docker compose exec app composer install --no-interaction --prefer-dist

# สร้าง application key
docker compose exec app php artisan key:generate

# รัน migration และ seed (ถ้ามี)
docker compose exec app php artisan migrate --force
docker compose exec app php artisan db:seed --force
```

5. ติดตั้ง dependencies ฝั่ง JS และ build (option)

# Option A: รันบนเครื่อง host (ถ้าต้องการ)
```bash
cd src
npm install
npm run dev   # หรือ npm run build สำหรับ build production
```

# Option B: ถ้า Docker image ของ `app` มี Node.js ติดตั้งไว้ สามารถรันใน container ได้
```bash
docker compose exec app bash -c "cd /var/www && npm install && npm run dev"
```

6. เข้าเว็บแอพ

เปิดเบราว์เซอร์ที่: http://localhost:8100

## คำสั่งที่เป็นประโยชน์

- ดูล็อกของคอนเทนเนอร์ `app`:
```bash
docker compose logs -f app
```
- เข้า shell ของคอนเทนเนอร์ `app`:
```bash
docker compose exec app bash
```
- ใช้ php artisan ภายในคอนเทนเนอร์ (ตัวอย่าง)
```bash
docker compose exec app php artisan migrate
docker compose exec app php artisan tinker
```

## คำอธิบายเชิงลึกของไฟล์สำคัญ

### Dockerfile (สรุปทีละบรรทัด)
- FROM php:8.2-fpm — ใช้ PHP-FPM base image
- RUN docker-php-ext-install bcmath pdo_mysql — ติดตั้ง PHP extensions ที่จำเป็น (เช่น pdo_mysql)
- RUN apt-get update && apt-get install -y git zip unzip — ติดตั้งเครื่องมือพื้นฐาน
- ติดตั้ง Node.js (จาก NodeSource) และ `nodejs` เพื่อให้สามารถรัน npm ใน container
- COPY --from=composer:latest /usr/bin/composer /usr/bin/composer — คัดลอก composer binary มาจาก official image
- WORKDIR /var/www — กำหนด working dir ให้ตรงกับโฟลเดอร์ที่แมปจาก host
- EXPOSE 9000 — php-fpm จะฟังบน port 9000 และ `nginx` จะเชื่อมต่อผ่าน port นี้

หมายเหตุ: image นี้ออกแบบมาสำหรับพัฒนา (รวม composer + node) ถ้าต้องการ production ควรปรับเป็น multi-stage build และตัดเครื่องมือที่ไม่จำเป็นออก

### Nginx config (`nginx/conf/app.conf`)
- `root /var/www/;` — ชี้ไปยังโค้ดใน container
- `try_files $uri $uri/ /index.php?$query_string;` — ให้ Laravel handle routing
- `fastcgi_pass app:9000;` — ส่งคำขอ PHP ไปที่ container `app` พอร์ต 9000

ถ้าต้องการเพิ่ม header, gzip หรือ security rules ให้แก้ไฟล์นี้แล้ว `docker compose restart nginx`

## ตัวอย่าง `.env` (แนะนำสำหรับ development)

คัดลอกจาก `src/.env.example` เป็น `src/.env` แล้วปรับค่าเช่นด้านล่าง:

```env
APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:GENERATED_BY_ARTISAN
APP_DEBUG=true
APP_URL=http://localhost:8100

LOG_CHANNEL=stack
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel12db
DB_USERNAME=admin
DB_PASSWORD=1234

BROADCAST_DRIVER=log
CACHE_DRIVER=redis
QUEUE_CONNECTION=sync
SESSION_DRIVER=redis
SESSION_LIFETIME=120

REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS=dev@example.com
MAIL_FROM_NAME="${APP_NAME}"
```

คำแนะนำสำคัญ:
- `DB_HOST=db` (ใช้ชื่อ service ภายใน Docker network)
- หากต้องการเชื่อมจากเครื่อง host ให้ใช้ `127.0.0.1:3308`

## ตัวอย่าง: เพิ่ม `worker` service ใน `docker-compose.yaml`
หากต้องการรัน queue worker แยกจาก web app ให้เพิ่ม service ดังนี้ (snippet):

```yaml
  worker:
    build:
      context: .
      dockerfile: Dockerfile
    image: laravel12-worker
    container_name: laravel12_worker
    command: php /var/www/artisan queue:work --sleep=3 --tries=3 --timeout=60
    volumes:
      - ./src:/var/www
    depends_on:
      - db
      - redis
    networks:
      - web_network
```

ข้อดี: แยก process ช่วยให้ scale และรีสตาร์ท worker โดยไม่กระทบ PHP-FPM

## Troubleshooting (ปัญหาที่พบบ่อย)

- Permission error (storage / bootstrap/cache)
  ```bash
  docker compose exec app chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache
  docker compose exec app chmod -R 775 /var/www/storage /var/www/bootstrap/cache
  ```
- Composer memory exhausted
  ```bash
  docker compose exec app bash -c "COMPOSER_MEMORY_LIMIT=-1 composer install --no-interaction --prefer-dist"
  ```
- MySQL ไม่เชื่อมต่อ
  - ตรวจสอบว่า `DB_HOST=db` และ container `db` รันอยู่ (`docker compose ps`)
  - หากเชื่อมจาก host ให้ใช้ `127.0.0.1:3308`
  - ดู log: `docker compose logs db`
- Port conflict (8100/8200/8025 ถูกใช้)
  - แก้ไฟล์ `docker-compose.yaml` เพื่อเปลี่ยนพอร์ตหรือปิดบริการที่ชนกันบน host
- phpMyAdmin เข้าระบบไม่ได้
  - ตรวจสอบ `PMA_HOST` และ credential ใน `docker-compose.yaml`

## Git / .gitignore แนะนำ

- อย่า commit `src/.env` (เก็บในเครื่องแต่ละคน)
- เพิ่มรายการต่อไปนี้ใน `.gitignore` (root หรือ `src/.gitignore` ตามโครงสร้าง):
```
/mysql/data/
/redis/data/
/vendor/
/node_modules/
/src/.env
/storage/
/public/storage
```

## คำสั่งที่เป็นประโยชน์ (สรุป)

- สร้าง/สตาร์ท: `docker compose up -d --build`
- หยุด: `docker compose down`
- เข้า shell app: `docker compose exec app bash`
- ติดตั้ง composer: `docker compose exec app composer install`
- สร้าง key: `docker compose exec app php artisan key:generate`
- รัน migration: `docker compose exec app php artisan migrate --force`
- ดู logs: `docker compose logs -f app`

## Tips พัฒนา

- ใช้ `php artisan storage:link` เพื่อเชื่อม `public/storage`
- พิจารณาเพิ่ม Xdebug ใน image ถ้าต้องการ debug แบบ step-by-step
- สร้าง `Makefile` หรือ `scripts/` เพื่อรวมคำสั่งที่ใช้บ่อย เช่น `make up`, `make install`, `make test`

---
ไฟล์ README นี้อัปเดตเพื่อรวมคำอธิบายเชิงลึกของ `Dockerfile`, `nginx` config, ตัวอย่าง `.env`, คำสั่งติดตั้ง/รัน และแนวทางแก้ไขปัญหาที่พบบ่อย ถ้าต้องการให้ผม:

- เพิ่มตัวอย่าง `.env` ที่มีค่าเริ่มต้นแบบปลอดภัย (mask secrets)
- เพิ่ม workflow สำหรับ CI/CD (GitHub Actions)
- รัน `docker compose up -d --build` แล้วสรุปสถานะคอนเทนเนอร์ให้

ให้ผมดำเนินการต่อแบบไหนบอกได้เลย
