# 시놀로지 설정 가이드 v2.1: MariaDB + WordPress + NPM + Redis + Cloudflare

## 📌 개요 및 주요 전제
- **기반**: Ubuntu Server + Docker, `docker run` 기반 설치 (compose 미사용)
- **데이터**: `/srv/egng/서비스명` 경로 통일, 외부 디스크는 백업용만
- **네트워크**: `egng_net` 브리지, 고정 IP 할당
- **재시작**: 모든 컨테이너 `--restart=always` 설정
- **순서**: DB → Redis → WordPress → phpMyAdmin → NPM 순으로 기동
- **Cloudflare**: 초기 DNS only → 테스트 후 Proxy 모드 전환

---

## 1. Docker 네트워크 구성 및 고정 IP

### 1.1 네트워크 생성
```bash
docker network create \
  --subnet=192.168.100.0/24 \
  --gateway=192.168.100.1 \
  egng_net
```

### 1.2 IP 할당 계획
| 서비스 | 고정 IP | 설명 |
|--------|---------|------|
| NPM | 192.168.100.2 | 프록시 관리자 |
| phpMyAdmin | 192.168.100.3 | DB 관리 도구 |
| Redis | 192.168.100.9 | 캐시 서버 |
| MariaDB | 192.168.100.11 | 데이터베이스 |
| WordPress000-005 | 192.168.100.100-105 | 워드프레스 사이트들 |

**IP 대역 규칙**: 시스템 도구(2-19), WordPress(100번대), 향후 확장 가능

---

## 2. MariaDB (단일 서버)

```bash
sudo mkdir -p /srv/egng/mariadb
sudo chown -R 999:999 /srv/egng/mariadb
sudo chmod -R 700 /srv/egng/mariadb

sudo docker run -d --name mariadb \
  --restart=always \
  --network egng_net \
  --ip 192.168.100.11 \
  -v /srv/egng/mariadb:/var/lib/mysql \
  -e MARIADB_ROOT_PASSWORD="rootpass" \
  mariadb:10.11
```

### 데이터베이스 및 사용자 생성
```bash
sudo docker exec -it mariadb mariadb -u root -p

CREATE USER 'wpuser'@'%' IDENTIFIED BY 'el2l3heekw!';
GRANT ALL PRIVILEGES ON *.* TO 'wpuser'@'%';

CREATE DATABASE wordpress000 CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE wordpress001 CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
-- wordpress002~005 필요한 만큼 추가

FLUSH PRIVILEGES;
```

---

## 3. phpMyAdmin (DB 관리 도구)

```bash
sudo docker run -d --name phpmyadmin \
  --restart=always \
  --network egng_net \
  --ip 192.168.100.3 \
  -e PMA_HOST=192.168.100.11 \
  -e PMA_PORT=3306 \
  -p 8080:80 \
  phpmyadmin/phpmyadmin
```
**접속**: http://[서버IP]:8080

---

## 4. WordPress 컨테이너 (예: wordpress001)

```bash
sudo mkdir -p /srv/egng/wp001
sudo chown -R 33:33 /srv/egng/wp001

sudo docker run -d --name wp001 \
  --restart=always \
  --network egng_net \
  --ip 192.168.100.101 \
  -v /srv/egng/wp001:/var/www/html \
  -e WORDPRESS_DB_HOST="192.168.100.11" \
  -e WORDPRESS_DB_USER="wpuser" \
  -e WORDPRESS_DB_PASSWORD="el2l3heekw!" \
  -e WORDPRESS_DB_NAME="wordpress001" \
  wordpress:latest
```
**권장**: 설치 후 wp-config.php에서 `$table_prefix = 'wp001_';`로 변경

---

## 5. Redis 캐시 서버 구성

```bash
sudo mkdir -p /srv/egng/redis

sudo docker run -d --name redis \
  --restart=always \
  --network egng_net \
  --ip 192.168.100.9 \
  -v /srv/egng/redis:/data \
  redis:alpine \
  redis-server --appendonly yes
```

### WordPress Redis 연동
각 WordPress에 Redis Object Cache 플러그인 설치 후:
```php
// wp-config.php에 추가
define('WP_REDIS_HOST', '192.168.100.9');
define('WP_REDIS_PORT', 6379);
define('WP_CACHE', true);
```

---

## 6. Nginx Proxy Manager (NPM)

```bash
sudo mkdir -p /srv/egng/npm/data /srv/egng/npm/letsencrypt

sudo docker run -d --name npm \
  --restart=always \
  --network egng_net \
  --ip 192.168.100.2 \
  -p 80:80 -p 81:81 -p 443:443 \
  -v /srv/egng/npm/data:/data \
  -v /srv/egng/npm/letsencrypt:/etc/letsencrypt \
  jc21/nginx-proxy-manager:latest
```
**접속**: http://[서버IP]:81

---

## 7. 서비스 재시작 정책 및 기동 순서

### 7.1 시작 스크립트 생성
```bash
sudo tee /usr/local/bin/start-all.sh << 'EOF'
#!/bin/bash
echo "EGNG 서비스 순차 시작..."

# 1. MariaDB 먼저 시작
docker start mariadb
echo "MariaDB 시작 대기 중..."
sleep 15

# 2. Redis 시작
docker start redis
echo "Redis 시작 완료"
sleep 5

# 3. WordPress 컨테이너들 시작
echo "WordPress 컨테이너들 시작..."
docker start wp000 wp001 wp002 wp003 wp004 wp005 2>/dev/null || true
sleep 10

# 4. phpMyAdmin 시작
docker start phpmyadmin
sleep 3

# 5. NPM 마지막 시작
docker start npm
echo "모든 EGNG 서비스 시작 완료!"
EOF

sudo chmod +x /usr/local/bin/start-all.sh
```

### 7.2 systemd 서비스 등록
```bash
sudo tee /etc/systemd/system/egng-services.service << 'EOF'
[Unit]
Description=EGNG Docker Services
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/start-all.sh
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable egng-services.service
```

---

## 8. 백업 및 복구 전략

### 8.1 백업 계획 (중요도별)

| 백업 항목 | 방식 | 주기 | 목적 |
|-----------|------|------|------|
| 시스템 전체 (VM) | Active Backup for Business | 매일 1회 | OS+구성+데이터 통합 복구 |
| MariaDB 데이터베이스 | mariadb-dump | 하루 1~2회 | 장애 시 빠른 복원 |
| WordPress 파일 | rsync | 하루 3회 | 변경된 파일 빠른 보호 |
| WordPress 파일 | tar | 주 1회 | 주간 기준 스냅샷 보존 |
| Redis | AOF (자동 저장) | 상시 | 운영 중 장애 복구 대비 |

### 8.2 자동 백업 스크립트
```bash
sudo tee /usr/local/bin/backup-egng.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/mnt/backup/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "백업 시작: $(date)"

# 1. MariaDB 전체 백업
docker exec mariadb mariadb-dump --all-databases --single-transaction \
  -u root -p"rootpass" | gzip > "$BACKUP_DIR/mariadb_full.sql.gz"

# 2. WordPress 사이트별 파일 백업
for wpdir in /srv/egng/wp*; do
    if [ -d "$wpdir" ]; then
        wp_name=$(basename "$wpdir")
        tar -czf "$BACKUP_DIR/${wp_name}_files.tar.gz" -C "$wpdir" .
    fi
done

# 3. NPM 설정 백업
tar -czf "$BACKUP_DIR/npm_config.tar.gz" -C /srv/egng/npm .

# 4. Redis AOF 백업
cp /srv/egng/redis/appendonly.aof "$BACKUP_DIR/" 2>/dev/null || echo "Redis AOF 없음"

# 5. 30일 이상 된 백업 삭제
find /mnt/backup -name "2*" -mtime +30 -exec rm -rf {} + 2>/dev/null

echo "백업 완료: $BACKUP_DIR"
EOF

sudo chmod +x /usr/local/bin/backup-egng.sh

# crontab 등록
sudo tee -a /etc/crontab << 'EOF'
# EGNG 백업 스케줄
0 2 * * * root /usr/local/bin/backup-egng.sh >> /var/log/backup.log 2>&1
0 8,16 * * * root /usr/local/bin/backup-egng.sh >> /var/log/backup.log 2>&1
EOF
```

### 8.3 복구 테스트 (월 1회 권장)
```bash
# 1. mariadb-dump 백업본을 새 MariaDB 컨테이너에 임시 복원하여 연결 테스트
# 2. /srv/egng/wpXXX 디렉터리를 tar 백업본에서 꺼내어 마운트 후 WordPress 기동 확인
# 3. Active Backup 스냅샷을 가상 머신으로 임시 복원하여 전체 시스템 정상 작동 확인
```

---

## 9. 모니터링 및 유지보수 팁

### 9.1 일상 점검 명령어
```bash
# 서비스 상태 확인
docker ps -a

# 특정 컨테이너 로그 확인
docker logs mariadb --tail 50
docker logs npm --tail 50

# 디스크 사용량 체크
df -h /srv/egng
du -sh /srv/egng/*

# 서비스 연결 확인
redis-cli -h 192.168.100.9 ping
docker exec mariadb mariadb -h localhost -u root -p -e "SELECT 1"

# NPM 인증서 갱신 주기 체크 (기본 90일)
```

---

## 10. 이메일 시스템 설정

### 10.1 Gmail SMTP 설정 예시
| 항목 | 값 |
|------|-----|
| SMTP 서버 | smtp.gmail.com |
| 포트 | 587 (STARTTLS) 또는 465 (SSL) |
| 사용자 | youraccount@gmail.com |
| 비밀번호 | 앱 비밀번호 (2FA 필수) |

### 10.2 시스템 SMTP 설정
```bash
# msmtp 설치
sudo apt install msmtp msmtp-mta -y

# 설정 파일 생성
sudo tee /etc/msmtprc << 'EOF'
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /var/log/msmtp.log

account        gmail
host           smtp.gmail.com
port           587
from           your-email@gmail.com
user           your-email@gmail.com
password       your-app-password

account default : gmail
EOF

sudo chmod 600 /etc/msmtprc
```

---

## 11. Cloudflare 설정

### 11.1 단계별 설정
1. **초기 설정**: DNS only (회색 구름)
   - Type: A
   - Name: a0, a1, a2... (a0.n1a.kr ~ a5.n1a.kr 형식)
   - IPv4 Address: 서버 공인 IP
   - Proxy status: DNS only

2. **테스트 완료 후**: Proxied (오렌지 구름)
   - SSL/TLS → Full (strict) 모드
   - Always Use HTTPS 활성화

### 11.2 NPM에서 프록시 호스트 설정
1. NPM 접속 후 Proxy Hosts 추가
2. 도메인명 → 내부 IP:포트 연결
3. Let's Encrypt SSL 인증서 발급

---

## 12. 향후 확장 계획

### 12.1 알림 시스템 확장
- `start-all.sh` 및 `backup-egng.sh` 실행 후 Slack/이메일 알림
- NPM 인증서 만료일 자동 확인 및 경고 시스템

### 12.2 추가 스택 연동
- PostgreSQL, Elasticsearch 등을 위한 네트워크 설계 확장
- 백업 파일을 외부 NAS 또는 클라우드(S3) 자동 업로드

---

## ✅ 핵심 포인트

- **IP 규칙**: 시스템 도구(2-19), WordPress(100번대)
- **기동 순서**: DB → Redis → WordPress → phpMyAdmin → NPM
- **백업 전략**: VM 전체(일 1회) + DB(일 1-2회) + 파일(일 3회)
- **복구 테스트**: 월 1회 이상 백업 유효성 검증 필수
- **모니터링**: 정기 상태 점검, 디스크 사용량, 인증서 만료일
- **확장성**: 표준화된 구조로 서비스 추가 용이

정기 점검과 백업 테스트로 안정적인 운영 환경을 유지하세요!