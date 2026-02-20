# PostgreSQL Docker Setup

PostgreSQL database server แบบ Production-ready สำหรับใช้งานร่วมกันระหว่างหลายโปรเจค Backend API

## คุณสมบัติ

- PostgreSQL 16 Alpine (เบา, พร้อมใช้งาน production)
- ใช้ network ร่วมกัน `nginx-proxy-gateway-network`
- **รองรับการเชื่อมต่อจาก client ภายนอก (PgAdmin, DBeaver ฯลฯ)**
- จัดเก็บข้อมูลแบบถาวร
- มี health checks
- ปรับแต่ง configuration สำหรับ production
- ปฏิบัติตาม security best practices
- พร้อม logging และ monitoring

## ข้อกำหนดเบื้องต้น

1. สร้าง Docker network ที่ใช้ร่วมกัน:
```bash
docker network create nginx-proxy-gateway-network
```

2. สร้างโฟลเดอร์สำหรับเก็บข้อมูล:
```bash
mkdir -p data init-scripts
```

## การติดตั้ง

1. คัดลอกไฟล์ environment:
```bash
cp .env.example .env
```

2. แก้ไขไฟล์ `.env` และตั้งค่ารหัสผ่านที่ปลอดภัย:
```bash
vim .env
```

3. เริ่มต้น PostgreSQL:
```bash
docker-compose up -d
```

4. ตรวจสอบสถานะ:
```bash
docker-compose ps
docker-compose logs -f postgres
```

## การเชื่อมต่อกับ PostgreSQL

### จาก Docker Containers (Backend APIs)

เพิ่มส่วนนี้ใน `docker-compose.yml` ของโปรเจค Backend API:

```yaml
services:
  your-api:
    # ... config ของ service
    networks:
      - nginx-proxy-gateway-network
    environment:
      DATABASE_URL: postgresql://postgres:your_password@postgres-shared-db:5432/your_db_name

networks:
  nginx-proxy-gateway-network:
    external: true
```

### จาก Client ภายนอก (PgAdmin, DBeaver ฯลฯ)

ใช้การตั้งค่าเหล่านี้ในการเชื่อมต่อ:

- **Host**: `IP-ของ-VPS` หรือ `domain.com`
- **Port**: `5432` (หรือ port ที่กำหนดใน .env)
- **Database**: `postgres` (หรือชื่อ database ที่ต้องการ)
- **Username**: `postgres` (หรือ username ที่ตั้งไว้)
- **Password**: (รหัสผ่านจากไฟล์ .env)

#### ตัวอย่างการเชื่อมต่อด้วย PgAdmin 4:
1. คลิกขวาที่ "Servers" → "Register" → "Server"
2. แท็บ General:
   - Name: `My VPS Database`
3. แท็บ Connection:
   - Host name/address: `123.45.67.89`
   - Port: `5432`
   - Maintenance database: `postgres`
   - Username: `postgres`
   - Password: `your_password`
   - Save password: ✓

#### ตัวอย่างการเชื่อมต่อด้วย DBeaver:
1. New Database Connection → PostgreSQL
2. แท็บ Main:
   - Host: `123.45.67.89`
   - Port: `5432`
   - Database: `postgres`
   - Username: `postgres`
   - Password: `your_password`

## การจัดการฐานข้อมูล

### สร้าง database ใหม่:
```bash
docker exec -it postgres-shared-db psql -U postgres -c "CREATE DATABASE myapp;"
```

### สร้าง user:
```bash
docker exec -it postgres-shared-db psql -U postgres -c "CREATE USER myapp WITH PASSWORD 'password';"
docker exec -it postgres-shared-db psql -U postgres -c "GRANT ALL PRIVILEGES ON DATABASE myapp TO myapp;"
```

### เชื่อมต่อเข้า psql:
```bash
docker exec -it postgres-shared-db psql -U postgres
```

### สำรองข้อมูล (Backup):
```bash
docker exec postgres-shared-db pg_dump -U postgres dbname > backup.sql
```

### กู้คืนข้อมูล (Restore):
```bash
docker exec -i postgres-shared-db psql -U postgres dbname < backup.sql
```

## Init Scripts

วาง SQL scripts ในโฟลเดอร์ `./init-scripts/` ซึ่งจะทำงานอัตโนมัติเมื่อรันครั้งแรก:

ตัวอย่าง `./init-scripts/01-init.sql`:
```sql
CREATE DATABASE app1;
CREATE DATABASE app2;
CREATE USER app1user WITH PASSWORD 'password1';
CREATE USER app2user WITH PASSWORD 'password2';
GRANT ALL PRIVILEGES ON DATABASE app1 TO app1user;
GRANT ALL PRIVILEGES ON DATABASE app2 TO app2user;
```

## การปรับแต่ง Configuration

ปรับแต่งไฟล์ `postgresql.conf` ตามสเปคของเซิร์ฟเวอร์:

- **shared_buffers**: 25% ของ RAM
- **effective_cache_size**: 50-75% ของ RAM
- **maintenance_work_mem**: 5-10% ของ RAM
- **max_connections**: ขึ้นอยู่กับ load ที่คาดการณ์

## ข้อควรระวังด้านความปลอดภัย

1. **ใช้รหัสผ่านที่แข็งแรงเสมอใน production**
2. **ตั้งค่า Firewall (แนะนำ)**: จำกัดการเข้าถึง port PostgreSQL เฉพาะ IP ที่ต้องการ
   ```bash
   # Ubuntu/Debian
   sudo ufw allow from 192.168.1.0/24 to any port 5432
   sudo ufw deny 5432

   # หรืออนุญาตเฉพาะ IP ที่ระบุ
   sudo ufw allow from YOUR_CLIENT_IP to any port 5432
   ```
3. **เปลี่ยน port เริ่มต้น**: แก้ไข `POSTGRES_PORT=5432` ในไฟล์ `.env` เป็น port อื่น
4. **จำกัด IP**: แก้ไขไฟล์ `pg_hba.conf` เพื่อจำกัดการเข้าถึงเฉพาะ IP range ที่ต้องการ:
   ```
   # เปลี่ยน 0.0.0.0/0 เป็น IP range ของคุณ
   host    all    all    203.0.113.0/24    scram-sha-256
   ```
5. พิจารณาใช้ SSL certificates (แก้ไข postgresql.conf)
6. ใช้ user account เฉพาะสำหรับแต่ละ application (ไม่ใช้ postgres superuser)
7. อัปเดตความปลอดภัยและสำรองข้อมูลอย่างสม่ำเสมอ

## การ Monitoring

### ตรวจสอบ performance:
```bash
docker exec -it postgres-shared-db psql -U postgres -c "SELECT * FROM pg_stat_activity;"
```

### ดู slow queries:
```bash
docker exec -it postgres-shared-db psql -U postgres -c "SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;"
```

## การดูแลรักษา

### Vacuum database:
```bash
docker exec postgres-shared-db vacuumdb -U postgres --all --analyze
```

### ตรวจสอบจำนวน connections:
```bash
docker exec postgres-shared-db psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"
```

## การแก้ไขปัญหา

### ดู logs:
```bash
docker-compose logs -f postgres
```

### Restart service:
```bash
docker-compose restart postgres
```

### ตรวจสอบ health:
```bash
docker inspect postgres-shared-db | grep -A 10 Health
```
