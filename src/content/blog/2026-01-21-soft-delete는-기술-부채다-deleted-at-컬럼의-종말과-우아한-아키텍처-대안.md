---
title: "Soft Delete는 기술 부채다: `deleted_at` 컬럼의 종말과 우아한 아키텍처 대안"
description: "Soft Delete는 기술 부채다: `deleted_at` 컬럼의 종말과 우아한 아키텍처 대안"
pubDate: "2026-01-21T07:58:10Z"
---

엔터프라이즈 시스템을 설계할 때 "데이터 삭제" 요구사항은 필연적으로 등장합니다. 대부분의 엔지니어는 관성적으로 `deleted_at` 타임스탬프나 `is_deleted` 불리언 컬럼을 추가하는 소위 'Soft Delete' 패턴을 적용합니다.

하지만 이 "단순한" 결정은 시간이 지남에 따라 쿼리 성능 저하, 인덱스 비효율성, 마이그레이션의 악몽, 그리고 운영 복잡성이라는 기술 부채로 돌아옵니다. 본 아티클에서는 Atlas9의 기술 리더가 분석한 Soft Delete의 구조적 문제점을 해부하고, PostgreSQL 기반의 **Trigger Archiving** 및 **WAL-based CDC** 패턴을 통해 이를 해결하는 엔지니어링 방법론을 심층 분석합니다.

---

## 1. `deleted_at` 패턴이 실패하는 지점

초기 개발 단계에서 `deleted_at` 컬럼은 매력적입니다. 데이터를 복구하기 쉽고, 감사(Audit) 로그 역할을 겸하는 것처럼 보이기 때문입니다. 하지만 시스템이 성장하면 다음과 같은 치명적인 문제들이 발생합니다.

### A. 쿼리와 인덱스의 오염 (Pollution)
*   **Predicate 누락 위험:** 모든 `SELECT` 쿼리에 `WHERE deleted_at IS NULL`을 강제해야 합니다. 개발자가 실수로 이를 누락하면 삭제된 데이터가 라이브 서비스에 노출됩니다.
*   **인덱스 효율성 저하:** 인덱스는 삭제된 행(Dead Rows)까지 포함하여 비대해집니다. Partial Index(`WHERE deleted_at IS NULL`)를 사용할 수 있지만, 이는 모든 인덱스 정의를 복잡하게 만듭니다.
*   **Unique Constraint 충돌:** 사용자가 계정을 삭제하고 동일한 이메일로 재가입하려 할 때, `deleted_at`이 설정된 기존 레코드가 Unique Index와 충돌합니다. 이를 해결하려면 인덱스에 `deleted_at`을 포함시켜야 하는 등 스키마가 지저분해집니다.

### B. 마이그레이션과 데이터 정합성
*   **Schema Drift:** 2년 전 삭제된 레코드는 현재의 데이터 검증 로직(Validation Rules)이나 스키마 제약조건을 만족하지 못할 가능성이 높습니다. 새로운 컬럼을 추가하거나 타입을 변경하는 마이그레이션 시, 삭제된 데이터가 발목을 잡습니다.
*   **복구의 복잡성:** 단순히 `SET deleted_at = NULL`로 복구가 끝나는 경우는 드뭅니다. 연관된 외부 시스템(결제, 서드파티 연동 등)의 상태까지 복구해야 하므로, 결국 별도의 복잡한 복구 로직이 필요해집니다.

---

## 2. 대안 아키텍처 1: Trigger 기반의 격리 저장소 (The Pragmatic Choice)

가장 현실적이고 강력한 대안은 **라이브 테이블에서 데이터를 즉시 삭제하고, 별도의 아카이브 테이블로 이관**하는 것입니다. 이 방식은 라이브 테이블을 가볍게 유지하면서도 데이터 보존 요구사항을 충족합니다.

### 아키텍처 구현 상세
PostgreSQL의 Trigger를 사용하여 삭제 직전의 데이터를 JSONB 형태로 아카이브 테이블에 저장합니다.

**1. 아카이브 테이블 스키마 설계**
스키마 변경에 유연하게 대응하기 위해 원본 데이터를 `JSONB`로 저장합니다.

```sql
CREATE TABLE archive (
    id UUID PRIMARY KEY,
    table_name TEXT NOT NULL,
    record_id TEXT NOT NULL,
    data JSONB NOT NULL,
    archived_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    caused_by_table TEXT,
    caused_by_id TEXT
);

CREATE INDEX idx_archive_table_record ON archive(table_name, record_id);
CREATE INDEX idx_archive_archived_at ON archive(archived_at);
```

**2. 아카이빙 트리거 함수 (JSON 변환)**
`OLD` 레코드를 통째로 JSON으로 변환하여 저장합니다.

```sql
CREATE OR REPLACE FUNCTION archive_on_delete()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO archive (id, table_name, record_id, data)
    VALUES (
        gen_random_uuid(),
        TG_TABLE_NAME,
        OLD.id::TEXT,
        to_jsonb(OLD)
    );
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;
```

**3. Cascade Delete 추적 (심화 기법)**
부모 레코드가 삭제될 때 자식 레코드가 `ON DELETE CASCADE`로 함께 삭제되는 경우, 자식 레코드의 삭제 원인을 추적하는 것이 중요합니다. PostgreSQL의 `current_setting` 세션 변수를 활용하여 컨텍스트를 전파할 수 있습니다.

```sql
CREATE OR REPLACE FUNCTION archive_on_delete()
RETURNS TRIGGER AS $$
DECLARE
    cause_table TEXT;
    cause_id TEXT;
BEGIN
    -- Cascade 컨텍스트 확인
    cause_table := current_setting('archive.cause_table', true);
    cause_id := current_setting('archive.cause_id', true);

    -- 최상위 삭제인 경우, 현재 레코드를 원인으로 설정
    IF cause_table IS NULL THEN
        PERFORM set_config('archive.cause_table', TG_TABLE_NAME, true);
        PERFORM set_config('archive.cause_id', OLD.id::TEXT, true);
        cause_table := TG_TABLE_NAME;
        cause_id := OLD.id::TEXT;
    END IF;

    INSERT INTO archive (id, table_name, record_id, data, caused_by_table, caused_by_id)
    VALUES (
        gen_random_uuid(),
        TG_TABLE_NAME,
        OLD.id::TEXT,
        to_jsonb(OLD),
        cause_table,
        cause_id
    );
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;
```

### 장점
*   **Zero Query Overhead:** 라이브 쿼리에서 `WHERE deleted_at IS NULL`을 제거할 수 있습니다.
*   **인덱스 최적화:** 라이브 테이블 인덱스가 작고 효율적으로 유지됩니다.
*   **수명 주기 관리:** 아카이브 테이블은 시간 기반 파티셔닝(Partitioning)을 적용하여 오래된 데이터를 `DROP PARTITION`으로 저비용 삭제할 수 있습니다.

---

## 3. 대안 아키텍처 2: WAL 기반 CDC (The Scalable Choice)

애플리케이션이나 DB 트리거에 의존하지 않고, 데이터베이스의 트랜잭션 로그(WAL)를 직접 구독하여 삭제 이벤트를 캡처하는 방식입니다.

### 작동 원리
PostgreSQL → Debezium (Kafka Connect) → Kafka → Consumer → Archive Storage (S3/Elasticsearch)

### 핵심 고려사항: `max_slot_wal_keep_size`
CDC 방식의 가장 큰 리스크는 **Replication Slot의 지연**입니다. 컨슈머가 장애로 멈추면, PostgreSQL은 복제를 위해 WAL 파일을 삭제하지 않고 계속 쌓아둡니다. 이는 디스크 풀(Disk Full)로 인한 **Primary DB 장애**로 이어질 수 있습니다.

PostgreSQL 13+에서는 이를 방지하기 위해 안전장치를 설정해야 합니다:
```sql
ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';
```
이 설정은 슬롯이 너무 뒤처지면 해당 슬롯을 무효화(Invalidate)하여 Primary DB의 안정성을 확보합니다(대신 CDC 데이터 유실 발생).

---

## 4. Community Insights: 현장의 목소리와 반론

Hacker News의 시니어 엔지니어들은 원문의 주장에 대해 다음과 같은 심도 있는 통찰을 추가했습니다.

**1. 금융권의 시각: Audit vs Soft Delete**
*   금융/은행 도메인에서는 데이터의 완전한 이력 추적이 필수입니다. 하지만 Soft Delete(`UPDATE`)만으로는 부족하며, **Audit Table**이나 **Temporal Table**(`valid_from`, `valid_to`) 패턴이 더 적합합니다. 단순히 "삭제된 상태"가 아니라 "누가, 언제, 무엇을" 변경했는지 알아야 하기 때문입니다.

**2. 성능 문제의 현실적 해결책: 파티셔닝(Partitioning)**
*   Soft Delete의 성능 저하(50~70%가 삭제된 행일 때)는 **Table Partitioning**으로 해결할 수 있습니다. `deleted_at`을 기준으로 파티션을 나누거나, 삭제된 데이터를 별도 파티션으로 이동시키면 인덱스 스캔 효율을 유지할 수 있습니다.

**3. Product vs Implementation**
*   "Soft Delete"는 구현 세부 사항입니다. 제품 관점에서는 "삭제(Delete)", "보관(Archive)", "숨김(Hide)"으로 기능을 명확히 구분해야 합니다. 사용자가 "삭제"를 원할 때, 이를 법적 규제 준수(GDPR)를 위한 물리적 삭제(Hard Delete)로 처리할지, 실수 방지를 위한 논리적 삭제로 처리할지는 비즈니스 요구사항에 따라 결정되어야 합니다.

**4. Immutable Data Model**
*   일부 엔지니어들은 아예 `UPDATE`와 `DELETE`를 허용하지 않는 **Immutable Database** (예: Datomic) 접근 방식을 제안합니다. 모든 변경 사항을 새로운 버전의 레코드로 `INSERT`하여 시간 여행(Time Travel) 기능을 기본으로 제공받는 방식입니다.

---

## 5. 결론 (Verdict)

`deleted_at` 컬럼 추가는 초기에는 쉽지만, 장기적으로는 시스템의 암적인 존재가 됩니다. 현대적인 백엔드 시스템에서는 다음 기준에 따라 아키텍처를 선택하십시오.

1.  **대부분의 경우 (권장):** **Trigger 기반 아카이빙**을 사용하십시오. 라이브 테이블을 깨끗하게 유지하고, 쿼리 복잡성을 제거하며, 인프라 비용이 낮습니다.
2.  **대규모 MSA/이벤트 기반 시스템:** **CDC (Debezium)**를 도입하여 삭제 이벤트를 스트림으로 처리하고 데이터 웨어하우스나 S3로 영구 보관하십시오.
3.  **엄격한 감사(Audit) 필요 시:** Soft Delete에 의존하지 말고, 별도의 **Audit Log 테이블**이나 **SCD Type 2** (Slowly Changing Dimension) 패턴을 적용하십시오.

**"데이터는 자산이지만, 관리되지 않는 데이터는 부채입니다."**

---

## References

*   **Original Article:** [The challenges of soft delete](https://atlas9.dev/blog/soft-delete.html)
*   **Hacker News Thread:** [Discussion on 'The challenges of soft delete'](https://news.ycombinator.com/item?id=46698061)
*   **External Reference:** [PostgreSQL Documentation: Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
