# MinIO 멀티 사이트 복제 가이드

## 문서 개요

### 목적
본 문서는 MinIO 클러스터 간 멀티 사이트 복제를 구성하여 지리적 분산, 재해 복구, 데이터 보호를 구현하는 방법을 설명합니다.

### 대상 독자
- 시스템 관리자
- DevOps 엔지니어
- 스토리지 아키텍트
- 재해 복구 담당자

### 문서 버전
- 버전: 1.0
- 작성일: 2025-08-11
- 최종 수정일: 2025-08-11

## 1. 멀티 사이트 복제 개요

### 1.1 비즈니스 요구사항
- **재해 복구**: 자연재해나 시스템 장애 시 서비스 연속성 보장
- **지리적 분산**: 글로벌 서비스를 위한 지역별 데이터 배치
- **규정 준수**: 데이터 주권 및 규제 요구사항 충족
- **성능 최적화**: 사용자와 가까운 위치에서 데이터 서비스
- **백업 및 아카이브**: 장기 보존을 위한 다중 사이트 백업

### 1.2 기술적 목표
- **RPO (Recovery Point Objective)**: < 5분
- **RTO (Recovery Time Objective)**: < 30분
- **데이터 일관성**: 최종 일관성 보장
- **네트워크 효율성**: 대역폭 최적화 및 압축
- **자동 장애 조치**: 무인 운영 환경 지원

### 1.3 복제 유형
MinIO는 다음과 같은 복제 방식을 지원합니다:

#### 단방향 복제 (One-way Replication)
```
Site A (Primary) ──────────▶ Site B (Replica)
```

#### 양방향 복제 (Bi-directional Replication)
```
Site A ◀──────────────────▶ Site B
```

#### 다중 사이트 복제 (Multi-site Replication)
```
        Site A
       ╱      ╲
      ╱        ╲
   Site B ◀──▶ Site C
      ╲        ╱
       ╲      ╱
        Site D
```

## 2. 아키텍처 설계

### 2.1 레퍼런스 아키텍처

#### 3-Site 지리적 분산 구성
```
┌─────────────────────────────────────────────────────────────────┐
│                    Seoul Data Center                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │   MinIO Node 1  │  │   MinIO Node 2  │  │   MinIO Node 3  │  │
│  │   (Primary)     │  │   (Primary)     │  │   (Primary)     │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                    Cluster: seoul-cluster                       │
└─────────────────────────────────────────────────────────────────┘
                                │
                                │ Replication
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Busan Data Center                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │   MinIO Node 1  │  │   MinIO Node 2  │  │   MinIO Node 3  │  │
│  │   (DR Site)     │  │   (DR Site)     │  │   (DR Site)     │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                    Cluster: busan-cluster                       │
└─────────────────────────────────────────────────────────────────┘
                                │
                                │ Replication
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Tokyo Data Center                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │   MinIO Node 1  │  │   MinIO Node 2  │  │   MinIO Node 3  │  │
│  │   (Archive)     │  │   (Archive)     │  │   (Archive)     │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                    Cluster: tokyo-cluster                       │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 네트워크 요구사항

#### 사이트 간 연결
```
최소 대역폭: 1Gbps (권장: 10Gbps)
지연시간: < 100ms (권장: < 50ms)
패킷 손실률: < 0.1%
가용성: 99.9% 이상
```

#### 네트워크 토폴로지
```
Seoul DC ──── 전용선/VPN ──── Busan DC
    │                           │
    │                           │
    └──── 전용선/VPN ──── Tokyo DC
```

## 3. 사전 준비사항

### 3.1 시스템 요구사항

#### 각 사이트별 MinIO 클러스터
- 최소 3노드 클러스터 (고가용성)
- 동일한 MinIO 버전
- 시간 동기화 (NTP)
- DNS 해상도

#### 네트워크 구성
```bash
# 각 사이트의 클러스터 엔드포인트
Seoul:  https://minio-seoul.example.com:9000
Busan:  https://minio-busan.example.com:9000
Tokyo:  https://minio-tokyo.example.com:9000
```

### 3.2 보안 설정

#### TLS 인증서 준비
각 사이트에서 유효한 TLS 인증서가 필요합니다:

```bash
# 각 사이트별 인증서 확인
openssl x509 -in /etc/minio/certs/public.crt -text -noout

# 인증서 체인 검증
openssl verify -CAfile ca-bundle.crt public.crt
```

#### 방화벽 설정
```bash
# 사이트 간 통신을 위한 포트 개방
# Seoul -> Busan
iptables -A OUTPUT -d busan-minio-subnet -p tcp --dport 9000 -j ACCEPT

# Seoul -> Tokyo  
iptables -A OUTPUT -d tokyo-minio-subnet -p tcp --dport 9000 -j ACCEPT
```

## 4. 복제 구성

### 4.1 기본 복제 설정

#### 4.1.1 MinIO 클라이언트 설정
각 사이트에서 다른 사이트들을 alias로 등록합니다:

```bash
# Seoul 사이트에서 실행
mc alias set seoul https://minio-seoul.example.com:9000 admin password123
mc alias set busan https://minio-busan.example.com:9000 admin password123
mc alias set tokyo https://minio-tokyo.example.com:9000 admin password123

# 연결 테스트
mc admin info seoul
mc admin info busan
mc admin info tokyo
```

#### 4.1.2 버킷 생성
복제할 버킷을 각 사이트에 생성합니다:

```bash
# 모든 사이트에 동일한 버킷 생성
mc mb seoul/data-bucket
mc mb busan/data-bucket
mc mb tokyo/data-bucket

# 버킷 정책 설정 (필요시)
mc policy set public seoul/data-bucket
mc policy set public busan/data-bucket
mc policy set public tokyo/data-bucket
```

### 4.2 단방향 복제 구성

#### Seoul → Busan 복제 설정
```bash
# Seoul에서 Busan으로의 복제 규칙 생성
mc replicate add seoul/data-bucket \
  --remote-bucket busan/data-bucket \
  --priority 1 \
  --tags "env=production"

# 복제 상태 확인
mc replicate ls seoul/data-bucket
```

#### Seoul → Tokyo 복제 설정
```bash
# Seoul에서 Tokyo로의 복제 규칙 생성
mc replicate add seoul/data-bucket \
  --remote-bucket tokyo/data-bucket \
  --priority 2 \
  --tags "env=archive"

# 복제 상태 확인
mc replicate ls seoul/data-bucket
```

### 4.3 양방향 복제 구성

#### Seoul ↔ Busan 양방향 복제
```bash
# Seoul → Busan
mc replicate add seoul/data-bucket \
  --remote-bucket busan/data-bucket \
  --priority 1

# Busan → Seoul
mc replicate add busan/data-bucket \
  --remote-bucket seoul/data-bucket \
  --priority 1
```

### 4.4 고급 복제 설정

#### 조건부 복제 (Prefix 기반)
```bash
# 특정 prefix만 복제
mc replicate add seoul/data-bucket \
  --remote-bucket busan/data-bucket \
  --prefix "important/" \
  --priority 1

# 특정 태그가 있는 객체만 복제
mc replicate add seoul/data-bucket \
  --remote-bucket tokyo/data-bucket \
  --tags "backup=true" \
  --priority 2
```

#### 스토리지 클래스 기반 복제
```bash
# 특정 스토리지 클래스로 복제
mc replicate add seoul/data-bucket \
  --remote-bucket tokyo/data-bucket \
  --storage-class "COLD" \
  --priority 3
```

## 5. 복제 모니터링 및 관리

### 5.1 복제 상태 모니터링

#### 복제 통계 확인
```bash
# 전체 복제 상태 확인
mc replicate ls seoul/data-bucket

# 상세 복제 통계
mc replicate status seoul/data-bucket

# 복제 메트릭 확인
mc admin replicate info seoul
```

#### 복제 지연 모니터링
```bash
# 복제 지연시간 확인
mc replicate status seoul/data-bucket --verbose

# 실패한 복제 작업 확인
mc replicate status seoul/data-bucket --failed
```

### 5.2 Prometheus 메트릭

#### 복제 관련 메트릭
```yaml
# prometheus.yml에 추가할 메트릭
- minio_replication_pending_count
- minio_replication_failed_count
- minio_replication_pending_bytes
- minio_replication_failed_bytes
- minio_replication_last_hour_failed_bytes
- minio_replication_total_failed_bytes
```

#### Grafana 대시보드 쿼리 예시
```promql
# 복제 대기 중인 객체 수
sum(minio_replication_pending_count) by (server)

# 복제 실패율
rate(minio_replication_failed_count[5m]) / rate(minio_replication_pending_count[5m]) * 100

# 복제 처리량
rate(minio_replication_pending_bytes[5m])
```

### 5.3 알림 설정

#### Alertmanager 규칙
```yaml
groups:
  - name: minio-replication.rules
    rules:
      - alert: MinIOReplicationLag
        expr: minio_replication_pending_count > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "MinIO replication lag detected"
          description: "Replication pending count is {{ $value }}"

      - alert: MinIOReplicationFailure
        expr: rate(minio_replication_failed_count[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "MinIO replication failures detected"
          description: "Replication failure rate is {{ $value }}/min"
```

## 6. 장애 조치 및 복구

### 6.1 자동 장애 조치

#### 헬스 체크 스크립트
```bash
#!/bin/bash
# minio-failover-check.sh

PRIMARY_SITE="seoul"
BACKUP_SITE="busan"
HEALTH_CHECK_TIMEOUT=10

check_site_health() {
    local site=$1
    timeout $HEALTH_CHECK_TIMEOUT mc admin info $site >/dev/null 2>&1
    return $?
}

# Primary 사이트 상태 확인
if ! check_site_health $PRIMARY_SITE; then
    echo "Primary site $PRIMARY_SITE is down, initiating failover to $BACKUP_SITE"
    
    # DNS 업데이트 또는 로드 밸런서 설정 변경
    # update_dns_record "minio.example.com" "$BACKUP_SITE"
    
    # 알림 발송
    send_alert "MinIO failover initiated: $PRIMARY_SITE -> $BACKUP_SITE"
    
    exit 1
else
    echo "Primary site $PRIMARY_SITE is healthy"
    exit 0
fi
```

#### Cron 작업 설정
```bash
# 5분마다 헬스 체크 실행
*/5 * * * * /usr/local/bin/minio-failover-check.sh >> /var/log/minio-failover.log 2>&1
```

### 6.2 수동 장애 조치

#### 긴급 장애 조치 절차
```bash
# 1. Primary 사이트 상태 확인
mc admin info seoul

# 2. Backup 사이트 상태 확인
mc admin info busan

# 3. 복제 상태 확인
mc replicate status seoul/data-bucket

# 4. DNS 또는 로드 밸런서 업데이트
# (실제 환경에 따라 다름)

# 5. 애플리케이션 설정 업데이트
# 애플리케이션의 MinIO 엔드포인트를 backup 사이트로 변경
```

### 6.3 사이트 복구

#### Primary 사이트 복구 절차
```bash
# 1. Primary 사이트 복구 후 상태 확인
mc admin info seoul

# 2. 데이터 동기화 상태 확인
mc diff busan/data-bucket seoul/data-bucket

# 3. 역방향 복제 설정 (필요시)
mc replicate add busan/data-bucket \
  --remote-bucket seoul/data-bucket \
  --priority 1

# 4. 동기화 완료 후 원래 설정으로 복원
# DNS 또는 로드 밸런서를 원래 설정으로 되돌림

# 5. 역방향 복제 제거
mc replicate rm busan/data-bucket --force
```

## 7. 성능 최적화

### 7.1 네트워크 최적화

#### 대역폭 제한 설정
```bash
# 복제 대역폭 제한 (예: 100MB/s)
mc replicate add seoul/data-bucket \
  --remote-bucket busan/data-bucket \
  --bandwidth 100MB \
  --priority 1
```

#### 압축 활성화
```bash
# 복제 시 압축 사용
mc replicate add seoul/data-bucket \
  --remote-bucket tokyo/data-bucket \
  --compress \
  --priority 2
```

### 7.2 복제 성능 튜닝

#### 동시 복제 작업 수 조정
```bash
# MinIO 서버 환경 변수 설정
export MINIO_REPLICATION_WORKERS=10
export MINIO_REPLICATION_FAILED_WORKERS=4
```

#### 배치 크기 최적화
```bash
# 복제 배치 크기 설정
export MINIO_REPLICATION_MAX_WORKERS=20
export MINIO_REPLICATION_QUEUE_SIZE=100000
```

## 8. 보안 고려사항

### 8.1 전송 중 암호화

#### TLS 설정 확인
```bash
# 각 사이트의 TLS 설정 확인
openssl s_client -connect minio-seoul.example.com:9000 -servername minio-seoul.example.com

# 인증서 체인 검증
mc admin info seoul --insecure=false
```

### 8.2 접근 제어

#### 복제 전용 사용자 생성
```bash
# 복제 전용 사용자 생성
mc admin user add seoul replication-user replication-password

# 복제 권한 정책 생성
cat > replication-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetBucketVersioning"
      ],
      "Resource": [
        "arn:aws:s3:::data-bucket"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging",
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags"
      ],
      "Resource": [
        "arn:aws:s3:::data-bucket/*"
      ]
    }
  ]
}
EOF

# 정책 적용
mc admin policy add seoul replication-policy replication-policy.json
mc admin policy set seoul replication-policy user=replication-user
```

## 9. 트러블슈팅

### 9.1 일반적인 문제 해결

#### 복제가 작동하지 않는 경우
```bash
# 1. 네트워크 연결 확인
telnet minio-busan.example.com 9000

# 2. 인증 정보 확인
mc admin info busan

# 3. 복제 규칙 확인
mc replicate ls seoul/data-bucket

# 4. 복제 상태 확인
mc replicate status seoul/data-bucket --verbose

# 5. 서버 로그 확인
mc admin logs seoul
```

#### 복제 지연 문제
```bash
# 복제 대기열 상태 확인
mc replicate status seoul/data-bucket

# 네트워크 대역폭 확인
iftop -i eth0

# 복제 워커 수 증가
export MINIO_REPLICATION_WORKERS=20
systemctl restart minio
```

#### 복제 실패 문제
```bash
# 실패한 복제 작업 확인
mc replicate status seoul/data-bucket --failed

# 실패한 작업 재시도
mc replicate resync start seoul/data-bucket

# 재동기화 상태 확인
mc replicate resync status seoul/data-bucket
```

### 9.2 로그 분석

#### 복제 관련 로그 필터링
```bash
# MinIO 서버 로그에서 복제 관련 항목 필터링
mc admin logs seoul | grep -i replication

# 특정 시간대 로그 확인
mc admin logs seoul --since 2h | grep -i "replication\|error"
```

## 10. 운영 절차

### 10.1 일상 운영 작업

#### 일일 복제 상태 점검 스크립트
```bash
#!/bin/bash
# daily-replication-check.sh

SITES=("seoul" "busan" "tokyo")
BUCKETS=("data-bucket" "backup-bucket")
REPORT_FILE="/var/log/minio-replication-daily-$(date +%Y%m%d).log"

echo "=== MinIO Multi-Site Replication Daily Report ===" > $REPORT_FILE
echo "Date: $(date)" >> $REPORT_FILE
echo "" >> $REPORT_FILE

for site in "${SITES[@]}"; do
    echo "=== Site: $site ===" >> $REPORT_FILE
    
    # 사이트 상태 확인
    if mc admin info $site >/dev/null 2>&1; then
        echo "Status: ONLINE" >> $REPORT_FILE
        
        for bucket in "${BUCKETS[@]}"; do
            echo "--- Bucket: $bucket ---" >> $REPORT_FILE
            
            # 복제 상태 확인
            mc replicate status $site/$bucket >> $REPORT_FILE 2>&1
            echo "" >> $REPORT_FILE
        done
    else
        echo "Status: OFFLINE" >> $REPORT_FILE
        echo "ALERT: Site $site is not accessible!" >> $REPORT_FILE
    fi
    echo "" >> $REPORT_FILE
done

# 이메일 발송 (선택사항)
# mail -s "MinIO Replication Daily Report" admin@example.com < $REPORT_FILE
```

### 10.2 정기 유지보수

#### 주간 복제 성능 분석
```bash
#!/bin/bash
# weekly-replication-analysis.sh

# 지난 주 복제 메트릭 수집
START_DATE=$(date -d "7 days ago" +%Y-%m-%d)
END_DATE=$(date +%Y-%m-%d)

echo "=== Weekly Replication Performance Analysis ==="
echo "Period: $START_DATE to $END_DATE"
echo ""

# Prometheus 쿼리를 통한 메트릭 수집 (예시)
# curl -G 'http://prometheus:9090/api/v1/query_range' \
#   --data-urlencode 'query=rate(minio_replication_pending_bytes[1h])' \
#   --data-urlencode "start=${START_DATE}T00:00:00Z" \
#   --data-urlencode "end=${END_DATE}T00:00:00Z" \
#   --data-urlencode 'step=3600'

# 복제 실패 분석
for site in seoul busan tokyo; do
    echo "=== Site: $site ==="
    mc replicate status $site/data-bucket --failed | head -20
    echo ""
done
```

## 11. 재해 복구 계획

### 11.1 재해 시나리오별 대응

#### 시나리오 1: Primary 사이트 완전 장애
```
1. 자동 헬스 체크에 의한 장애 감지
2. DNS/로드 밸런서를 통한 트래픽 우회
3. Backup 사이트로 서비스 전환
4. 사용자 및 관련 팀 알림
5. Primary 사이트 복구 작업 시작
```

#### 시나리오 2: 네트워크 분할 (Split-brain)
```
1. 각 사이트의 독립적 운영 모드 전환
2. 네트워크 복구 후 데이터 일관성 검사
3. 충돌 해결 및 데이터 병합
4. 정상 복제 모드 복원
```

### 11.2 복구 시간 목표

#### RTO/RPO 매트릭스
```
시나리오                    RTO        RPO
단일 노드 장애             5분        1분
사이트 부분 장애           15분       5분
사이트 완전 장애           30분       10분
네트워크 분할              60분       30분
```

## 12. 비용 최적화

### 12.1 네트워크 비용 최적화

#### 복제 스케줄링
```bash
# 야간 시간대에만 대용량 복제 수행
# crontab 설정
0 2 * * * mc replicate add seoul/archive-bucket --remote-bucket tokyo/archive-bucket --priority 1
0 6 * * * mc replicate rm seoul/archive-bucket --force
```

#### 압축 및 중복 제거
```bash
# 복제 시 압축 활성화로 네트워크 비용 절감
mc replicate add seoul/data-bucket \
  --remote-bucket busan/data-bucket \
  --compress \
  --priority 1
```

### 12.2 스토리지 비용 최적화

#### 계층화된 복제 전략
```bash
# Hot 데이터: 실시간 복제
mc replicate add seoul/hot-data \
  --remote-bucket busan/hot-data \
  --priority 1

# Warm 데이터: 일일 복제
mc replicate add seoul/warm-data \
  --remote-bucket tokyo/warm-data \
  --priority 2 \
  --storage-class "STANDARD_IA"

# Cold 데이터: 주간 복제
mc replicate add seoul/cold-data \
  --remote-bucket tokyo/cold-data \
  --priority 3 \
  --storage-class "GLACIER"
```

## 13. 결론 및 모범 사례

### 13.1 핵심 모범 사례

1. **점진적 구현**: 작은 규모로 시작하여 단계적 확장
2. **철저한 테스트**: 프로덕션 배포 전 충분한 테스트
3. **모니터링 우선**: 복제 설정과 동시에 모니터링 구축
4. **문서화**: 모든 설정과 절차 문서화
5. **정기 훈련**: 재해 복구 시나리오 정기 훈련

### 13.2 주의사항

- **네트워크 대역폭**: 충분한 대역폭 확보 필수
- **시간 동기화**: 모든 사이트의 정확한 시간 동기화
- **보안**: 사이트 간 통신 암호화 필수
- **용량 계획**: 복제로 인한 추가 스토리지 용량 고려
- **비용 관리**: 네트워크 및 스토리지 비용 지속적 모니터링

### 13.3 향후 고려사항

- **글로벌 확장**: 추가 지역으로의 복제 확장
- **자동화**: Infrastructure as Code를 통한 자동화
- **AI/ML 통합**: 지능형 복제 최적화
- **엣지 컴퓨팅**: 엣지 위치로의 복제 확장

---

**문서 승인**
- 시스템 아키텍트: [서명]
- 네트워크 관리자: [서명]
- 보안 담당자: [서명]
- 날짜: 2025-08-11
