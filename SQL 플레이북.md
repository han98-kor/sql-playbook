# SQL 플레이북

> **TL;DR**
>
> 내가 자주 쓰는 조합: **CTE + 윈도우 + 조건부 집계 + 서브쿼리 최소화**
>
> **대상 DB:** PostgreSQL, MySQL, Presto/Athena (필요 시 차이 노트 참고)

---

## 1) 스타일 가이드

* 컬럼/테이블 **snake\_case**, 의미 있는 alias 사용 *(u, o 금지 → users, orders)*
* `SELECT *` 지양, **필요한 컬럼만 선택**
* **NULL 처리 원칙:** `COALESCE`, `LEFT JOIN` 시 기본값 명시

---

## 2) 핵심 패턴 스니펫

### 2-1. 최신 레코드만 뽑기(키별 dedupe)

```sql
WITH ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY updated_at DESC) AS rn
  FROM user_events
)
SELECT *
FROM ranked
WHERE rn = 1;
```

### 2-2. Top-N(그룹 내 상위 k개)

```sql
SELECT *
FROM (
  SELECT p.*,
         ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rk
  FROM products p
) t
WHERE rk <= 3;
```

### 2-3. 세션화(30분 이탈 기준)

```sql
WITH seq AS (
  SELECT user_id, ts,
         CASE WHEN ts - LAG(ts) OVER (PARTITION BY user_id ORDER BY ts) > INTERVAL '30 minutes'
              THEN 1 ELSE 0 END AS is_new
  FROM pageviews
),
sessioned AS (
  SELECT user_id, ts,
         SUM(is_new) OVER (PARTITION BY user_id ORDER BY ts ROWS UNBOUNDED PRECEDING) AS session_id
  FROM seq
)
SELECT *
FROM sessioned;
```

### 2-4. 날짜/시간대 안전 집계(경계 포함/제외)

```sql
SELECT DATE_TRUNC('day', occurred_at) AS d,
       COUNT(*) AS cnt
FROM events
WHERE occurred_at >= TIMESTAMP '2025-07-01 00:00:00'
  AND occurred_at <  TIMESTAMP '2025-08-01 00:00:00'
GROUP BY 1
ORDER BY 1;
```

### 2-5. 조건부 집계(피벗 대체)

```sql
SELECT store_id,
       SUM(CASE WHEN method = 'card' THEN amount ELSE 0 END) AS amt_card,
       SUM(CASE WHEN method = 'cash' THEN amount ELSE 0 END) AS amt_cash
FROM payments
GROUP BY store_id;
```

---

## 3) 조인 가이드

* **기본키 → 외래키** 방향으로 조인, **카디널리티 확인**(중복 폭발 방지)
* **범위 조건은 `JOIN ... ON`**, \*\*필터는 `WHERE`\*\*로 분리해 SARGability 확보
* 스케치: **소테이블 → 대테이블** 순으로 단계적 조인

---

## 4) 성능/튜닝 메모

* 인덱스 후보: **WHERE/JOIN**에 쓰는 컬럼 + **선택도 높은 컬럼**
* **범위 조건 + 정렬**이 함께 있으면 복합 인덱스(선두열=필터, 다음=정렬)
* **EXPLAIN 체크리스트:** 풀 스캔? 조인 순서? 해시/네스티드루프? 필터 푸시다운?

**EXPLAIN 기록 예시(요약)**

```
QUERY PLAN
Hash Join  (cost=…)
  -> Seq Scan on payments …
  -> Index Scan using idx_store on stores …
Filter: (occurred_at >= … AND occurred_at < …)
```

* *변경 전/후* 실행시간, 읽은 로우 수 **비교 표**로 남기기

---

## 5) 안티패턴

* `DISTINCT`로 중복을 **덮어 숨김** *(원인 = 잘못된 조인)*
* 타임존 **미고려**(일자 경계 깨짐)
* 조건을 `ON`/`WHERE`에 혼재해 **로직이 바뀌는 실수**

---

## 6) DB별 차이 노트(옵션)

* **Postgres** `DATE_TRUNC`, **MySQL** `DATE_FORMAT`, **Presto** `date_trunc` 차이
* 문자열 → 날짜 **캐스팅 규칙** 정리

---

## 7) 실전 스니펫(내 프로젝트)

* **\[모빌리티] 지하철 혼잡도 집계 쿼리:** 원천 스키마, 변환 CTE, 최종 KPI, 샘플 결과 10행
* **\[카드/소비] 업종별 객단가 분석 쿼리:** 지표 정의서(분자/분모), 누락 처리, NA 전략

---

## 8) 쿼리 검수 체크리스트

* [ ] 비즈니스 정의와 **지표 수식**이 일치
* [ ] **NULL/중복/경계(≥, <)** 처리
* [ ] **카디널리티** 예상 vs 실제 비교
* [ ] **타임존/일자 경계** 확인
* [ ] **인덱스/파티션** 활용 여부, **EXPLAIN** 첨부
* [ ] 결과 샘플과 **기대값 크로스체크**(테스트 케이스)

---

## 9) 인터뷰/코테 단답 레시피

* 윈도우함수로 **중복제거**, **세션화** 로직 설명, *“GROUP BY vs 윈도우”* 차이
* 느린 쿼리 **진단 절차**(플랜 → 핫스팟 → 인덱스/리라이트)

---

## 10) 참고/레퍼런스(개인 링크)

* 내 깃허브 `queries/` 폴더, Tableau 대시보드 URL, 프로젝트 README

---

### 운영 팁

* **폴더화:** `queries/`에 문제별 SQL 파일 보관, *플레이북에는 개요·핵심만.*
* **Before/After:** 성능 개선 전후 수치(시간/스캔로우/플랜 변화) **표로 기록**.
