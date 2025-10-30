
````markdown
# ELK Stack 설치 가이드 (WSL/Linux)

## 목차
1. [사전 준비](#1-사전-준비)
2. [Elasticsearch 설치](#2-elasticsearch-설치)
3. [Kibana 설치](#3-kibana-설치)
4. [Filebeat 설치](#4-filebeat-설치)
5. [테스트 로그 생성](#5-테스트-로그-생성)
6. [확인 및 검증](#6-확인-및-검증)
7. [Kibana에서 검색](#7-kibana에서-검색)
8. [자동 시작 설정](#8-자동-시작-설정)
9. [문제 해결](#9-문제-해결)

---

## 1. 사전 준비

### 1.1 Java 설치 (Elasticsearch 필수)
```bash
# Java 17 설치
sudo apt update
sudo apt install openjdk-17-jdk -y

# 설치 확인
java -version
# 출력: openjdk version "17.0.x"
```

### 1.2 작업 디렉토리 생성
```bash
cd ~
mkdir elastic-stack
cd elastic-stack
```

---

## 2. Elasticsearch 설치

### 2.1 다운로드 및 압축 해제
```bash
cd ~/elastic-stack

# Elasticsearch 다운로드
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.11.0-linux-x86_64.tar.gz

# 압축 해제
tar -xzf elasticsearch-8.11.0-linux-x86_64.tar.gz
cd elasticsearch-8.11.0
```

### 2.2 설정 파일 편집
```bash
vi config/elasticsearch.yml
```

**config/elasticsearch.yml 내용:**
```yaml
# 클러스터 설정
cluster.name: airline-logs-cluster
node.name: node-1

# 데이터 경로 (30일치 로그 저장)
path.data: /home/YOUR_USERNAME/elastic-stack/elasticsearch-8.11.0/data
path.logs: /home/YOUR_USERNAME/elastic-stack/elasticsearch-8.11.0/logs

# 네트워크 설정
network.host: 0.0.0.0
http.port: 9200

# 보안 비활성화 (테스트/내부망용)
xpack.security.enabled: false
xpack.security.enrollment.enabled: false
xpack.security.http.ssl.enabled: false
xpack.security.transport.ssl.enabled: false

# 단일 노드 모드
discovery.type: single-node
```

> **주의:** `YOUR_USERNAME`을 실제 사용자명으로 변경하세요.

### 2.3 JVM 메모리 설정
```bash
vi config/jvm.options
```

**32-33번 줄 수정 (WSL/테스트용):**
```
-Xms2g
-Xmx2g
```

> **주의:** 
> - 앞에 공백 없이 작성
> - 운영 서버: 4-8GB 권장
> - WSL 테스트: 1-2GB로 충분

### 2.4 Elasticsearch 시작
```bash
# 포그라운드 실행 (테스트용)
./bin/elasticsearch

# 정상 실행 확인 후 Ctrl+C로 종료

# 백그라운드 실행
./bin/elasticsearch -d

# 확인
curl http://localhost:9200
```

**정상 응답 예시:**
```json
{
  "name" : "node-1",
  "cluster_name" : "airline-logs-cluster",
  "cluster_uuid" : "...",
  "version" : {
    "number" : "8.11.0",
    ...
  },
  "tagline" : "You Know, for Search"
}
```

---

## 3. Kibana 설치

### 3.1 다운로드 및 압축 해제
```bash
cd ~/elastic-stack

# Kibana 다운로드
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.11.0-linux-x86_64.tar.gz

# 압축 해제
tar -xzf kibana-8.11.0-linux-x86_64.tar.gz
cd kibana-8.11.0
```

### 3.2 설정 파일 편집
```bash
vi config/kibana.yml
```

**config/kibana.yml 내용:**
```yaml
# 서버 설정
server.port: 5601
server.host: "0.0.0.0"

# Elasticsearch 연결
elasticsearch.hosts: ["http://localhost:9200"]

# 보안 비활성화
xpack.security.enabled: false
```

### 3.3 Kibana 시작
```bash
# 포그라운드 실행 (테스트용)
./bin/kibana

# 정상 실행 확인 후 Ctrl+C로 종료

# 백그라운드 실행
nohup ./bin/kibana > kibana.log 2>&1 &

# 확인
# 브라우저에서 http://localhost:5601 접속
```

---

## 4. Filebeat 설치

### 4.1 다운로드 및 압축 해제
```bash
cd ~/elastic-stack

# Filebeat 다운로드
wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.11.0-linux-x86_64.tar.gz

# 압축 해제
tar -xzf filebeat-8.11.0-linux-x86_64.tar.gz
cd filebeat-8.11.0
```

### 4.2 설정 파일 작성
```bash
# 기존 파일 백업
mv filebeat.yml filebeat.yml.bak

# 새 설정 파일 작성
vi filebeat.yml
```

**filebeat.yml 내용 (기본 버전):**
```yaml
# 로그 입력 설정
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /home/YOUR_USERNAME/test-logs/airlineMessageHandler_*.log
    
    # 항공사 코드 추출
    processors:
      - dissect:
          tokenizer: "%{}/airlineMessageHandler_%{airline}.log"
          field: "log.file.path"
      
      # 불필요한 메타데이터 제거
      - drop_fields:
          fields: ["agent", "ecs", "log", "input", "host"]
          ignore_missing: true
    
    fields:
      app: airline-handler
      env: test
    fields_under_root: true

# Elasticsearch 출력
output.elasticsearch:
  hosts: ["localhost:9200"]
  index: "airline-logs-%{+yyyy.MM.dd}"

# 인덱스 템플릿 설정
setup.template.name: "airline-logs"
setup.template.pattern: "airline-logs-*"
setup.ilm.enabled: false

# 로그 레벨
logging.level: error
logging.to_files: true
logging.files:
  path: /home/YOUR_USERNAME/elastic-stack/filebeat-8.11.0/logs
  name: filebeat
  keepfiles: 7
```

> **주의:** `YOUR_USERNAME`을 실제 사용자명으로 변경하세요.

### 4.3 설정 파일 작성 (고급 버전 - 로그 파싱 포함)
```yaml
# 로그 입력 설정
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /home/YOUR_USERNAME/test-logs/airlineMessageHandler_*.log
    
    processors:
      # 1. 항공사 코드 추출
      - dissect:
          tokenizer: "%{}/airlineMessageHandler_%{airline}.log"
          field: "log.file.path"
      
      # 2. 로그 메시지 파싱
      - dissect:
          tokenizer: "I, [%{timestamp} #%{pid}]  %{level} -- #<TCPSocket:0x%{socket_id}>: %{log_message}"
          field: "message"
          ignore_failure: true
      
      # 3. IP 주소 추출
      - dissect:
          tokenizer: "%{?prefix}(%{client_ip})%{?suffix}"
          field: "log_message"
          ignore_failure: true
      
      # 4. 메시지 타입 분류
      - script:
          lang: javascript
          source: >
            function process(event) {
              var msg = event.Get("log_message");
              if (!msg) return;
              
              if (msg.includes("SSQ received")) {
                event.Put("message_type", "SSQ_RECEIVED");
                event.Put("direction", "INBOUND");
              } else if (msg.includes("SSR sent")) {
                event.Put("message_type", "SSR_SENT");
                event.Put("direction", "OUTBOUND");
              } else if (msg.includes("Socket connection")) {
                event.Put("message_type", "CONNECTION");
              } else if (msg.includes("not allowed")) {
                event.Put("message_type", "BLOCKED");
                event.Put("alert_level", "WARNING");
              }
            }
      
      # 5. 불필요한 필드 제거
      - drop_fields:
          fields: ["agent", "ecs", "log", "input", "host", "message"]
          ignore_missing: true
    
    fields_under_root: true

# Elasticsearch 출력
output.elasticsearch:
  hosts: ["localhost:9200"]
  index: "airline-logs-%{+yyyy.MM.dd}"

# 인덱스 템플릿 설정
setup.template.enabled: false
setup.ilm.enabled: false

# 로그 레벨
logging.level: error
```

### 4.4 설정 테스트
```bash
# 설정 파일 문법 체크
./filebeat test config

# Elasticsearch 연결 테스트
./filebeat test output
```

### 4.5 Filebeat 시작
```bash
# 포그라운드 실행 (테스트용)
./filebeat -e -c filebeat.yml

# 정상 실행 확인 후 Ctrl+C로 종료

# 백그라운드 실행
nohup ./filebeat -e -c filebeat.yml > filebeat.log 2>&1 &
```

---

## 5. 테스트 로그 생성

### 5.1 로그 디렉토리 생성
```bash
mkdir -p ~/test-logs
```

### 5.2 로그 생성 스크립트 작성
```bash
vi ~/test-logs/generate_logs.sh
```

**generate_logs.sh 내용:**
```bash
#!/bin/bash

LOG_FILE="$HOME/test-logs/airlineMessageHandler_1E.log"

echo "Generating logs to: $LOG_FILE"
echo "Press Ctrl+C to stop"

while true; do
    TIMESTAMP=$(date '+%Y-%m-%dT%H:%M:%S.%6N')
    PID=$$
    MESSAGES=(
        "Socket connection occured(192.168.150.3)"
        "Address is not allowed. Socket disconnected(192.168.150.3)"
        "SSQ received (Airline --> iAPP)"
        "SSR sent (Airline : $MSG" >> "$LOG_FILE"
    
    sleep 1
done
```

### 5.3 스크립트 실행
```bash
# 실행 권한 부여
chmod +x ~/test-logs/generate_logs.sh

# 로그 생성 시작
~/test-logs/generate_logs.sh &

# 확인
tail -f ~/test-logs/airlineMessageHandler_1E.log
```

### 5.4 여러 항공사 로그 생성 (선택사항)
```bash
vi ~/test-logs/generate_all_logs.sh
```

**generate_all_logs.sh 내용:**
```bash
#!/bin/bash

AIRLINES=("1E" "DL" "RF" "BX" "LJ1" "XA1")

for airline in "${AIRLINES[@]}"; do
    LOG_FILE="$HOME/test-logs/airlineMessageHandler_${airline}.log"
    
    (
        while true; do
            TIMESTAMP=$(date '+%Y-%m-%dT%H:%M:%S.%6N')
            PID=$$
            MESSAGES=(
                "Socket connection occured(192.168.150.3)"
                "Address is not allowed. Socket disconnected(192.168.150.3)"
                "SSQ received (Airline --> iAPP)"
                "SSR sent (Airline : $MSG" >> "$LOG_FILE"
            
            sleep $((RANDOM % 5 + 1))
        done
    ) &
    
    echo "Started log generator for airline: $airline (PID: $!)"
done

echo ""
echo "All log generators started!"
echo "Log files: $HOME/test-logs/airlineMessageHandler_*.log"
echo "Stop with: pkill -f generate_all_logs"
```
```bash
chmod +x ~/test-logs/generate_all_logs.sh
~/test-logs/generate_all_logs.sh
```

---

## 6. 확인 및 검증

### 6.1 Elasticsearch 데이터 확인
```bash
# 인덱스 목록
curl "localhost:9200/_cat/indices?v"

# 문서 개수
curl "localhost:9200/airline-logs-*/_count"

# 실제 데이터 조회
curl "localhost:9200/airline-logs-*/_search?size=1&pretty"
```

### 6.2 프로세스 확인
```bash
# 실행 중인 프로세스
ps aux | grep -E 'elasticsearch|kibana|filebeat'

# 포트 확인
netstat -tlnp | grep -E '9200|5601'
```

### 6.3 리소스 사용량 확인
```bash
# CPU 및 메모리
top

# 디스크 사용량
du -sh ~/elastic-stack/elasticsearch-8.11.0/data/
```

---

## 7. Kibana에서 검색

### 7.1 Index Pattern 생성

1. 브라우저에서 `http://localhost:5601` 접속
2. 좌측 메뉴 → `Management` → `Stack Management`
3. `Kibana` → `Data Views` (또는 `Index Patterns`)
4. `Create data view` 클릭
5. 설정:
   - **Name:** `airline-logs-*`
   - **Timestamp field:** `@timestamp`
6. `Save data view` 클릭

### 7.2 Discover에서 로그 확인

1. 좌측 메뉴 → `Analytics` → `Discover`
2. 상단에서 `airline-logs-*` 선택
3. 시간 범위: `Last 15 minutes`
4. 로그가 표시되면 성공! ✅

### 7.3 기본 검색 예시
```
# 특정 항공사
airline:1E

# 메시지 검색
message:"Socket connection"

# 조합 검색
airline:1E AND message:"SSQ received"

# 차단된 IP
message:"Address is not allowed"

# 시간 범위
@timestamp:[2025-10-23T19:00:00 TO 2025-10-23T20:00:00]
```

### 7.4 필드 선택

좌측 `Available fields`에서 표시할 필드 선택:
- ✅ `@timestamp`
- ✅ `airline`
- ✅ `message_type`
- ✅ `log_message`
- ✅ `client_ip`

### 7.5 실시간 모니터링

1. 우측 상단 시간 선택기 옆 `Auto-refresh` 아이콘 클릭
2. `5 seconds` 선택
3. 화면이 5초마다 자동 갱신됨 (tail -f 효과)

---

## 8. 자동 시작 설정

### 8.1 시작 스크립트 작성
```bash
vi ~/elastic-stack/start-all.sh
```

**start-all.sh 내용:**
```bash
#!/bin/bash

echo "Starting Elasticsearch..."
cd ~/elastic-stack/elasticsearch-8.11.0
./bin/elasticsearch -d

echo "Waiting for Elasticsearch..."
sleep 15

echo "Starting Kibana..."
cd ~/elastic-stack/kibana-8.11.0
nohup ./bin/kibana > kibana.log 2>&1 &

echo "Waiting for Kibana..."
sleep 10

echo "Starting Filebeat..."
cd ~/elastic-stack/filebeat-8.11.0
nohup ./filebeat -e -c filebeat.yml > filebeat.log 2>&1 &

echo ""
echo "All services started!"
echo "Elasticsearch: http://localhost:9200"
echo "Kibana: http://localhost:5601"
echo ""
echo "Check status:"
echo "  curl http://localhost:9200"
echo "  ps aux | grep -E 'elasticsearch|kibana|filebeat'"
```

### 8.2 종료 스크립트 작성
```bash
vi ~/elastic-stack/stop-all.sh
```

**stop-all.sh 내용:**
```bash
#!/bin/bash

echo "Stopping all services..."

pkill -f elasticsearch
pkill -f kibana
pkill -f filebeat

sleep 3

echo "All services stopped!"
echo ""
echo "Verify:"
echo "  ps aux | grep -E 'elasticsearch|kibana|filebeat'"
```

### 8.3 상태 확인 스크립트 작성
```bash
vi ~/elastic-stack/status.sh
```

**status.sh 내용:**
```bash
#!/bin/bash

echo "=========================================="
echo "  ELK Stack Status"
echo "=========================================="
echo ""

echo "=== Elasticsearch ==="
if curl -s http://localhost:9200 > /dev/null 2>&1; then
    echo "✅ Running on port 9200"
    curl -s http://localhost:9200 | grep -E 'cluster_name|version'
else
    echo "❌ Not running"
fi

echo ""
echo "=== Kibana ==="
if curl -s http://localhost:5601/api/status > /dev/null 2>&1; then
    echo "✅ Running on port 5601"
else
    echo "❌ Not running"
fi

echo ""
echo "=== Filebeat ==="
if pgrep -f filebeat > /dev/null; then
    echo "✅ Running (PID: $(pgrep -f filebeat))"
else
    echo "❌ Not running"
fi

echo ""
echo "=== Indices ==="
curl -s "localhost:9200/_cat/indices/airline-logs-*?v" 2>/dev/null || echo "No indices found"

echo ""
echo "=== Document Count ==="
curl -s "localhost:9200/airline-logs-*/_count?pretty" 2>/dev/null | grep count || echo "No documents"

echo ""
echo "=========================================="
```

### 8.4 실행 권한 부여
```bash
chmod +x ~/elastic-stack/start-all.sh
chmod +x ~/elastic-stack/stop-all.sh
chmod +x ~/elastic-stack/status.sh
```

### 8.5 사용 방법
```bash
# 전체 시작
~/elastic-stack/start-all.sh

# 상태 확인
~/elastic-stack/status.sh

# 전체 종료
~/elastic-stack/stop-all.sh
```

---

## 9. 문제 해결

### 9.1 Elasticsearch 시작 안됨
```bash
# 로그 확인
cat ~/elastic-stack/elasticsearch-8.11.0/logs/*.log

# 일반적인 문제:
# 1. 메모리 부족 → jvm.options에서 -Xms1g -Xmx1g로 줄이기
# 2. 포트 충돌 → elasticsearch.yml에서 포트 변경
# 3. 권한 문제 → chmod -R 755 ~/elastic-stack/elasticsearch-8.11.0
```

### 9.2 Kibana 접속 안됨
```bash
# Kibana 로그 확인
cat ~/elastic-stack/kibana-8.11.0/kibana.log

# Elasticsearch 연결 확인
curl http://localhost:9200

# Kibana 프로세스 확인
ps aux | grep kibana

# 재시작
pkill -f kibana
cd ~/elastic-stack/kibana-8.11.0
nohup ./bin/kibana > kibana.log 2>&1 &
```

### 9.3 Filebeat 데이터 안 들어옴
```bash
# Filebeat 로그 확인
cat ~/elastic-stack/filebeat-8.11.0/logs/filebeat

# 설정 테스트
cd ~/elastic-stack/filebeat-8.11.0
./filebeat test config
./filebeat test output

# 로그 파일 경로 확인
ls -la ~/test-logs/*.log

# Filebeat 재시작
pkill filebeat
nohup ./filebeat -e -c filebeat.yml > filebeat.log 2>&1 &
```

### 9.4 디스크 부족
```bash
# 디스크 사용량 확인
df -h

# 오래된 인덱스 삭제
curl -X DELETE "localhost:9200/airline-logs-2025.09.*"

# 데이터 디렉토리 정리
du -sh ~/elastic-stack/elasticsearch-8.11.0/data/*
```

### 9.5 검색 느림
```bash
# 캐시 클리어
curl -X POST "localhost:9200/_cache/clear"

# 인덱스 최적화
curl -X POST "localhost:9200/airline-logs-*/_forcemerge?max_num_segments=1"

# Elasticsearch 재시작
pkill -f elasticsearch
cd ~/elastic-stack/elasticsearch-8.11.0
./bin/elasticsearch -d
```

---

## 부록 A: 30일 자동 삭제 설정

### curator 설치 및 설정
```bash
# curator 설치
pip3 install elasticsearch-curator

# 설정 파일 작성
vi ~/curator-action.yml
```

**curator-action.yml 내용:**
```yaml
actions:
  1:
    action: delete_indices
    description: "30일 지난 인덱스 삭제"
    options:
      ignore_empty_list: True
    filters:
      - filtertype: pattern
        kind: prefix
        value: airline-logs-
      - filtertype: age
        source: name
        direction: older
        timestring: '%Y.%m.%d'
        unit: days
        unit_count: 30
```

### crontab 등록
```bash
# crontab 편집
crontab -e

# 매일 새벽 2시 실행
0 2 * * * curator ~/curator-action.yml >> ~/curator.log 2>&1
```

---

## 부록 B: 유용한 명령어 모음

### Elasticsearch
```bash
# 클러스터 상태
curl "localhost:9200/_cluster/health?pretty"

# 모든 인덱스 목록
curl "localhost:9200/_cat/indices?v"

# 특정 인덱스 크기
curl "localhost:9200/_cat/indices/airline-logs-*?v&h=index,store.size"

# 인덱스 삭제
curl -X DELETE "localhost:9200/airline-logs-2025.10.01"

# 전체 검색
curl "localhost:9200/airline-logs-*/_search?pretty"

# 특정 항공사 검색
curl -X GET "localhost:9200/airline-logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "term": {"airline": "1E"}
  }
}
'
```

### 로그 관리
```bash
# 로그 생성 중지
pkill -f generate_logs

# 로그 파일 확인
ls -lh ~/test-logs/

# 로그 파일 삭제
rm ~/test-logs/*.log

# 로그 실시간 확인
tail -f ~/test-logs/airlineMessageHandler_1E.log
```

### 프로세스 관리
```bash
# 전체 프로세스 확인
ps aux | grep -E 'elasticsearch|kibana|filebeat'

# 특정 프로세스 종료
pkill -f elasticsearch
pkill -f kibana
pkill -f filebeat

# 메모리 사용량
ps aux | grep elasticsearch | awk '{print $6/1024 "MB"}'
```

---

## 부록 C: 체크리스트

### 설치 체크리스트

- [ ] Java 17 설치 확인
- [ ] Elasticsearch 설치 및 실행 (포트 9200)
- [ ] Kibana 설치 및 실행 (포트 5601)
- [ ] Filebeat 설치 및 설정
- [ ] 테스트 로그 생성
- [ ] Kibana에서 Index Pattern 생성
- [ ] Discover에서 로그 확인
- [ ] 자동 시작 스크립트 작성

### 일일 체크리스트

- [ ] Elasticsearch 상태 확인 (`curl localhost:9200`)
- [ ] Kibana 접속 확인 (`http://localhost:5601`)
- [ ] 디스크 사용량 확인 (`df -h`)
- [ ] 로그 수집 확인 (Discover에서 최근 데이터)
- [ ] 인덱스 개수 확인 (30개 이하 유지)

---

## 참고 자료

- Elasticsearch 공식 문서: https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
- Kibana 공식 문서: https://www.elastic.co/guide/en/kibana/current/index.html
- Filebeat 공식 문서: https://www.elastic.co/guide/en/beats/filebeat/current/index.html

---

**설치 완료!** 🎉

이제 SSH 없이 웹 브라우저에서 모든 로그를 검색할 수 있습니다.
````

 