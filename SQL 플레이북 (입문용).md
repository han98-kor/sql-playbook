# SQL 플레이북 (입문용)

> **목적**: 면접/코테/업무에서 바로 쓰는 최소한의 SQL 레시피만 담은 버전입니다. 복잡한 이론 없이 “왜 이 패턴을 쓰는지”까지 한 줄 설명을 붙였습니다.

---

## 0) 세 줄 요약

1. **날짜 경계**는 `>= 시작, < 종료` (타임존/자정 문제 예방)
2. **조인**은 **PK ← FK** 방향 + 중복 확인, `LEFT JOIN + COALESCE`로 누락값 처리
3. **집계**는 필요한 컬럼만, 피벗은 `SUM(CASE WHEN)`으로 간단히

---

## 1) 최소 문법 1페이지

```sql
-- 뼈대
SELECT <보고 싶은 컬럼들>
FROM   <테이블>
WHERE  <필터 조건>
GROUP BY <집계 기준>
ORDER BY <정렬 기준>
LIMIT  <최대 행 수>;
```

예시 테이블(가정):

* `users(id, joined_at, city)`
* `orders(id, user_id, amount, method, ordered_at)`

---

## 2) 자주 쓰는 5가지 레시피

### 2-1) 일자별 주문 건수 (경계 안전)

```sql
SELECT DATE_TRUNC('day', ordered_at) AS d,
       COUNT(*) AS cnt
FROM orders
WHERE ordered_at >= TIMESTAMP '2025-07-01 00:00:00'
  AND ordered_at <  TIMESTAMP '2025-08-01 00:00:00'
GROUP BY 1
ORDER BY 1;
```

> 왜? `>=, <`로 하면 타임존/말일 23:59:59 누락 문제를 피할 수 있음.

### 2-2) 사용자별 총 결제액 (조인 + NULL 처리)

```sql
SELECT u.id,
       COALESCE(SUM(o.amount), 0) AS total_amount
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```

> 왜? 주문이 없는 사용자도 **보이게** 하려면 `LEFT JOIN`, NULL은 `0`으로.

### 2-3) 사용자별 최신 이벤트만 남기기 (중복 제거)

```sql
WITH r AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY updated_at DESC) AS rn
  FROM user_events
)
SELECT *
FROM r
WHERE rn = 1;
```

> 왜? `DISTINCT`로 덮지 말고 \*\*원인(중복)\*\*을 통제: 키별 1순위만 남김.

### 2-4) 결제수단별 매출(간이 피벗)

```sql
SELECT DATE_TRUNC('month', ordered_at) AS ym,
       SUM(CASE WHEN method = 'card' THEN amount ELSE 0 END) AS amt_card,
       SUM(CASE WHEN method = 'cash' THEN amount ELSE 0 END) AS amt_cash
FROM orders
WHERE ordered_at >= TIMESTAMP '2025-01-01 00:00:00'
  AND ordered_at <  TIMESTAMP '2026-01-01 00:00:00'
GROUP BY 1
ORDER BY 1;
```

> 왜? BI 도구 없이도 “월×결제수단” 교차집계를 빠르게.

### 2-5) 카테고리별 상위 3개 상품 (Top-N)

```sql
SELECT *
FROM (
  SELECT p.*,
         ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rk
  FROM products p
) t
WHERE rk <= 3;
```

> 왜? 그룹 내부에서 **상위 k개**만 뽑는 전형 패턴.

---

## 3) 작업 순서 체크리스트 (진짜 최소)

* [ ] 문제를 한 줄로 정의 (예: “월별 결제수단 매출 추이”)
* [ ] 필요한 컬럼만 나열 (예: `ordered_at, method, amount`)
* [ ] `>= 시작, < 종료`로 기간 범위 설정
* [ ] 조인 시 PK←FK, 결과 **건수**와 **중복** 점검
* [ ] 샘플 기대값 3행을 메모하고 결과와 비교
* [ ] `EXPLAIN`으로 풀스캔/조인순서만 확인해 캡처 저장

---

## 4) 안티패턴 5줄 요약

* `SELECT *` 남발 금지 (불필요 스캔/카디널리티 증가)
* `DISTINCT`로 중복을 가리기 금지 (원인 해결 X)
* 문자열로 날짜 비교 금지 (형 변환 이슈)
* `JOIN ... ON`과 `WHERE` 조건 혼용으로 로직 바뀌지 않게 분리
* 타임존 무시 금지 (일자 경계 깨짐)

---

## 5) 내 도메인 예시 (템플릿)

**\[주택] 시군구별 월별 거래량 10행 미리보기**

```sql
-- 테이블 가정: transactions(sigungu, occurred_at, price, ...)
WITH base AS (
  SELECT sigungu,
         occurred_at -- TIMESTAMP(또는 DATETIME)
  FROM transactions
  WHERE occurred_at >= TIMESTAMP '2024-01-01 00:00:00'
    AND occurred_at <  TIMESTAMP '2025-01-01 00:00:00'
), monthly AS (
  SELECT sigungu,
         DATE_TRUNC('month', occurred_at) AS ym,
         COUNT(*) AS trades
  FROM base
  GROUP BY 1, 2
)
SELECT *
FROM monthly
ORDER BY ym, sigungu
LIMIT 10;
```

> 왜? 월별 경계 안전 + 시군구 단위 KPI를 바로 확인.

---

## 6) 저장소 구조 가이드 (추천)

```
.
├─ playbook/
│  └─ SQL_플레이북_입문.md
├─ queries/
│  ├─ mobility_subway_congestion.sql      -- (나중에 추가)
│  └─ card_basket_value.sql               -- (나중에 추가)
└─ docs/
   └─ explain_before_after.md             -- EXPLAIN 전/후 표 템플릿
```

---

## 7) EXPLAIN 전/후 표 템플릿

| 변경     | 실행시간(ms) | 스캔 행수 | 주요 변화 |
| ------ | -------: | ----: | ----- |
| Before |          |       |       |
| After  |          |       |       |

> 쿼리 링크와 플랜 스냅샷을 함께 남기면 **포트폴리오 설득력**이 급상승합니다.
