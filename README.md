# Linux 시스템 로그 분석
> 기간 : 2026.03.19

<br />

## 👩🏻‍💻 팀원 소개

| <img src="https://avatars.githubusercontent.com/u/118096607?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/170337448?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/98938482?v=4" width="150" height="150"/> |  <img src="https://avatars.githubusercontent.com/u/105345819?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/73377952?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/130548845?v=4" width="150" height="150"/> |
| :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: |  
|                 서가영<br/>[@caminobelllo](https://github.com/caminobelllo)                 |                       김민채<br/>[@minchaeki](https://github.com/minchaeki)                       |                김종연<br/>[@jongyeon0214](https://github.com/jongyeon0214)                |                박성준<br/>[@Sungjun24s](https://github.com/Sungjun24s)                |                백주연<br/>[@juyeonbaeck](https://github.com/juyeonbaeck)                |                이채유<br/>[@chaeyuuu](https://github.com/chaeyuuu)                |   


<br />

---

## 📌 프로젝트 개요

카드사 시스템 운영팀은 서비스 안정성 확보를 위해 실시간으로 시스템 로그를 수집하고 분석합니다. 해당 실습을 통해 실제 카드사 운영 환경에서 발생할 수 있는 이상 상황을 리눅스 명령어를 활용하여 탐지하고 대응하며 수집된 로그를 ELK 스택(Elasticsearch, Logstash, Kibana)으로 시각화하는 시나리오로 시스템 운영 관점과 보안 관점으로 구성됩니다.

**시스템 운영 관점**

서비스 운영 중 발생하는 용량 사용 이상, I/O 요청 이상, 성능 저하를 `system.log` 를 통해 조기에 탐지하고 장애 발생 전에 선제적으로 대응합니다.

**보안 관점**

외부 침입자가 터미널을 통해 DB에 접근하여 파일을 탐색하는 시도를 탐지하고 자동 차단하는 스크립트와 해외 결제 이상 거래를 실시간으로 감지하고 차단하는 스크립트를 구현합니다.

---

## 💻 PART 1. 시스템 운영 관점

### 📁 테스트 데이터 구성

```
~/card-logs/
└── system_log.csv   # DB I/O 로그 (1,000건)
```

### system_log.csv 컬럼 설명

|컬럼|필드명|설명|
|---|---|---|
|$1|`timestamp`|로그 발생 시각 (예: 2026-03-19T09:00:00)|
|$2|`db`|데이터베이스 이름 (orders, billing, inventory)|
|$3|`host`|서버 호스트 (db01, db02)|
|$4|`operation`|작업 유형 (write, read, temp_write, checkpoint, flush, bulk_insert, backup)|
|$5|`target`|대상 경로 (/data/pg_wal, /data/tmp, /data/main, /data/archive, /data/index)|
|$6|`bytes`|처리 데이터 크기 (bytes)|
|$7|`iops`|초당 I/O 요청 수|
|$8|`latency_ms`|응답 지연시간 (ms)|
|$9|`session_id`|세션 ID|
|$10|`query_id`|쿼리 ID|
|$11|`status`|상태 (OK / WARN / CRIT)|
|$12|`message`|상세 메시지|

### 📊 임계치 설정 근거

카드사 DB I/O 로그 분석에서 사용하는 임계치는 **AWS EBS 스펙**, **PostgreSQL 운영 기준**, **금융권 SLA** 세 가지 근거를 기반으로 설정합니다.

|지표|WARN|CRIT|근거|
|---|---|---|---|
|bytes|50MB (52,428,800)|100MB (104,857,600)|PostgreSQL OLTP 특성상 단일 트랜잭션 100MB 초과는 WAL 폭증 또는 배치 쿼리 의심|
|iops|1,500|3,000|AWS EBS gp3 기본 스펙 3,000 IOPS의 50% 도달 시점 / 한계치 기준|
|latency_ms|50ms|100ms|카드 결제 SLA 200~300ms 기준, DB I/O 단독 100ms 초과 시 고객 체감 수준|

---

## 🕹️ 설정 시나리오

### 1️⃣ 용량 쓰기 이상 탐지 (grep + awk)

- **상황** DB I/O 로그에서 대용량 쓰기가 특정 경로에 집중되고 있는지 확인해야 합니다. 큰 wirte가 어디 위치하고, 어느 target에 얼마나 몰렸는지 파악해야 합니다.
- **목표** 100MB 초과 쓰기 로그를 추출하고, target 경로별 총 bytes 합계를 집계하여 쓰기가 집중된 경로를 찾습니다. 임계치: WARN 50MB / CRIT 100MB

**Linux**

```bash
# WARN/CRIT 상태 로그만 빠르게 추출
grep -E ',WARN,|,CRIT,' db_io_dummy_logs_1000.csv

# CRIT 상태이면서 100MB 초과인 로그 추출
grep ',CRIT,' db_io_dummy_logs_1000.csv | awk -F, '$6 > 104857600'

# target 경로별 총 bytes 합계 (많은 순 정렬)
awk -F, 'NR>1 {sum[$5]+=$6} END {for (t in sum) print t, sum[t]}' db_io_dummy_logs_1000.csv | sort -k2 -nr
```

**Jq**

```bash
# awk 결과를 JSON 배열로 변환 후 jq로 포맷
awk -F, 'NR>1 {sum[$5]+=$6} END {
  printf "["
  first=1
  for (t in sum) {
    if (!first) printf ","
    printf "{\\"target\\":\\"%s\\",\\"bytes\\":%d}", t, sum[t]
    first=0
  }
  printf "]"
}' system_log.csv | jq 'sort_by(-.bytes)'

# CRIT 로그만 JSON으로 변환 후 특정 필드만 추출
grep ',CRIT,' system_log.csv | awk -F, '{
  printf "{\\"timestamp\\":\\"%s\\",\\"target\\":\\"%s\\",\\"bytes\\":%d,\\"iops\\":%d,\\"latency\\":%d}\\n",
  $1,$5,$6,$7,$8
}' | jq 'select(.bytes > 104857600)'
```

- 실행 결과

```
/data/pg_wal   32816171780
/data/tmp      25630801491
/data/main     10499077854
/data/archive   6609063313
/data/index       25620480
```

WAL 로그 폭증 → 복제 지연 또는 디스크 풀 위험 신호

대형 쿼리의 sort/join 연산이 메모리를 초과해 디스크로 spill되고 있는 상황

grep으로 WARN/CRIT를 먼저 걸러낸 뒤 awk로 집계하면 전체 1,000건을 다 읽지 않아도 되어 효율적

🔧 **대응 조치:** `/data/pg_wal` 디스크 사용률 즉시 확인 → WAL 아카이빙 설정 점검 → 필요 시 pg_wal 경로 디스크 증설

---

### 2️⃣ I/O 요청 이상 탐지 (grep + awk)

- **상황** 특정 세션이나 쿼리가 반복적으로 높은 I/O를 발생시키고 있는지 확인해야 합니다. 순간적인 iops 급증뿐 아니라 누가 지속적으로 부하를 유발하는지 추적해야 합니다.
- **목표** iops 3,000 초과 로그를 추출하고, session별 반복 이상 행위를 집계하여 부하 유발 원인을 찾습니다. 임계치: WARN 1,500 IOPS / CRIT 3,000 IOPS

**Linux**

```bash
# operation별 개수 집계
awk -F, 'NR>1 && ($4=="temp_write" || $4=="checkpoint" || $4=="flush") {cnt[$4]++} END {for (op in cnt) print op, cnt[op]}' db_io_dummy_logs_1000.csv

# iops 1,500 초과 로그 추출 (WARN 기준)
awk -F, 'NR==1 || $7 > 1500' db_io_dummy_logs_1000.csv

# iops 3,000 초과 로그 추출 (CRIT 기준)
awk -F, 'NR==1 || $7 > 3000' db_io_dummy_logs_1000.csv
```

**Jq**

```bash
# session별 반복 이상 행위 집계 (bytes + iops 누적, 많은 순 정렬)
jq -r '
  group_by(.session_id)
  | map({
      session_id: .[0].session_id,
      count: length,
      total_bytes: (map(.bytes) | add),
      total_iops: (map(.iops) | add)
    })
  | sort_by(.count) | reverse
  | .[]
  | "\\(.session_id) 건수=\\(.count) bytes=\\(.total_bytes) iops=\\(.total_iops)"
' db_io_dummy_logs_1000.json

# 특정 session이 temp_write를 반복하는지 확인
jq -r '
  map(select(.operation == "temp_wte"))
  | group_by(.session_id + "," + .query_id)
  | map({
      key: (.[0].session_id + "," + .[0].query_id),
      count: length
    })
  | sort_by(.count) | reverse
  | .[]
  | "\\(.key) \\(.count)"
' db_io_dummy_logs_1000.json
```

- **실행 결과**

```
[operation별 개수]
checkpoint   99건
temp_write   85건
flush        45건
전체 합계    229건

[iops 초과 건수]
iops 1,500 초과  261건
iops 3,000 초과   94건

[session별 반복 이상 행위 top5]
s777  건수=45  bytes=22,822,084,513  iops=190,304
s888  건수=34  bytes=17,547,461,944  iops=151,283
s999  건수=21  bytes=10,068,867,382  iops= 95,183
s521  건수=19  bytes= 2,253,629,559  iops= 34,030
s520  건수=18  bytes= 2,346,572,250  iops= 35,268

[temp_write 반복 session top5]
s777,q9901   12회
s888,q9902    8회
s999,q9903    5회
s521,q7002    5회
s523,q7001    3회
```

`checkpoint`가 99건으로 가장 많은 것은 WAL 버퍼가 자주 flush되고 있다는 신호입니다. checkpoint 빈도가 높으면 Disk I/O 부하가 지속적으로 발생합니다.

`s777`이 45건, 22.8GB, iops 190,304을 기록한 것은 단일 세션이 전체 I/O 부하의 상당 부분을 차지하고 있음을 의미합니다. `s777,q9901` 조합이 temp_write를 12회 반복하는 것과 연결하면 해당 쿼리가 메모리를 초과해 디스크 spill을 반복하는 것으로 판단할 수 있습니다.

iops 3,000 초과(gp3 한계치)가 94건 발생한 것은 EBS 볼륨 업그레이드 또는 Read Replica 분산을 검토해야 하는 수준입니다.

🔧 **대응 조치:** `s777` 세션 즉시 종료 → `q9901` 쿼리 실행계획 분석 → `work_mem` 증설 검토 → EBS gp3 → io2 업그레이드 고려

---

### 3️⃣ 복합 이상 징후 탐지 — 위험 순위 출력 (awk)

- **상황** 단일 지표 이상이 아니라 bytes + iops + latency가 동시에 임계치를 초과하는 강한 이상 징후를 찾아야 합니다. 어떤 session이 가장 위험한지 순위를 뽑아 대응 우선순위를 결정해야 합니다.
- **목표** bytes 100MB + iops 3,000 + latency 100ms 복합 조건으로 강한 이상 징후를 추출하고, session별 위험 순위를 출력합니다. 임계치: bytes CRIT 100MB + iops CRIT 3,000 + latency CRIT 100ms 동시 초과

**Linux**

```bash
# 복합 이상 징후 추출 (세 지표 모두 CRIT 기준 초과 + WARN/CRIT 상태)
awk -F, 'NR==1 || ($6 > 104857600 && $7 > 3000 && $8 > 100 && ($11=="WARN" || $11=="CRIT"))' db_io_dummy_logs_1000.csv

# 반복적으로 high bytes + high latency를 내는 session 순위
awk -F, 'NR>1 && $6 > 104857600 && $8 > 100 {cnt[$9]++; bytes[$9]+=$6} END {for (s in cnt) print s, cnt[s]"회", bytes[s]"bytes"}' db_io_dummy_logs_1000.csv | sort -k2 -nr

# 분 단위 bytes burst 구간 탐지 (500MB 초과 구간만)
awk -F, 'NR>1 {minute=substr($1,1,16); sum[minute]+=$6} END {for (m in sum) if (sum[m] > 500000000) print m, sum[m]}' db_io_dummy_logs_1000.csv | sort -k2 -nr

```

**Jq**

```bash
# 에러 메시지 있는 로그에서 target별 건수 집계
jq -r '
  .[]
  | select(.message | test("disk full|spill|fsync slow|archive failed|flush spike"; "i"))
  | .target
' db_io_dummy_logs_1000.json \\
| sort \\
| uniq -c

# 복합 이상 + 에러 메시지 동시 조건 (가장 강한 이상 징후)
jq '
  .[]
  | select(
      (.operation == "temp_write" or .operation == "checkpoint" or .operation == "flush")
      and (.latency_ms > 100)
      and (.message | test("disk full|spill|fsync slow|archive failed"))
    )
' db_io_dummy_logs_1000.json
```

- **실행 결과**

```
[복합 이상 징후 건수]
세 지표 동시 초과 (bytes 100MB + iops 3,000 + latency 100ms + CRIT)   94건
위험 target + WARN/CRIT                                               244건
에러 메시지 포함 로그                                                  138건

[session별 위험 순위]
s777   45회   22,822,084,513 bytes   ← 1순위 즉시 대응
s888   34회   17,547,461,944 bytes   ← 2순위
s999   21회   10,068,867,382 bytes   ← 3순위
s523   10회    1,553,791,490 bytes
s520   10회    1,427,229,673 bytes

[분 단위 burst 상위 5개]
2026-03-19T11:46   10,979,351,786 bytes   ← 최대 burst
2026-03-19T11:49   10,511,698,778 bytes
2026-03-19T11:40   10,294,207,441 bytes
2026-03-19T11:52    9,902,531,660 bytes
2026-03-19T11:43    8,751,738,286 bytes

[에러 메시지 target별 건수]
/data/tmp      51건
/data/pg_wal   50건
/data/main     25건
/data/archive  12건

[복합 이상 발생 시간대]
2026-03-19T11:40 ~ 11:52 구간에 집중 발생
```

복합 조건 필터(bytes + iops + latency 동시 초과)로 뽑힌 94건은 모두 11:40~11:52 구간에 집중되어 있습니다. 단순 이상이 아니라 특정 시간대에 시스템 전반이 한계에 도달한 상황입니다.

`s777`이 45회, 22.8GB를 기록한 것은 전체 CRIT 로그의 절반 가까이를 단일 세션이 유발했다는 의미입니다.

11:46분에 약 11GB가 집중된 것은 대형 배치 작업 또는 체크포인트가 겹친 시점으로, 해당 시간대 작업 스케줄을 분산해야 합니다.

🔧 **대응 조치:** `s777` 즉시 세션 종료 → `q990`

---

## 🔐 PART 2. 보안 관점

### 📁 테스트 데이터 구성

```
~/card-logs/
└── auth.json   # 로그인 감사 로그
└── payment.log (csv)   # 결제 트랜잭션 로그
```

### payment.log 컬럼 설명

|컬럼|필드명|설명|
|---|---|---|
|$1|`timestamp`|결제 요청 시간|
|$2|`User`|사용자 식별자|
|$3|`Amount`|결제 금액|
|$4|`Currency`|결제 통화|
|$5|`Merchant`|가맹점명|
|$6|`IP`|결제 요청 IP 주소|
|$7|`Status`|결제 처리 상태|

### auth.json 컬럼 설명

|타입|필드명|설명|
|---|---|---|
|string|`time`|접속 일시(YYYY-MM-DD HH:MM:SS)|
|string|`user`|사용자 고유 식별자|
|string|`ip`|접속 시도 IP 주소|
|string|`country`|IP 기반 국가 코드 (ISO 3166-1)|
|string|`status`|로그인 성공 여부|
|string|`device`|접속 기기 종류|

---

## 🕹️ 설정 시나리오 1

> 외부 침입자가 터미널을 통해 DB에 접근하여 파일을 탐색하는 시도를 탐지하고 자동 차단하는 스크립트와, **해외결제 이상 거래를 실시간으로 감지하고 차단하는 스크립트**를 구현합니다. 즉, 리눅스 인프라 기술을 활용한 카드사 FDS(부정결제 탐지 시스템)의 프로토타입 설계를 위한 예제 입니다.

### 1️⃣ 로그인 실패 감사 (grep + jq)

- **상황** 최근 해킹 시도가 의심되어 로그인에 실패한 기록만 따로 추출하여 보고해야 합니다.
- **목표**`auth.json`에서 로그인 실패 기록만 추출하여 `/tmp/login_failure.json`에 저장합니다.

```bash
# 방법 A — grep (빠르고 간단)
grep '"status": "fail"' auth.json > /tmp/login_failure.json

# 방법 B — jq (정확한 JSON 키 매칭, 권장)
jq -c 'select(.status == "fail")' auth.json > /tmp/login_failure.json

# 결과 확인
wc -l /tmp/login_failure.json
```

`>` 는 파일을 새로 생성/덮어쓰기, `>>` 는 기존 파일에 누적 저장합니다. `jq`를 사용하면 `"status_fail"` 같은 유사 키도 오탐 없이 정확히 필터링할 수 있습니다.

🔧 **대응 조치:** 실패 기록 57건 보안팀 공유 → 반복 실패 IP 차단 검토

---

### 2️⃣ 특정 대역 접속 추적 (grep + awk)

- **상황** 특정 IP 대역(211.234.x.x)에서 발생하는 결제 요청을 전수 조사해야 합니다.
- **목표**`payment.log`에서 211.234.x.x 대역 결제 요청 건수를 확인합니다.

```bash
# 방법 A — grep -c (카운팅 플래그)
grep -c '211\\\\.234\\\\.' payment.log

# 방법 B — grep + wc -l (파이프 방식)
grep '211\\\\.234\\\\.' payment.log | wc -l

# 방법 C — awk (IP 컬럼($6)만 정확히 검사, 오탐 방지)
awk -F, '$6 ~ /^211\\\\.234\\\\./ {count++} END {print count}' payment.log
```

`211.234.`처럼 `\\\\.` 없이 쓰면 `2112344` 같은 문자열도 매칭될 수 있습니다. `awk`의 `$6 ~ /패턴/` 방식은 특정 컬럼만 검사하므로 더 정밀합니다.

🔧 **대응 조치:** 97건 전수 조사 → 비정상 패턴 확인 시 해당 대역 결제 일시 차단

---

### 3️⃣ 봇 의심 패턴 탐지 (grep + awk)

- **상황** 사람이 아닌 봇(Bot) 프로그램에 의한 자동 결제인지 확인해야 합니다. 로그인 후 비정상적으로 짧은 시간 내에 결제가 발생하는 패턴을 탐지합니다.
- **목표** 특정 IP의 로그인 시각과 결제 시각을 비교하여 봇 의심 여부를 판단합니다.

```bash
# 결제 로그에서 해당 IP의 거래 내역 추출
grep "54.12.33.1" payment.log | awk -F, '{print "시간: "$1, "상점: "$5, "금액: "$3}'

# auth.json에서 같은 IP의 로그인 성공 시각 확인
grep "54.12.33.1" auth.json | grep "success"
```

- **실행 결과**

```
시간: 2025-06-10 14:00:30 상점: Amazon US 금액: 1200.0
시간: 2025-06-10 14:00:45 상점: Netflix 금액: 9.99
```

**이상 징후 판단 근거**

|이벤트|시각|경과 시간|
|---|---|---|
|로그인 성공 (auth.json)|2025-06-10 14:00:15|기준|
|첫 결제 — Amazon US $1,200|2025-06-10 14:00:30|+15초|
|두 번째 결제 — Netflix $9.99|2025-06-10 14:00:45|+30초|

로그인 후 15초 이내 고액 결제 발생은 정상적인 사람의 행동 패턴으로 보기 어렵습니다. 봇 의심 계정으로 분류하여 추가 인증 요구 또는 차단 조치가 필요합니다.

🔧 **대응 조치:** `user_0077` 계정 잠금 → 본인 확인 후 해제 → 해당 결제 건 보류 처리

---

### 4️⃣ 고액 결제 이상 탐지 (awk)

- **상황** 동일 유저가 짧은 간격으로 금액을 키워 결제하는 카드 털이 전형적 패턴을 탐지합니다.
- **목표** USD 결제 중 $1,000 이상인 고액 건을 탐지하고 상태별 분포를 확인합니다.

```bash
# USD 결제 중 $1,000 이상인 고액 건 탐지
awk -F, '$4 == "USD" && $3 >= 1000 { print "고액 의심 거래! 사용자: " $2 ", 금액: $" $3 ", 상점: " $5 }' payment.log

# 상태(APPROVED/DECLINED/PENDING)별 분포 확인
awk -F, '$4=="USD" && $3>=1000 {print $7}' payment.log | sort | uniq -c
```

- **실행 결과**

```
고액 의심 거래! 사용자: user_0010, 금액: $1500.0, 상점: GS25 강남점
고액 의심 거래! 사용자: user_0077, 금액: $1200.0, 상점: Amazon US
고액 의심 거래! 사용자: user_0021, 금액: $1050.0, 상점: Apple Store
고액 의심 거래! 사용자: user_0009, 금액: $1800.0, 상점: Steam
```

**미션 3과 연계 분석 (user_0077)**

소액 테스트 결제 후 고액 결제를 시도하는 전형적인 카드 털이 패턴입니다. `&&`로 통화와 금액 조건을 동시에 걸면 오탐 없이 정확한 필터링이 가능합니다.

🔧 **대응 조치:** 탐지된 10건 결제 보류 → 카드 소유자 본인 확인 → 이상 확인 시 해당 카드 즉시 정지

---

### 5️⃣ 자동화 모니터링 구축 (find + crontab)

- **상황** 해외 접속 로그인(`country`가 `KR`이 아닌 건)을 실시간으로 감시하여 보안팀과 자동으로 공유해야 합니다.
- **목표**`find -mmin`으로 최근 변경 파일을 탐색하고, `crontab`으로 10분마다 자동 실행되는 모니터링 스크립트를 구축합니다.

**Step 1 — 셸 스크립트 작성**

```bash
# 파일 위치: /usr/local/bin/foreign_login_monitor.sh

#!/bin/bash

ALERT_FILE="/var/log/alerts/foreign_login.log"
LOG_DIR="/var/log/auth"

# 알림 디렉토리 없으면 생성
mkdir -p /var/log/alerts

# 최근 10분 내 수정된 auth.json 파일을 순회
find $LOG_DIR -name "auth.json" -mmin -10 | \\\\
while read FILE; do
    grep -v '"country": "KR"' "$FILE" >> $ALERT_FILE
done
```

**Step 2 — 실행 권한 부여 및 crontab 등록**

```bash
# 실행 권한 부여
chmod +x /usr/local/bin/foreign_login_monitor.sh

# crontab 등록
crontab -e
```

```
# 매 10분마다 해외 로그인 감지 스크립트 실행
*/10 * * * * /usr/local/bin/foreign_login_monitor.sh
```

- **실행 결과**

```json
{"time": "2025-06-10 14:00:15", "user": "user_0077", "ip": "54.12.33.1", "country": "US", "status": "success"}
{"time": "2025-06-08 04:18:00", "user": "user_0023", "ip": "5.101.0.12", "country": "RU", "status": "fail"}
{"time": "2025-06-12 01:25:00", "user": "user_0007", "ip": "194.165.16.6", "country": "RU", "status": "fail"}
```

`find -mmin -10`은 최근 10분 이내 수정된 파일만 탐색하므로 전체 로그를 매번 스캔하지 않아 효율적입니다. `>>`로 누적 저장하면 이전 알림 기록이 덮어쓰이지 않습니다.

🔧 **대응 조치:** 10분 주기 자동 탐지 → 알림 파일 보안팀 Slack 연동 → 임계 건수 초과 시 자동 IP 차단 룰 추가

---

## 🕹️ 설정 시나리오 2

> **1단계**: HTTP 헤더에 OGNL 표현식을 삽입해 서버에서 원격 명령을 실행하는 Struts RCE 공격을 `grep`으로 탐지합니다.

> **2단계**: 침투 성공 후 공격자가 `find`로 평문 비밀번호 파일을 수집한 행적을 `bash_history`로 추적합니다.

### 1️⃣ 최초 침투 탐지 (grep)

- **상황**: HTTP Content-Type 헤더에 OGNL 표현식이 삽입된 요청이 반복 탐지 → 공격 패턴을 추출하여 분석해야 하는 상황입니다.
- **목표**: `access.log`에서 OGNL 인젝션 패턴을 탐지하고 공격자 IP와 빈도를 확인합니다.

```bash
# 1. OGNL/java.lang 패턴 탐지
grep -E "ognl|java.lang" /app/payment-system/logs/access.log

# 2. 공격자 IP 기준 전체 요청 추출
grep "10.10.10.99" access.log

# 3. HTTP 500 에러만 필터링
grep " 500 " access.log

# 4. 공격 횟수 카운트
grep -c "ognl" access.log

# 5. 시간대별 공격 빈도 분석
grep "10.10.10.99" access.log | awk '{print $4}' | cut -d: -f2 | sort | uniq -c
```

`E` 옵션은 확장 정규식을 활성화해 `|` OR 조건으로 여러 패턴을 동시에 탐지합니다.

`" 500 "` 처럼 공백으로 감싸야 URL 내 숫자 500과 구분할 수 있습니다. (1500, 5000 등과 구분해야 하므로)

🔧 **대응 조치**: 공격자 IP `10.10.10.99` 총 12회 탐지 → WAF Content-Type 헤더 필터링 적용 + 해당 IP 차단

---

### 2️⃣ 내부 탐색 & 권한 상승 탐지 (find + bash_history)

- **상황**: 초기 침투 이후 공격자가 서버 내부를 탐색하며 → 평문 비밀번호가 저장된 설정 파일을 수집한 정황이 포착되었습니다.
- **목표**: `bash_history`에서 공격자의 명령어 행적을 추적하고 노출된 설정 파일을 확인합니다.

```bash
# 공격자 명령어 — 설정 파일에서 비밀번호 검색
find /var/www -name "*.cfg" | xargs grep "password"
find /app -name "*.yaml" | xargs grep -i "password\\|passwd\\|pwd"

# Admin 대응 — bash_history로 행적 추적
cat ~/.bash_history | tail -n 20
grep -E "find|grep|mysqldump|curl" ~/.bash_history
```

`xargs`는 `find` 결과를 `grep`의 입력으로 연결해줍니다.

공격자가 `history -c`로 삭제를 시도해도 `/root/.bash_history` 파일에 기록이 남아 있습니다.

🔧 **대응 조치**: `db_config.yaml`, `server_auth.json` 등 6개 파일 평문 비밀번호 노출 확인 → `chmod 600` 적용 + AWS Secrets Manager로 교체

---

## 🔗 참고 자료

- [GNU awk 공식 문서](https://www.gnu.org/software/gawk/manual/)
- [crontab.guru](http://crontab.guru/) [— cron 표현식 검증기](https://crontab.guru/)
- [jq 공식 문서](https://jqlang.github.io/jq/manual/)
