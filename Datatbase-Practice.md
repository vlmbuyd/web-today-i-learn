# Database 실습 정답

## DDL 실습

### 문제 1: 테이블 생성하기 (CREATE TABLE)

**생각해보기: 중복된 컬럼은?**

`crew_id`와 `nickname`은 항상 같은 쌍으로 반복된다. 즉, `crew_id`가 결정되면 `nickname`도 자동으로 결정되는 함수적 종속 관계이므로, 동일한 크루가 출석할 때마다 `nickname`이 중복 저장된다.

**crew 테이블 구성**

`crew_id`(PK)와 `nickname` 두 컬럼으로 구성한다. 출석 기록에서 닉네임을 참조할 때 `crew_id`를 기준으로 crew 테이블에서 조회하면 된다.

```sql
-- 크루 목록 조회 (중복 제거)
SELECT DISTINCT crew_id, nickname
FROM attendance
ORDER BY crew_id;
```

```sql
-- crew 테이블 생성
CREATE TABLE crew (
  crew_id INT NOT NULL,
  nickname VARCHAR(50) NOT NULL,
  PRIMARY KEY (crew_id)
);
```

```sql
-- attendance 테이블에서 크루 정보 추출하여 삽입
INSERT INTO crew (crew_id, nickname)
SELECT DISTINCT crew_id, nickname
FROM attendance
ORDER BY crew_id;
```

---

### 문제 2: 테이블 컬럼 삭제하기 (ALTER TABLE)

**생각해보기: 불필요해지는 컬럼은?**

crew 테이블이 생성되어 `crew_id`로 `nickname`을 조회할 수 있으므로, attendance 테이블의 `nickname` 컬럼이 불필요해진다.

```sql
ALTER TABLE attendance DROP COLUMN nickname;
```

---

### 문제 3: 외래키 설정하기

**생각해보기**

외래키 제약이 없으면 crew 테이블에 존재하지 않는 `crew_id`가 attendance에 삽입될 수 있어 데이터 무결성이 깨진다. 외래키를 설정하면 참조 무결성이 보장된다.

```sql
ALTER TABLE attendance
  ADD CONSTRAINT fk_attendance_crew
  FOREIGN KEY (crew_id) REFERENCES crew(crew_id);
```

---

### 문제 4: 유니크 키 설정

**생각해보기**

현재 crew 테이블에는 nickname 컬럼에 UNIQUE 제약이 없으므로, 동일한 닉네임을 가진 크루가 여러 명 등록될 수 있다. UNIQUE 제약을 추가해 닉네임 중복을 방지한다.

```sql
ALTER TABLE crew
  ADD CONSTRAINT uq_crew_nickname UNIQUE (nickname);
```

---

## DML(CRUD) 실습

### 문제 5: 크루 닉네임 검색하기 (LIKE)

3월 4일에 출석한 크루 중 닉네임 첫 글자가 '디'인 크루를 찾는다.

```sql
SELECT c.nickname, a.attendance_date, a.start_time
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
WHERE a.attendance_date = '2025-03-04'
  AND c.nickname LIKE '디%';
```

> 결과: **디노** (09:59 등교)

---

### 문제 6: 출석 기록 확인하기 (SELECT + WHERE)

어셔의 3월 6일 출석 기록이 실제로 누락됐는지 확인한다.

```sql
SELECT a.*
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
WHERE c.nickname = '어셔'
  AND a.attendance_date = '2025-03-06';
```

> 결과가 없으면 기록이 누락된 것이다.

---

### 문제 7: 누락된 출석 기록 추가 (INSERT)

어셔가 crew 테이블에 없다면 먼저 등록하고, 이후 출석 기록을 추가한다.

```sql
-- crew 테이블에 어셔 추가 (아직 등록되지 않은 경우)
INSERT INTO crew (nickname) VALUES ('어셔');

-- 출석 기록 추가
INSERT INTO attendance (crew_id, attendance_date, start_time, end_time)
VALUES (
  (SELECT crew_id FROM crew WHERE nickname = '어셔'),
  '2025-03-06',
  '09:31',
  '18:01'
);
```

---

### 문제 8: 잘못된 출석 기록 수정 (UPDATE)

주니의 3월 12일 등교 시각을 10:05에서 10:00으로 수정한다.

```sql
UPDATE attendance a
JOIN crew c ON a.crew_id = c.crew_id
SET a.start_time = '10:00'
WHERE c.nickname = '주니'
  AND a.attendance_date = '2025-03-12';
```

---

### 문제 9: 허위 출석 기록 삭제 (DELETE)

아론의 3월 12일 허위 출석 기록을 삭제한다.

```sql
DELETE a FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
WHERE c.nickname = '아론'
  AND a.attendance_date = '2025-03-12';
```

---

### 문제 10: 출석 정보 조회하기 (JOIN)

`crew_id`를 기준으로 crew 테이블에서 `nickname`을 가져와 함께 조회한다.

```sql
SELECT c.nickname, a.attendance_date, a.start_time, a.end_time
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
ORDER BY a.attendance_date, c.crew_id;
```

---

### 문제 11: nickname으로 쿼리 처리하기 (서브 쿼리)

닉네임을 입력하면 해당 크루의 출석 기록을 조회하는 서브 쿼리를 사용한다.

```sql
SELECT *
FROM attendance
WHERE crew_id = (
  SELECT crew_id FROM crew WHERE nickname = '검프'
);
```

---

### 문제 12: 가장 늦게 하교한 크루 찾기

3월 5일에 가장 늦게 하교한 크루의 닉네임과 하교 시각을 조회한다.

```sql
SELECT c.nickname, a.end_time
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
WHERE a.attendance_date = '2025-03-05'
ORDER BY a.end_time DESC
LIMIT 1;
```

> 결과: **네오** (18:15 하교)

---

## 집계 함수 실습

### 문제 13: 크루별로 '기록된' 날짜 수 조회

```sql
SELECT c.nickname, COUNT(*) AS attendance_count
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
GROUP BY a.crew_id, c.nickname
ORDER BY a.crew_id;
```

---

### 문제 14: 크루별로 등교 기록이 있는(start_time IS NOT NULL) 날짜 수 조회

`COUNT(컬럼명)`은 NULL을 자동으로 제외하고 집계한다.

```sql
SELECT c.nickname, COUNT(a.start_time) AS checkin_count
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
GROUP BY a.crew_id, c.nickname
ORDER BY a.crew_id;
```

---

### 문제 15: 날짜별로 등교한 크루 수 조회

```sql
SELECT attendance_date, COUNT(*) AS crew_count
FROM attendance
GROUP BY attendance_date
ORDER BY attendance_date;
```

---

### 문제 16: 크루별 가장 빠른 등교 시각(MIN)과 가장 늦은 등교 시각(MAX)

```sql
SELECT c.nickname,
       MIN(a.start_time) AS earliest_checkin,
       MAX(a.start_time) AS latest_checkin
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
GROUP BY a.crew_id, c.nickname
ORDER BY a.crew_id;
```
