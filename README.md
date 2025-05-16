# mate53key
A Practical, High-Performance Unique ID Generator for Oracle-based MES & Logistics Systems

mate53key는 중견 기업의 MES 및 물류 시스템에서 현실적인 요구를 반영하여 설계된 고성능 유일 키 생성기입니다.
Twitter의 Snowflake ID를 참고하되, 밀리초 기반 시퀀스 초기화와 클러스터 분산 환경의 복잡성을 피하고 NonSql DB가 아닌 RDBMS(Oracle)등을 전제로 최적화한 ID 체계입니다.

이름 의미(Naming)

	• mate: 회사명 ComputerMate의 일부
	• 53: JavaScript의 안전 정수 범위 (2^53 - 1)

## 설계 철학
“현실 기반의 유일성 보장, 복잡도 최소화”

	• 중견기업은 대형 클러스터가 필요하지 않으며, 초당 수천만 건의 트랜잭션도 발생하지 않음
 	• 밀리초 단위 시퀀스 초기화는 오버엔지니어링이며, 초 단위 기준으로도 충분한 유일성을 확보할 수 있음
 	• 시간 역전 문제는 비즈니스 설계와 트랜잭션 그룹화로 해소 가능

## Design Philosophy
“Practical uniqueness, not over-engineered complexity.”

	• Mid-sized companies don’t need distributed Snowflake-style clusters.
	• Per-second granularity is enough for almost all use cases.
	• Time-reversal is tolerable and can be logically grouped.

## 스펙 요약 (mate53key Structure)

| Component | Bits | Description 
|------------|------|------------------|
|Timestamp | 32bit | Based on secounds ( 136 Years usable )|
|Server ID | 4bit | Up to 16 Servers |
|Circle Seq | 17bit | 131K per second |
|Total | 53bit | Compatible with JavaScript safe integers|

## 시간 역전 현상 대응 전략(Time Reversal Strategy)
시퀀스는 순환 방식이며 초기화하지 않아, 서버에 대한 부하가 없음

Sequence never resets; rolling counter wraps naturally.

역전 확률 예시

	• 초당 20건 × 하루 86400초 = 1,728,000건
	• 17bit 시퀀스 (131,072) 대비 역전 확률: 약 0.015% 미만
    Theoretical reversal frequency: under 0.015% per day at 20 TPS
 
보완 방안(Complemented with)

	• created_at 컬럼으로 실제 트랜잭션 시간 명시 ( created_at: timestamp column for real order )
	• group_key 컬럼으로 사건 그룹화하여 정렬 문제 해소 ( group_key: groups logically related events )
	• 트랜잭션은 논리적 순서보다 사건의 정확한 분류가 더 중요
    Real-world example: Inbound/outbound auto-generated logistics transactions don’t require strict millisecond ordering.

## Snowflake의 단점과 mate53key의 개선점

| 항목 | Snowflake | mate53key |
|------|-----------|------------|
| Reset behavior | 밀리초 단위 초기화 필요 Millisecond-based | 초기화 없음 (순환 방식) Never resets |
| Time resolution| 밀리초 Millisecond | 초 (복잡도 감소) Second |
| Cluster support | 예 (서버 ID, 데이터센터 ID 필수) | 단일 시스템 또는 최대 16개 서버까지 사용 가능 Optional (up to 16) |
| Bit length | 63bit 이상 | 53bit |
| JavaScript-safe | ❌ (unsafe) | ✅ (safe) |
| Use complexity | High | Low (RDBMS native) |
| Use case | BigTech scale| Practical MES scale |

## Oracle Function 

<pre lang="markdown">
```sql
CREATE OR REPLACE FUNCTION MATE53KEY(p_server_id IN NUMBER) RETURN NUMBER IS
  -- 기준일: 2000-01-01
  l_epoch CONSTANT DATE := TO_DATE('2000-01-01', 'YYYY-MM-DD');
  l_now_seconds NUMBER;
  l_server_id   NUMBER := TRUNC(p_server_id);
  l_seq         NUMBER;
  l_key         NUMBER;
BEGIN
  -- 서버 ID 유효성 검사
  IF l_server_id < 0 OR l_server_id > 15 THEN
    RAISE_APPLICATION_ERROR(-20001, 'SERVER_ID must be between 0 and 15');
  END IF;

  -- 현재 시각 초단위로 변환 (기준일 기준)
  l_now_seconds := TRUNC((SYSDATE - l_epoch) * 24 * 60 * 60);

  -- 32bit 초 제한 검사
  IF l_now_seconds > POWER(2, 32) - 1 THEN
    RAISE_APPLICATION_ERROR(-20002, 'Timestamp exceeds 32bit second limit (~2136-01-01)');
  END IF;

  -- 순환 시퀀스 생성 (0~131071, 17bit)
  SELECT MOD(mate53_seq.NEXTVAL, POWER(2, 17))
    INTO l_seq
    FROM DUAL;

  -- 53bit key 구성
  -- bit layout: [ 32bit time ][ 4bit server ][ 17bit sequence ]
  l_key := l_now_seconds * POWER(2, 21) + l_server_id * POWER(2, 17) + l_seq;

  -- 53bit 범위 검사
  IF l_key > POWER(2, 53) - 1 THEN
    RAISE_APPLICATION_ERROR(-20003, 'Generated key exceeds 53-bit limit');
  END IF;

  RETURN l_key;
END;

CREATE SEQUENCE mate53_seq NOCACHE NOCYCLE;
```
</pre>

## License
MIT License. Contributions and usage welcome. Please give credit when possible. 

Created by Kihyun Ryu – ComputerMate Co., Ltd. SI Support Team Leader
