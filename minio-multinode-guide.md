# 멀티 노드 고성능 베어메탈 MinIO 구성 및 설치 가이드

## 개요
이 가이드는 베어메탈 환경에서 고성능 멀티 노드 MinIO 클러스터를 구성하는 방법을 설명합니다.

## 시스템 요구사항

### 하드웨어 요구사항
- **최소 4개 노드** (고가용성을 위해 짝수 개 권장)
- **CPU**: 최소 4코어, 권장 8코어 이상
- **메모리**: 최소 8GB, 권장 32GB 이상
- **스토리지**: 
  - 각 노드당 최소 4개의 드라이브
  - SSD 권장 (NVMe SSD 최적)
  - 각 드라이브 최소 1TB
- **네트워크**: 10Gbps 이상 권장

### 소프트웨어 요구사항
- Linux OS (Ubuntu 20.04+, CentOS 8+, RHEL 8+ 권장)
- 시간 동기화 (NTP/Chrony)
- 방화벽 설정

## 1. 사전 준비

### 1.1 호스트 설정
각 노드에서 다음 설정을 수행합니다:

```bash
# 호스트명 설정 (각 노드별로 다르게)
sudo hostnamectl set-hostname minio-node1
sudo hostnamectl set-hostname minio-node2
sudo hostnamectl set-hostname minio-node3
sudo hostnamectl set-hostname minio-node4

# /etc/hosts 파일 업데이트 (모든 노드에서 동일하게)
sudo tee -a /etc/hosts << EOF
192.168.1.10 minio-node1
192.168.1.11 minio-node2
192.168.1.12 minio-node3
192.168.1.13 minio-node4
EOF
```

### 1.2 시간 동기화 설정
```bash
# NTP 설치 및 설정
sudo apt update
sudo apt install -y ntp

# NTP 서비스 시작
sudo systemctl enable ntp
sudo systemctl start ntp

# 시간 동기화 확인
timedatectl status
```

### 1.3 방화벽 설정
```bash
# UFW 사용 시
sudo ufw allow 9000/tcp  # MinIO API
sudo ufw allow 9001/tcp  # MinIO Console

# iptables 사용 시
sudo iptables -A INPUT -p tcp --dport 9000 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 9001 -j ACCEPT
sudo iptables-save > /etc/iptables/rules.v4
```

## 2. 스토리지 준비

### 2.1 디스크 파티셔닝 및 포맷
각 노드에서 데이터 드라이브를 준비합니다:

```bash
# 사용 가능한 디스크 확인
lsblk

# 각 디스크에 대해 파티션 생성 (예: /dev/sdb, /dev/sdc, /dev/sdd, /dev/sde)
for disk in sdb sdc sdd sde; do
    sudo parted /dev/$disk mklabel gpt
    sudo parted /dev/$disk mkpart primary 0% 100%
    sudo mkfs.xfs /dev/${disk}1
done
```

### 2.2 마운트 포인트 생성 및 설정
```bash
# 마운트 포인트 생성
sudo mkdir -p /mnt/disk{1,2,3,4}

# fstab 설정
sudo tee -a /etc/fstab << EOF
/dev/sdb1 /mnt/disk1 xfs defaults,noatime 0 2
/dev/sdc1 /mnt/disk2 xfs defaults,noatime 0 2
/dev/sdd1 /mnt/disk3 xfs defaults,noatime 0 2
/dev/sde1 /mnt/disk4 xfs defaults,noatime 0 2
EOF

# 마운트 실행
sudo mount -a

# 마운트 확인
df -h
```

## 3. MinIO 설치

### 3.1 MinIO 바이너리 다운로드
모든 노드에서 실행:

```bash
# MinIO 서버 다운로드
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

# MinIO 클라이언트 다운로드
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# 버전 확인
minio --version
mc --version
```

### 3.2 MinIO 사용자 및 디렉토리 생성
```bash
# MinIO 사용자 생성
sudo useradd -r -s /sbin/nologin -d /home/minio minio

# 데이터 디렉토리 소유권 설정
sudo chown -R minio:minio /mnt/disk{1,2,3,4}

# 로그 디렉토리 생성
sudo mkdir -p /var/log/minio
sudo chown minio:minio /var/log/minio
```

## 4. MinIO 클러스터 구성

### 4.1 환경 변수 설정
각 노드에서 환경 설정 파일을 생성합니다:

```bash
sudo tee /etc/default/minio << EOF
# MinIO 루트 사용자 자격증명
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin123

# MinIO 서버 옵션
MINIO_OPTS="--console-address :9001"

# MinIO 볼륨 설정 (모든 노드의 모든 드라이브)
MINIO_VOLUMES="http://minio-node{1...4}/mnt/disk{1...4}"

# 성능 최적화 설정
MINIO_API_REQUESTS_MAX=10000
MINIO_API_REQUESTS_DEADLINE=10s
EOF
```

### 4.2 systemd 서비스 파일 생성
```bash
sudo tee /etc/systemd/system/minio.service << EOF
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local/

User=minio
Group=minio

EnvironmentFile=/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"\${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"

ExecStart=/usr/local/bin/minio server \$MINIO_OPTS \$MINIO_VOLUMES

# Let systemd restart this service always
Restart=always

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536

# Specifies the maximum number of threads this process can create
TasksMax=infinity

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target
EOF
```

## 5. 클러스터 시작 및 확인

### 5.1 서비스 시작
모든 노드에서 동시에 실행:

```bash
# systemd 데몬 리로드
sudo systemctl daemon-reload

# MinIO 서비스 활성화 및 시작
sudo systemctl enable minio
sudo systemctl start minio

# 서비스 상태 확인
sudo systemctl status minio

# 로그 확인
sudo journalctl -u minio -f
```

### 5.2 클러스터 상태 확인
```bash
# MinIO 클라이언트 설정
mc alias set myminio http://minio-node1:9000 minioadmin minioadmin123

# 클러스터 정보 확인
mc admin info myminio

# 서버 상태 확인
mc admin service status myminio
```

## 6. 성능 최적화

### 6.1 커널 매개변수 튜닝
```bash
sudo tee -a /etc/sysctl.conf << EOF
# 네트워크 성능 최적화
net.core.rmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_default = 262144
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 65536 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# 파일 시스템 최적화
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
vm.swappiness = 1

# 파일 디스크립터 제한
fs.file-max = 2097152
EOF

# 설정 적용
sudo sysctl -p
```

### 6.2 사용자 제한 설정
```bash
sudo tee -a /etc/security/limits.conf << EOF
minio soft nofile 65536
minio hard nofile 65536
minio soft nproc 65536
minio hard nproc 65536
EOF
```

### 6.3 XFS 파일시스템 최적화
```bash
# 마운트 옵션 최적화 (fstab 수정)
sudo sed -i 's/defaults,noatime/defaults,noatime,largeio,inode64,allocsize=16m/g' /etc/fstab

# 재마운트
sudo umount /mnt/disk{1,2,3,4}
sudo mount -a
```

## 7. 보안 설정

### 7.1 TLS/SSL 설정
```bash
# 인증서 디렉토리 생성
sudo mkdir -p /home/minio/.minio/certs

# 자체 서명 인증서 생성 (프로덕션에서는 CA 인증서 사용 권장)
sudo openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
    -subj "/C=KR/ST=Seoul/L=Seoul/O=MyOrg/CN=minio-cluster" \
    -keyout /home/minio/.minio/certs/private.key \
    -out /home/minio/.minio/certs/public.crt

# 소유권 설정
sudo chown -R minio:minio /home/minio/.minio/certs
sudo chmod 600 /home/minio/.minio/certs/private.key
sudo chmod 644 /home/minio/.minio/certs/public.crt
```

### 7.2 방화벽 강화
```bash
# 특정 IP 대역만 허용 (예시)
sudo ufw allow from 192.168.1.0/24 to any port 9000
sudo ufw allow from 192.168.1.0/24 to any port 9001
```

## 8. 모니터링 및 로그

### 8.1 로그 로테이션 설정
```bash
sudo tee /etc/logrotate.d/minio << EOF
/var/log/minio/*.log {
    daily
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 644 minio minio
    postrotate
        systemctl reload minio
    endscript
}
EOF
```

### 8.2 기본 모니터링 스크립트
```bash
sudo tee /usr/local/bin/minio-health-check.sh << 'EOF'
#!/bin/bash

# MinIO 헬스 체크 스크립트
MINIO_ENDPOINT="http://localhost:9000"
LOG_FILE="/var/log/minio/health-check.log"

# 헬스 체크 수행
if curl -f -s $MINIO_ENDPOINT/minio/health/live > /dev/null; then
    echo "$(date): MinIO is healthy" >> $LOG_FILE
    exit 0
else
    echo "$(date): MinIO health check failed" >> $LOG_FILE
    exit 1
fi
EOF

sudo chmod +x /usr/local/bin/minio-health-check.sh

# cron 작업 추가
echo "*/5 * * * * /usr/local/bin/minio-health-check.sh" | sudo crontab -u minio -
```

## 9. 백업 및 복구

### 9.1 데이터 백업 스크립트
```bash
sudo tee /usr/local/bin/minio-backup.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="/backup/minio"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$BACKUP_DIR/minio_backup_$DATE"

# 백업 디렉토리 생성
mkdir -p $BACKUP_PATH

# MinIO 데이터 백업 (mc mirror 사용)
mc mirror myminio/ $BACKUP_PATH/

# 압축
tar -czf $BACKUP_PATH.tar.gz -C $BACKUP_DIR minio_backup_$DATE

# 정리
rm -rf $BACKUP_PATH

echo "Backup completed: $BACKUP_PATH.tar.gz"
EOF

sudo chmod +x /usr/local/bin/minio-backup.sh
```

## 10. 트러블슈팅

### 10.1 일반적인 문제 해결

**클러스터 노드가 연결되지 않는 경우:**
```bash
# 네트워크 연결 확인
telnet minio-node2 9000

# 방화벽 상태 확인
sudo ufw status

# 시간 동기화 확인
timedatectl status
```

**성능 문제:**
```bash
# 디스크 I/O 확인
iostat -x 1

# 네트워크 사용량 확인
iftop

# MinIO 메트릭 확인
mc admin prometheus metrics myminio
```

**로그 확인:**
```bash
# MinIO 서비스 로그
sudo journalctl -u minio -f

# 시스템 로그
sudo tail -f /var/log/syslog
```

## 11. 성능 벤치마크

### 11.1 성능 테스트
```bash
# MinIO 내장 벤치마크 도구 사용
mc admin speedtest myminio

# 사용자 정의 성능 테스트
mc cp /dev/zero myminio/testbucket/testfile --size 1GB
time mc cp myminio/testbucket/testfile /tmp/testfile
```

## 결론

이 가이드를 따라 구성하면 고성능, 고가용성의 MinIO 클러스터를 베어메탈 환경에서 운영할 수 있습니다. 프로덕션 환경에서는 다음 사항을 추가로 고려하세요:

- 전용 CA 인증서 사용
- 네트워크 분리 (관리/데이터 네트워크)
- 전문 모니터링 솔루션 (Prometheus + Grafana)
- 자동화된 백업 및 재해 복구 계획
- 로드 밸런서 구성

정기적인 백업과 모니터링을 통해 안정적인 서비스를 유지하시기 바랍니다.
