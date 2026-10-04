# DuckDB + SQLite + Parquet 실전 팁 정리

대용량 CSV를 다루면서 겪은 `DB Browser for SQLite`의 한계와 `DuckDB`로 해결한 방법 총정리.

---

## 1. CSV Import - 전부 TEXT로 읽기

### 1-1. DB Browser for SQLite에서 강제 TEXT화
DB Browser는 CSV import시 타입을 자동 추론함. 주소코드 앞자리 0이 날아가는 문제 해결법.

**정석: 테이블을 먼저 만들고 Import**

```sql
-- 1. SQL 실행 탭에서 테이블 먼저 생성
DROP TABLE IF EXISTS 사업장;
CREATE TABLE 사업장 (
    자료생성년월 TEXT,
    사업장명 TEXT,
    사업자등록번호 TEXT,
    고객법정동주소코드 TEXT, -- TEXT로 고정
    고객행정동주소코드 TEXT,
    가입자수 TEXT,
    당월고지금액 TEXT
);
```

파일 > 가져오기 > CSV 파일로부터 테이블
* 테이블 이름: `사업장` (방금 만든 이름)
* `새 테이블 생성` 체크 해제
* `기존 테이블에 데이터 추가` 선택

### 1-2. DuckDB로 전부 TEXT로 읽기 (추천)
`all_varchar=true`가 핵심.

```python
import duckdb
duckdb.sql("""
    SELECT * FROM read_csv('원본.csv', 
        all_varchar=true, 
        header=true, 
        encoding='utf-8'
    ) LIMIT 5
""").show()
```

---

## 2. CSV -> Parquet 변환 (메모리 안 터지게)

### 2-1. 기본 변환

```python
import duckdb

duckdb.sql("""
    COPY (
        SELECT * FROM read_csv('원본.csv', all_varchar=true, header=true)
    ) TO '사업장.parquet' (FORMAT PARQUET, COMPRESSION ZSTD, ROW_GROUP_SIZE 100000)
""")
```

### 2-2. 측정열만 INT로 바꾸면서 변환
주소코드는 TEXT 유지, 측정열만 BIGINT 변환.

```python
import duckdb

duckdb.sql("""
    COPY (
        SELECT
            * EXCLUDE (가입자수, 당월고지금액, 신규취득자수, 상실가입자수),
            -- 4.0 같은 소수점 대비해서 DOUBLE 거쳤다가 BIGINT
            TRY_CAST(CAST(가입자수 AS DOUBLE) AS BIGINT) AS 가입자수,
            TRY_CAST(CAST(당월고지금액 AS DOUBLE) AS BIGINT) AS 당월고지금액,
            TRY_CAST(신규취득자수 AS BIGINT) AS 신규취득자수,
            TRY_CAST(상실가입자수 AS BIGINT) AS 상실가입자수
        FROM read_csv('원본.csv', all_varchar=true, header=true)
    ) TO '사업장.parquet' (FORMAT PARQUET, COMPRESSION ZSTD)
""")
```

> `CAST`는 실패시 에러, `TRY_CAST`는 실패시 NULL로 넘어감. 빈칸 있는 파일은 무조건 `TRY_CAST`.

---

## 3. SQLite .db <-> Parquet 변환

### 3-1. SQLite .db -> Parquet
DB Browser로 만든 .db 파일을 DuckDB가 직접 읽음.

```python
import duckdb

duckdb.sql("""
    COPY (
        SELECT * FROM sqlite_scan('내DB.db', '사업장')
    ) TO '사업장.parquet' (FORMAT PARQUET)
""")
```

### 3-2. Parquet + SQLite .db 동시 생성 (최적 루트)
CSV -> Parquet, DB 둘 다 한 번에 뽑기.

```python
import duckdb
con = duckdb.connect()

con.sql("""
    CREATE TEMP VIEW 정리된데이터 AS
    SELECT
        * EXCLUDE (가입자수, 당월고지금액),
        TRY_CAST(CAST(가입자수 AS DOUBLE) AS BIGINT) AS 가입자수,
        TRY_CAST(CAST(당월고지금액 AS DOUBLE) AS BIGINT) AS 당월고지금액
    FROM read_csv('원본.csv', all_varchar=true, header=true);

    -- Parquet 저장
    COPY (SELECT * FROM 정리된데이터) TO '사업장.parquet' (FORMAT PARQUET, COMPRESSION ZSTD);

    -- SQLite DB 저장
    ATTACH '사업장.db' AS db (TYPE SQLITE);
    DROP TABLE IF EXISTS db.사업장;
    CREATE TABLE db.사업장 AS SELECT * FROM 정리된데이터;
""")
```

---

## 4. 대용량 Parquet 읽기 (pandas 메모리 터짐 방지)

`pd.read_parquet()`은 전체를 RAM에 올려서 터짐. DuckDB로 필요한 것만 잘라서 가져오기.

```python
import duckdb

# 필요한 컬럼, 필요한 행만 필터링해서 pandas로
df = duckdb.sql("""
    SELECT 고객법정동주소코드, 가입자수, 당월고지금액
    FROM '사업장.parquet'
    WHERE 자료생성년월 = '2026-08' AND 가입자수 > 10
    LIMIT 100000
""").df()

# 집계는 RAM 안 쓰고 DuckDB에서 바로
duckdb.sql("SELECT COUNT(*), SUM(당월고지금액) FROM '사업장.parquet'").show()
```

### 다른 대안
```python
# PyArrow로 컬럼 일부만 읽기
import pyarrow.parquet as pq
table = pq.read_table('사업장.parquet', columns=['고객법정동주소코드', '가입자수'])
df = table.to_pandas()

# Polars Lazy - 50GB 이상일 때
import polars as pl
df = pl.scan_parquet('사업장.parquet').filter(
    pl.col('자료생성년월') == '2026-08'
).select(['고객법정동주소코드', '가입자수']).collect()
```

---

## 5. Parquet 정보 보기 (pandas info() / describe() 대체)

```python
import duckdb

# 1. 컬럼명, 타입 확인 = df.info()
duckdb.sql("DESCRIBE SELECT * FROM '사업장.parquet'").show()

# 2. 통계 + NULL% + 유니크수 = info() + describe() 합친거
duckdb.sql("SUMMARIZE SELECT * FROM '사업장.parquet'").show()

# 3. 숫자 컬럼만 통계
duckdb.sql("SUMMARIZE SELECT 가입자수, 당월고지금액 FROM '사업장.parquet'").show()

# 4. 전체 로우 수 (메타데이터만 읽어서 빠름)
duckdb.sql("SELECT COUNT(*) as total_rows FROM '사업장.parquet'").show()

# 5. Parquet 파일 자체 메타데이터 (pandas엔 없는 기능)
duckdb.sql("SELECT * FROM parquet_metadata('사업장.parquet')").show()
duckdb.sql("SELECT * FROM parquet_schema('사업장.parquet')").show()
```

---

## 6. Parquet 특정 필드 타입 변경

Parquet은 수정 불가. 읽으면서 CAST해서 새 파일로 덮어쓰기.

```python
import duckdb

# 특정 필드 1개만 INT로 변경해서 덮어쓰기
duckdb.sql("""
    COPY (
        SELECT
            * EXCLUDE (당월고지금액),
            TRY_CAST(당월고지금액 AS BIGINT) AS 당월고지금액
        FROM '사업장.parquet'
    ) TO '사업장.parquet' 
    (FORMAT PARQUET, COMPRESSION ZSTD, OVERWRITE_OR_IGNORE true)
""")

# 확인
duckdb.sql("DESCRIBE SELECT * FROM '사업장.parquet'").show()
```

---

## 7. Parquet 2개 JOIN

```python
import duckdb

# 파일 자체를 테이블처럼 JOIN
duckdb.sql("""
    SELECT a.사업장명, a.가입자수, b.법정동명
    FROM '사업장.parquet' AS a
    LEFT JOIN '법정동코드.parquet' AS b
      ON a.고객법정동주소코드 = b.법정동코드
    LIMIT 10
""").show()

# JOIN 결과를 다시 Parquet으로 저장
duckdb.sql("""
    COPY (
        SELECT a.*, b.시도명, b.시군구명
        FROM '사업장.parquet' a
        JOIN '법정동코드.parquet' b
          ON a.고객법정동주소코드 = b.법정동코드
    ) TO '사업장_주소붙인.parquet' (FORMAT PARQUET)
""")

# parquet + csv + sqlite 섞어서 JOIN도 가능
duckdb.sql("""
    SELECT *
    FROM '사업장.parquet' a
    JOIN '업종코드.parquet' b ON a.사업장업종코드 = b.업종코드
    JOIN read_csv('시군구.csv', all_varchar=true) c ON a.시군구코드 = c.code
""")
```

---

## 8. JOIN할 때 필드명 기억 안날 때 스키마 확인 도구

### 8-1. 스키마 + 샘플 빠르게 보기
```python
import duckdb
con = duckdb.connect()

files = ['사업장.parquet', '법정동코드.parquet']
for f in files:
    print(f"\n=== {f} ===")
    con.sql(f"DESCRIBE SELECT * FROM '{f}'").show()
    con.sql(f"SELECT * FROM '{f}' LIMIT 3").show()
```

### 8-2. 공통 컬럼 자동 찾기 (JOIN 키 추천)
```python
import duckdb
con = duckdb.connect()

def find_join_keys(file1, file2):
    cols1 = [r[0] for r in con.sql(f"DESCRIBE SELECT * FROM '{file1}'").fetchall()]
    cols2 = [r[0] for r in con.sql(f"DESCRIBE SELECT * FROM '{file2}'").fetchall()]

    print(f"\n[{file1}] 컬럼: {cols1}")
    print(f"[{file2}] 컬럼: {cols2}")

    common = set(cols1) & set(cols2)
    if common:
        print(f"\n✅ 공통 컬럼 (JOIN 후보): {list(common)}")
    else:
        print("\n❌ 공통 컬럼명 없음. 유사한 이름 찾는중...")
        for c1 in cols1:
            for c2 in cols2:
                if c1 in c2 or c2 in c1:
                    print(f" 유사 후보: {c1} <-> {c2}")

find_join_keys('사업장.parquet', '법정동코드.parquet')
```

### 8-3. 값이 실제로 겹치는지 검증
이름만 비슷하고 값이 다르면 JOIN 실패. 겹치는 값 개수로 검증.

```python
duckdb.sql("""
    SELECT
        (SELECT COUNT(DISTINCT 고객법정동주소코드) FROM '사업장.parquet') as 사업장_코드수,
        (SELECT COUNT(DISTINCT 법정동코드) FROM '법정동코드.parquet') as 법정동_코드수,
        (SELECT COUNT(*) FROM
            (SELECT DISTINCT 고객법정동주소코드 as code FROM '사업장.parquet'
             INTERSECT
             SELECT DISTINCT 법정동코드 as code FROM '법정동코드.parquet')
        ) as 겹치는_코드수
""").show()
```

