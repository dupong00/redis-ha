# Redis 고가용성(HA) 실습

Redis의 고가용성 구성을 **Docker로 직접 띄워서** 동작을 확인하는 실습 저장소입니다.
개념 비교에서 그치지 않고, 노드를 실제로 죽여보며 로그로 검증하는 데 목적이 있습니다.

## 실습 목록

| 구성 | 내용 | 상태 |
|---|---|---|
| [sentinel/](./sentinel) | Master 1 + Replica 2 + Sentinel 3. 마스터를 죽여 자동 페일오버 관찰 | ✅ 완료 |
| cluster/ | 슬롯 분배와 노드 장애 시 동작 확인 | 예정 |

## Sentinel 실습 요약

`docker stop redis-master` 이후 **약 7초** 만에 새 마스터로 전환됐습니다.

```
+sdown master mymaster redis-master 6379              # 주관적 다운 (5초 경과)
+odown master mymaster redis-master 6379 #quorum 3/2  # 객관적 다운 확정
+vote-for-leader ...                                  # 페일오버 리더 선출
+switch-master mymaster redis-master 6379 172.19.0.3 6379   # 전환 완료
```

죽었던 마스터를 다시 켜면 마스터 자리를 되찾지 않고 **새 마스터의 복제본으로 재편입**됩니다.

자세한 실행 방법과 검증 과정은 [sentinel/README.md](./sentinel/README.md) 참고.

## 요구 사항

- Docker Desktop (Docker Engine 20.10+)
- macOS / Linux
