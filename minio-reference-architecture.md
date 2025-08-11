# MinIO 멀티 노드 베어메탈 클러스터 레퍼런스 아키텍처

## 문서 개요

### 목적
본 문서는 엔터프라이즈급 고성능 오브젝트 스토리지 서비스를 위한 MinIO 멀티 노드 베어메탈 클러스터의 레퍼런스 아키텍처를 제시합니다.

### 대상 독자
- 시스템 아키텍트
- 인프라 엔지니어
- DevOps 엔지니어
- 스토리지 관리자

### 문서 버전
- 버전: 1.0
- 작성일: 2025-08-11
- 최종 수정일: 2025-08-11

## 1. 아키텍처 개요

### 1.1 비즈니스 요구사항
- **고가용성**: 99.99% 이상의 서비스 가용성
- **확장성**: 페타바이트급 스토리지 확장 지원
- **성능**: 높은 처리량과 낮은 지연시간
- **데이터 보호**: 다중 복제 및 erasure coding
- **비용 효율성**: 상용 하드웨어 기반 구성

### 1.2 기술적 목표
- **처리량**: 10GB/s 이상의 집계 처리량
- **IOPS**: 100,000 IOPS 이상
- **지연시간**: 1ms 미만의 평균 응답시간
- **내구성**: 99.999999999% (11 9's) 데이터 내구성
- **복구 시간**: RTO < 15분, RPO < 1분

## 2. 전체 시스템 아키텍처

### 2.1 논리적 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Applications                       │
├─────────────────────────────────────────────────────────────────┤
│                      Load Balancer Layer                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │   HAProxy/F5    │  │   HAProxy/F5    │  │   HAProxy/F5    │  │
│  │   (Primary)     │  │   (Secondary)   │  │   (Tertiary)    │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                      MinIO Cluster Layer                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │   MinIO Node 1  │  │   MinIO Node 2  │  │   MinIO Node 3  │  │
│  │   (minio-01)    │  │   (minio-02)    │  │   (minio-03)    │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │   MinIO Node 4  │  │   MinIO Node 5  │  │   MinIO Node 6  │  │
│  │   (minio-04)    │  │   (minio-05)    │  │   (minio-06)    │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                      Storage Layer                              │
│           Direct Attached Storage (DAS) per Node                │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 물리적 아키텍처

```
Data Center / Server Room Layout

┌─────────────────────────────────────────────────────────────────┐
│                          Rack 1                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Load Bal #1 │  │ Load Bal #2 │  │ Load Bal #3 │              │
│  │ (HAProxy)   │  │ (HAProxy)   │  │ (HAProxy)   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Switch    │  │   Switch    │  │   Switch    │              │
│  │ (10/25GbE)  │  │ (10/25GbE)  │  │ (10/25GbE)  │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                          Rack 2                                 │
│  ┌─────────────┐  ┌─────────────┐                               │
│  │ MinIO-01    │  │ MinIO-02    │                               │
│  │ 2U Server   │  │ 2U Server   │                               │
│  │ 12x NVMe    │  │ 12x NVMe    │                               │
│  └─────────────┘  └─────────────┘                               │
│  ┌─────────────┐  ┌─────────────┐                               │
│  │ MinIO-03    │  │ MinIO-04    │                               │
│  │ 2U Server   │  │ 2U Server   │                               │
│  │ 12x NVMe    │  │ 12x NVMe    │                               │
│  └─────────────┘  └─────────────┘                               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                          Rack 3                                 │
│  ┌─────────────┐  ┌─────────────┐                               │
│  │ MinIO-05    │  │ MinIO-06    │                               │
│  │ 2U Server   │  │ 2U Server   │                               │
│  │ 12x NVMe    │  │ 12x NVMe    │                               │
│  └─────────────┘  └─────────────┘                               │
│  ┌─────────────┐  ┌─────────────┐                               │
│  │ Monitoring  │  │ Management  │                               │
│  │ Server      │  │ Server      │                               │
│  └─────────────┘  └─────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 하드웨어 사양

### 3.1 MinIO 노드 사양

#### 권장 하드웨어 구성
```
서버 모델: Dell PowerEdge R750 또는 동급
폼팩터: 2U 랙마운트

CPU:
- Intel Xeon Gold 6338 (32코어, 2.0GHz) x2
- 또는 AMD EPYC 7543 (32코어, 2.8GHz) x2

메모리:
- 256GB DDR4-3200 ECC RDIMM
- 16x 16GB 모듈

스토리지:
- NVMe SSD: 12x 7.68TB Enterprise NVMe SSD
- RAID 컨트롤러: HBA330 (JBOD 모드)
- 총 용량: 92TB Raw per Node

네트워크:
- 25GbE Dual Port SFP28 NIC x2 (본딩)
- 1GbE Quad Port RJ45 NIC (관리용)

전원:
- 이중화 전원공급장치 (1100W x2)
- UPS 연결
```

### 3.2 네트워크 인프라

#### 네트워크 토폴로지
```
┌─────────────────────────────────────────────────────────────────┐
│                    Core Network Layer                           │
│  ┌─────────────────┐              ┌─────────────────┐           │
│  │  Core Switch 1  │──────────────│  Core Switch 2  │           │
│  │  (100GbE)       │              │  (100GbE)       │           │
│  └─────────────────┘              └─────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
           │                                    │
           │                                    │
┌─────────────────────────────────────────────────────────────────┐
│                 Distribution Layer                              │
│  ┌─────────────────┐              ┌─────────────────┐           │
│  │  Dist Switch 1  │──────────────│  Dist Switch 2  │           │
│  │  (25/100GbE)    │              │  (25/100GbE)    │           │
│  └─────────────────┘              └─────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
     │         │         │                │         │         │
     │         │         │                │         │         │
┌─────────────────────────────────────────────────────────────────┐
│                   Access Layer                                  │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                │
│ │ToR Switch 1 │ │ToR Switch 2 │ │ToR Switch 3 │                │
│ │ (25GbE)     │ │ (25GbE)     │ │ (25GbE)     │                │
│ └─────────────┘ └─────────────┘ └─────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

#### VLAN 설계
```
VLAN 100: Management Network (192.168.100.0/24)
- 서버 관리 인터페이스
- IPMI/iDRAC 접근
- 모니터링 트래픽

VLAN 200: Storage Data Network (10.0.200.0/24)
- MinIO 클러스터 간 통신
- 클라이언트 데이터 트래픽
- 고대역폭 전용

VLAN 300: Backup Network (10.0.300.0/24)
- 백업 트래픽 전용
- 복제 트래픽
```

## 4. 소프트웨어 아키텍처

### 4.1 MinIO 클러스터 구성

#### Erasure Coding 설정
```
클러스터 구성: 6 노드 x 12 드라이브 = 72 드라이브
Erasure Set 크기: 12 드라이브 (EC:6)
- 데이터 샤드: 6개
- 패리티 샤드: 6개
- 허용 장애: 최대 6개 드라이브 동시 장애

총 Erasure Set 수: 6개
실제 사용 가능 용량: 약 50% (패리티 오버헤드 포함)
```

#### 클러스터 엔드포인트 구성
```bash
# MinIO 서버 시작 명령어 예시
MINIO_VOLUMES="http://minio-{01...06}.example.com/mnt/disk{1...12}"

# 각 노드의 엔드포인트
minio-01.example.com:9000
minio-02.example.com:9000
minio-03.example.com:9000
minio-04.example.com:9000
minio-05.example.com:9000
minio-06.example.com:9000
```

### 4.2 고가용성 구성

#### 로드 밸런서 설정
```nginx
# HAProxy 구성 예시
global
    maxconn 4096
    log stdout local0

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog

frontend minio_frontend
    bind *:9000
    bind *:9001
    default_backend minio_backend

backend minio_backend
    balance roundrobin
    option httpchk GET /minio/health/live
    server minio-01 minio-01.example.com:9000 check
    server minio-02 minio-02.example.com:9000 check
    server minio-03 minio-03.example.com:9000 check
    server minio-04 minio-04.example.com:9000 check
    server minio-05 minio-05.example.com:9000 check
    server minio-06 minio-06.example.com:9000 check
```

## 5. 보안 아키텍처

### 5.1 네트워크 보안

#### 방화벽 규칙
```bash
# MinIO 노드 간 통신
iptables -A INPUT -s 10.0.200.0/24 -p tcp --dport 9000 -j ACCEPT
iptables -A INPUT -s 10.0.200.0/24 -p tcp --dport 9001 -j ACCEPT

# 로드 밸런서에서의 접근
iptables -A INPUT -s 10.0.200.10-12 -p tcp --dport 9000 -j ACCEPT

# 관리 네트워크
iptables -A INPUT -s 192.168.100.0/24 -p tcp --dport 22 -j ACCEPT
```

### 5.2 데이터 보안

#### 암호화 구성
```yaml
# 전송 중 암호화 (TLS)
TLS_CERT_FILE: /etc/minio/certs/public.crt
TLS_KEY_FILE: /etc/minio/certs/private.key
TLS_CA_FILE: /etc/minio/certs/ca.crt

# 저장 시 암호화 (KMS)
MINIO_KMS_KES_ENDPOINT: https://kes.example.com:7373
MINIO_KMS_KES_KEY_FILE: /etc/minio/certs/kes-client.key
MINIO_KMS_KES_CERT_FILE: /etc/minio/certs/kes-client.crt
```

## 6. 모니터링 및 관찰성

### 6.1 모니터링 스택

#### Prometheus + Grafana 구성
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'minio'
    static_configs:
      - targets:
        - 'minio-01.example.com:9000'
        - 'minio-02.example.com:9000'
        - 'minio-03.example.com:9000'
        - 'minio-04.example.com:9000'
        - 'minio-05.example.com:9000'
        - 'minio-06.example.com:9000'
    metrics_path: /minio/v2/metrics/cluster
    scheme: https
```

### 6.2 핵심 메트릭

#### 성능 메트릭
```
- minio_cluster_capacity_usable_total
- minio_cluster_capacity_usable_free
- minio_http_requests_duration_seconds
- minio_s3_requests_total
- minio_s3_errors_total
- minio_cluster_nodes_online_total
- minio_cluster_drives_online_total
```

#### 알림 규칙
```yaml
groups:
  - name: minio.rules
    rules:
      - alert: MinIONodeDown
        expr: minio_cluster_nodes_online_total < 6
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "MinIO node is down"
          
      - alert: MinIODriveOffline
        expr: minio_cluster_drives_offline_total > 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "MinIO drive is offline"
```

## 7. 백업 및 재해 복구

### 7.1 백업 전략

#### 3-2-1 백업 규칙
```
3개의 복사본: 원본 + 2개 백업
2개의 다른 미디어: 로컬 + 클라우드
1개의 오프사이트: 원격 지역
```

#### 백업 아키텍처
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Primary       │    │   Local Backup  │    │  Remote Backup  │
│   MinIO         │───▶│   MinIO         │───▶│   AWS S3/       │
│   Cluster       │    │   Cluster       │    │   Azure Blob    │
│   (Production)  │    │   (DR Site)     │    │   (Cloud)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 7.2 재해 복구 계획

#### RTO/RPO 목표
```
RTO (Recovery Time Objective): 15분
RPO (Recovery Point Objective): 1분
MTTR (Mean Time To Repair): 30분
MTBF (Mean Time Between Failures): 8760시간
```

## 8. 성능 최적화

### 8.1 시스템 튜닝

#### 커널 매개변수
```bash
# /etc/sysctl.conf
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.core.netdev_max_backlog = 5000
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
```

#### 파일시스템 최적화
```bash
# XFS 마운트 옵션
/dev/nvme0n1 /mnt/disk1 xfs defaults,noatime,largeio,inode64,allocsize=16m 0 2
```

### 8.2 성능 벤치마크

#### 예상 성능 지표
```
순차 읽기: 15-20 GB/s (집계)
순차 쓰기: 12-15 GB/s (집계)
랜덤 읽기: 500,000 IOPS
랜덤 쓰기: 300,000 IOPS
평균 지연시간: < 1ms
99th percentile 지연시간: < 5ms
```

## 9. 운영 절차

### 9.1 일상 운영 작업

#### 헬스 체크 스크립트
```bash
#!/bin/bash
# daily-health-check.sh

echo "=== MinIO Cluster Health Check ==="
echo "Date: $(date)"

# 클러스터 상태 확인
mc admin info minio-cluster

# 드라이브 상태 확인
mc admin drive list minio-cluster

# 성능 메트릭 확인
mc admin speedtest minio-cluster

# 용량 사용률 확인
mc admin usage minio-cluster
```

### 9.2 확장 절차

#### 노드 추가 절차
```bash
# 1. 새 노드 준비
# 2. 기존 클러스터 중지
systemctl stop minio

# 3. 새 구성으로 재시작
MINIO_VOLUMES="http://minio-{01...08}.example.com/mnt/disk{1...12}"
systemctl start minio

# 4. 리밸런싱 시작
mc admin rebalance start minio-cluster
```

## 10. 비용 분석

### 10.1 초기 투자 비용 (USD)

```
하드웨어:
- 서버 (6대): $180,000
- 네트워크 장비: $50,000
- 랙 및 전원: $20,000
소계: $250,000

소프트웨어:
- MinIO Enterprise: $30,000/년
- 모니터링 도구: $10,000/년
소계: $40,000/년

운영비용:
- 전력 (50kW): $36,000/년
- 냉각: $15,000/년
- 인건비: $120,000/년
소계: $171,000/년

총 3년 TCO: $883,000
```

### 10.2 ROI 분석

```
대안 클라우드 비용 (3년):
- AWS S3 (500TB): $1,200,000
- 데이터 전송: $300,000
총계: $1,500,000

절약 비용: $617,000 (41% 절약)
투자 회수 기간: 18개월
```

## 11. 결론 및 권장사항

### 11.1 핵심 이점
- **비용 효율성**: 클라우드 대비 40% 이상 비용 절약
- **성능**: 클라우드 스토리지 대비 10배 이상 높은 성능
- **제어권**: 완전한 데이터 제어 및 보안
- **확장성**: 페타바이트급까지 선형 확장 가능

### 11.2 구현 권장사항
1. **단계적 구현**: 4노드로 시작하여 점진적 확장
2. **파일럿 테스트**: 프로덕션 배포 전 충분한 테스트
3. **모니터링 우선**: 배포와 동시에 모니터링 구축
4. **백업 전략**: 첫날부터 백업 체계 구축
5. **문서화**: 모든 절차와 구성 문서화

### 11.3 향후 고려사항
- **멀티 사이트 복제**: 지리적 분산을 위한 사이트 간 복제
- **AI/ML 워크로드**: 고성능 데이터 파이프라인 지원
- **컨테이너 통합**: Kubernetes 환경과의 통합
- **자동화**: Infrastructure as Code 적용

---

**문서 승인**
- 아키텍트: [서명]
- 보안 담당자: [서명]
- 운영 담당자: [서명]
- 날짜: 2025-08-11
