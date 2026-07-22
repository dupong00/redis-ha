# Redis Sentinel 페일오버 실습

Redis Sentinel의 **자동 페일오버**가 실제로 어떻게 동작하는지 Docker로 확인하는 실습 환경입니다.
Master 1 + Replica 2 + Sentinel 3 구성을 띄운 뒤 마스터를 강제로 죽여
`감시 → 과반 판정 → 자동 승격 → 옛 마스터 재편입` 흐름을 로그로 관찰합니다.

## 구성

```
                 ┌─────────────┐
                 │ redis-master│ (6379)
                 └──────┬──────┘
              복제      │      복제
        ┌──────────────┴──────────────┐
 ┌──────▼───────┐              ┌───────▼──────┐
 │redis-replica1│(6380)        │redis-replica2│(6381)
 └──────────────┘              └──────────────┘

 sentinel1(26379) / sentinel2(26380) / sentinel3(26381)
```

| 컨테이너 | 역할 | 호스트 포트 |
|---|---|---|
| redis-master | 마스터 | 6379 |
| redis-replica1 | 복제본 | 6380 |
| redis-replica2 | 복제본 | 6381 |
| sentinel1~3 | 감시 · 페일오버 | 26379 / 26380 / 26381 |

포트가 다른 건 **호스트 매핑**뿐이고, 컨테이너 내부는 모두 Redis `6379` / Sentinel `26379` 입니다.
따라서 `docker exec`로 컨테이너 안에서 실행할 때는 항상 내부 포트를 씁니다.

## 요구 사항

- Docker Desktop (Docker Engine 20.10+)
- macOS / Linux

## 디렉토리 구조

```
sentinel/
├── docker-compose.yml
├── conf/
│   ├── master.conf
│   ├── replica.conf
│   └── sentinel.conf
└── README.md
```

## 실행

```bash
cd sentinel
docker compose up -d
docker compose ps        # 컨테이너 6개가 모두 Up
```

## 1. 복제 확인

```bash
docker exec -it redis-master redis-cli info replication
```

```
role:master
connected_slaves:2
slave0:ip=172.19.0.3,port=6379,state=online,offset=41837,lag=0
slave1:ip=172.19.0.4,port=6379,state=online,offset=41837,lag=1
master_failover_state:no-failover
master_replid:4c828588166579768ba05bdb77cd9e19d69eaae3
```

- `state=online` — 초기 동기화가 끝나고 실시간 복제 중
- `offset`이 `master_repl_offset`과 같으면 완전 동기화 상태. **페일오버 시 offset이 가장 앞선 replica가 새 마스터로 선택됩니다**

데이터 전파 확인:

```bash
docker exec -it redis-master   redis-cli set hello world
docker exec -it redis-replica1 redis-cli get hello    # -> "world"
docker exec -it redis-replica2 redis-cli get hello    # -> "world"
```

replica는 읽기 전용입니다:

```bash
docker exec -it redis-replica1 redis-cli set foo bar
# -> READONLY You can't write against a read only replica.
```

## 2. Sentinel 인식 확인

```bash
docker exec -it sentinel1 redis-cli -p 26379 sentinel master mymaster
```

```
name                  mymaster
ip                    redis-master
flags                 master
num-slaves            2
num-other-sentinels   2
quorum                2
```

- `num-other-sentinels: 2` — **conf에는 마스터 주소만 적었는데 나머지 Sentinel 2개를 스스로 발견했습니다.**
  Sentinel들은 마스터의 `__sentinel__:hello` pub/sub 채널에 자기 정보를 뿌려 서로를 찾습니다
- `ip`가 IP가 아닌 호스트명(`redis-master`)인 것은 `resolve-hostnames yes` 효과입니다

## 3. 페일오버 관찰

터미널 A — 로그를 먼저 띄웁니다.

```bash
docker logs -f sentinel1
```

터미널 B — 마스터를 죽입니다.

```bash
docker stop redis-master
```

### 실제 로그

```
02:05:42  # Failed to resolve hostname 'redis-master'
02:05:46  # +sdown master mymaster redis-master 6379
02:05:47  # +new-epoch 1
02:05:47  # +vote-for-leader e834aac75a593b383d315da135b2dd58dedd697c 1
02:05:48  # +odown master mymaster redis-master 6379 #quorum 3/2
02:05:48  * Next failover delay: I will not start a failover before ...
02:05:49  # +config-update-from sentinel e834aac7... 172.19.0.6 26379
02:05:49  # +switch-master mymaster redis-master 6379 172.19.0.3 6379
02:05:49  * +slave slave 172.19.0.4:6379 ... @ mymaster 172.19.0.3 6379
02:05:49  * +slave slave redis-master:6379 ... @ mymaster 172.19.0.3 6379
```

| 시각 | 사건 |
|---|---|
| 02:05:42 | 마스터 응답 없음 감지 시작 |
| 02:05:46 | `+sdown` — 주관적 다운 (5초 경과) |
| 02:05:47 | `+vote-for-leader` — 페일오버 리더 투표 |
| 02:05:48 | `+odown #quorum 3/2` — 객관적 다운 확정 |
| 02:05:49 | `+switch-master` — **새 마스터 전환 완료** |

`docker stop`부터 복구까지 **약 7초**. 이 중 5초는 `down-after-milliseconds`로 설정한 대기 시간이므로,
페일오버 자체는 2초 남짓에 끝났습니다.

### 로그에서 눈여겨볼 부분

- **`+vote-for-leader` + `Next failover delay`** — 이 Sentinel은 리더가 아니라 다른 Sentinel에게 투표한 뒤 물러났습니다.
  페일오버는 선출된 리더 **하나만** 지휘합니다
- **`+config-update-from sentinel ...`** — 스스로 판단한 게 아니라 리더의 결정을 전달받았다는 뜻입니다
- **`+slave slave redis-master:6379 ... @ mymaster 172.19.0.3`** — 아직 꺼져 있는 옛 마스터를
  **미리 새 마스터의 슬레이브로 등록**해 둡니다. 돌아왔을 때 마스터 자리를 되찾지 못하는 이유입니다

### 승격 확인

```bash
docker exec -it redis-replica1 redis-cli info replication
```

```
role:master
connected_slaves:1
slave0:ip=172.19.0.4,port=6379,state=online
master_replid :daac54c65eb814590373bef80d4487c2dbe84d46   # 새 ID
master_replid2:4c828588166579768ba05bdb77cd9e19d69eaae3   # 옛 마스터의 ID
second_repl_offset:119132
```

`master_replid2`가 **1번에서 본 옛 마스터의 `master_replid`와 같습니다.**
"나는 원래 저 마스터의 복제본이었고 119132 오프셋부터 이어받았다"는 이력으로,
나중에 돌아온 노드가 전체 재동기화 없이 부분 동기화(partial resync)할 수 있게 해줍니다.

## 4. 옛 마스터 복귀

```bash
docker start redis-master
docker exec -it redis-master redis-cli info replication
```

```
role:slave
master_host:172.19.0.3
master_link_status:up
slave_read_only:1
```

```
# sentinel1 로그
-sdown slave redis-master:6379 redis-master 6379 @ mymaster 172.19.0.3 6379
+slave slave 172.19.0.2:6379 172.19.0.2 6379 @ mymaster 172.19.0.3 6379
```

`-sdown` 뒤가 `master`가 아니라 **`slave`** 인 점이 핵심입니다.
Sentinel은 옛 마스터가 돌아오기 전에 이미 슬레이브로 분류해 두었고,
기동 직후 `REPLICAOF`를 내려 강등시킵니다. **마스터라고 우길 기회 자체가 없습니다.**

데이터도 보존됩니다 (`appendonly yes`로 복원 + 새 마스터로부터 동기화):

```bash
docker exec -it redis-master redis-cli get hello    # -> "world"
```

최종 상태 — 마스터만 바뀐 채 원래 구성으로 복구됩니다:

```bash
docker exec -it redis-replica1 redis-cli info replication
# role:master, connected_slaves:2  (replica2 + 옛 마스터)
```

## 정리

```bash
docker compose down       # 컨테이너 정리
docker compose down -v    # 볼륨까지 삭제
```

## 설정 메모

- **`sentinel monitor mymaster redis-master 6379 2`** — 끝의 `2`가 **quorum**. 다운 판정에 필요한 Sentinel 수입니다
- **Sentinel은 홀수(3, 5)로** — 다운 판정은 quorum이지만, 페일오버 **리더 선출에는 전체 과반**이 별도로 필요합니다.
  2개만 두면 하나가 죽었을 때 과반을 못 채워 페일오버가 시작되지 않습니다
- **conf를 `/tmp`로 복사해 기동** — Redis/Sentinel은 페일오버 시 `CONFIG REWRITE`로 설정 파일을 다시 씁니다.
  macOS Docker Desktop에서는 bind-mount된 **단일 파일**에 쓰면 `Device or resource busy`가 발생하므로,
  읽기 전용(`:ro`)으로 마운트한 뒤 컨테이너 안 복사본으로 실행합니다
- **검증은 `docker exec`로** — Sentinel이 알려주는 마스터 주소는 Docker 내부 IP라,
  호스트(맥)에서 `localhost`로 붙으면 그 IP로 접속되지 않습니다
- **`Failed to resolve hostname 'redis-master'`는 정상** — `resolve-hostnames yes` 사용 시 컨테이너를 stop하면
  Docker 내부 DNS에서 이름이 사라져 발생합니다. 다운 판정에는 영향이 없고, 컨테이너를 다시 켜면 사라집니다
- **`down-after-milliseconds`가 감지 속도를 좌우** — 짧으면 일시적 지연에도 페일오버가 터지고, 길면 장애 감지가 늦어집니다.
  데이터도 보존됩니다 (`appendonly yes`로 복원 + 새 마스터로부터 동기화):

  ```bash
  docker exec -it redis-master redis-cli get hello    # -> "world"
  ```

  최종 상태 — 마스터만 바뀐 채 원래 구성으로 복구됩니다:

  ```bash
  docker exec -it redis-replica1 redis-cli info replication
  # role:master, connected_slaves:2  (replica2 + 옛 마스터)
  ```

  ## 정리

  ```bash
  docker compose down       # 컨테이너 정리
  docker compose down -v    # 볼륨까지 삭제
  ```

  ## 설정 메모

  - **`sentinel monitor mymaster redis-master 6379 2`** — 끝의 `2`가 **quorum**. 다운 판정에 필요한 Sentinel 수입니다
  - **Sentinel은 홀수(3, 5)로** — 다운 판정은 quorum이지만, 페일오버 **리더 선출에는 전체 과반**이 별도로 필요합니다.
    2개만 두면 하나가 죽었을 때 과반을 못 채워 페일오버가 시작되지 않습니다
  - **conf를 `/tmp`로 복사해 기동** — Redis/Sentinel은 페일오버 시 `CONFIG REWRITE`로 설정 파일을 다시 씁니다.
    macOS Docker Desktop에서는 bind-mount된 **단일 파일**에 쓰면 `Device or resource busy`가 발생하므로,
    읽기 전용(`:ro`)으로 마운트한 뒤 컨테이너 안 복사본으로 실행합니다
  - **검증은 `docker exec`로** — Sentinel이 알려주는 마스터 주소는 Docker 내부 IP라,
    호스트(맥)에서 `localhost`로 붙으면 그 IP로 접속되지 않습니다
  - **`Failed to resolve hostname 'redis-master'`는 정상** — `resolve-hostnames yes` 사용 시 컨테이너를 stop하면
    Docker 내부 DNS에서 이름이 사라져 발생합니다. 다운 판정에는 영향이 없고, 컨테이너를 다시 켜면 사라집니다
  - **`down-after-milliseconds`가 감지 속도를 좌우** — 짧으면 일시적 지연에도 페일오버가 터지고, 길면 장애 감지가 늦어집니다.