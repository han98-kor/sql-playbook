# SQL 플레이북
현업에서 자주 쓰는 SQL 패턴과 성능 튜닝 메모를 정리했습니다.  
역할: 데이터 전처리/분석/시각화, 지표 정의 및 검증.

## 어떻게 읽나
- `/playbook/SQL_플레이북.md` 핵심 패턴 요약
- `/queries/` 실전 쿼리 (설명 + EXPLAIN 전/후 비교)
- 대상 DB: PostgreSQL, MySQL, Presto/Athena

## 폴더 구조
.
├─ [SQL 플레이북 (입문용)](./playbook/SQL_플레이북_입문.md)
├─ playbook/SQL_플레이북.md
├─ queries/
│  ├─ mobility_subway_congestion.sql
│  ├─ card_basket_value.sql
│  └─ common_snippets.sql
└─ docs/
   └─ explain_before_after.md

## 하이라이트
- 윈도우/세션화/조건부 집계/범위집계 경계처리 레시피
- EXPLAIN 체크리스트 + 성능 개선 전/후 표

## 연락
- 이메일: gkswl0611@gmail.com
