---
layout: post
title: 4주차
---

## 박병준 : 387 ~ 453 (2부 1.4.6절)

## 부분범위처리 (Partial range scan)
부분범위처리란?
- WHERE 절에 주어진 조건을 만족하는 전체범위를 모두 처리하는 것이 아닌, 운반단위(Array size)까지만 먼저 처리하여 결과를 추출하는 것.
- 일부분만 처리하고서도 정확한 결과를 추출하기 위해서 옵티마이저의 특성을 먼저 이해해야 함.


### 1.1 부분범위 처리의 개념
- 처리범위가 넓다는 것은 데이터의 입장에서 보는것이고, 데이터를 운반하는 임장에서 보면 그 반대가 될 수 있음. 그림 2-1-1.
- 운반단위와 부분범위 처리 간에는 묘한 상관관계가 성립됨. 그림 2-1-2
- 옵티마이저가 생성한 실행계획에서 드라이빙 조건과 체크 조건이 어떤것이 되었느냐에 따라 처리가 달라짐. 그림 2-1-3
- 전체범위처리 vs 부분범위처리. 그림 2-1-4
- 전체범위처리
  - 드라이빙 조건을 만족하는 범위를 모두 스캔하여 체크조건으로 검증 한 후 성공한건에 대해 임시저장
  - 2차 가공
  - 운반단위만큼 추출
- 부분범위처리
  - 드라이빙 조건을 만족하는 범위를 차례로 스캔하면서 체크조건을 검증하여 성공한 건을 바로 운반단위로 보냄
  - 운반단위가 채워지면 추출
- 실행계획에서 전체범위처리를 나타내는 항목들 : SORT, VIEW, MERGE JOIN, HASH JOIN(X)
- 우리가 사용하는 툴이 부분범위 처리를 하는지 알 수 있는 가장 간단한 방법은 수만건 이상의 데이터를 가진 테이블을 WHERE 절 없이 조회해보는 것. 만약 결과가 즉시 추출되면 부분범위 처리를 하고 있는것이며, 한참 기다려야 한다면 전체범위처리 방식으로 수행되고 있는 것.


### 1.2 부분범위처리의 적용원칙
모든 경우의 처리를 부분범위 처리로 수행할 수는 없다. 부분범위 처리로 수행되기 위한 방법을 이해해야 함.

#### 1.2.1 부분범위 처리의 자격
부분범위 처리를 할 수 있는 자격 : 논리적으로 보았을 때 반드시 전체범위를 읽어서 가공을 해야하는 하는 경우를 제외한 모든 형태

1. 그룹함수

```sql
SELECT SUM(ordqty)
  FROM order
 WHERE ord_date LIKE '201705%'
 
SELECT ord_dept, COUNT(*)
  FROM order
 WHERE ord_date LIKE '201705%'
 GROUP BY ord_dept
```

2. ORDER BY

```sql
SELECT ord_date, ordqty * 1000
  FROM order
 WHERE ord_date LIKE '201705%'
 ORDER BY ord_date
```

- 인덱스와 ORDER BY 에 사용된 컬럼이 동일하다면 부분범위 처리가 가능.

```sql
SELECT * 
 FROM phm_employee
--ORDER BY emp_nm
--ORDER BY emp_id

Query plan1:
sscan
    class: phm_employee node[0]
    cost:  4114 card 14269

Query plan2:
temp(order by)
    subplan: sscan
                 class: phm_employee node[0]
                 cost:  4114 card 14269
    sort:  7 asc
    cost:  4301 card 14269

Query plan3:
iscan
    class: phm_employee node[0]
    index: pk_phm_employee_emp_id_start_ymd term[0]
    sort:  1 asc, 2 asc
    cost:  4149 card 14269
```

- 인덱스로 엑세스 하는 순서와 ORDER BY 순서가 동일하므로 옵티마이져는 SQL에 ORDER BY가 있더라도 이를 무시하고 인덱스로 처리하여 바로 리턴하는 부분범위 처리 방식으로 실행계획을 수립하게 됨.
- 주의할 점은 ORDER BY 에 사용된 컬럼의 순서와 개수가 생성된 인덱스의 앞부분과 정확히 동일해야만 ORDER BY가 무시되므로, ORDER BY를 하지 않더라도 결과가 인덱스의 순서를 따른다면 ORDER BY는 기술하지 않는 것이 좋음.

3. 집합연산 
- 집합연산은 그 결과가 반드시 유일해야 하기 때문에 전체범위를 모두 액세스 한 후 SORT(UNIQUE)을 수행하는 단계가 있음.
- ex) {1,2,2,3}, {2,4,6} => {1,2,2,2,3,4,6} => {1,2,3,4,6}
- 반대로 UNION ALL 은 부분범위 처리가 가능.
- 굳이 두개의 집합에 있는 중복을 제거할 이유가 없다면 UNION 보다 UNION ALL 이 더 유리하다.

#### 1.2.2 옵티마이져  모드에 따른 부분범위처리

- FIRST_ROWS vs ALL_ROWS

```sql
SELECT ord_dept, ordqty
  FROM order
 WHERE ord_dept > '1000' 
```

### 1.3 부분범위처리의 수행속도 향상원리 

```sql
SELECT * FROM order;

SELECT * FROM order
 ORDER BY item;
```

- 처리가 늦어지는 이유는 정렬작업 때문만은 아님.

```sql
SELECT * FROM order
 WHERE item > ' ';
```

- SQL에 따라 어떤 컬럼의 처리범위가 좁아질수록 수행속도가 향상되는 경우가 있고, 오히려 넓어질수록 수행속도가 향상되는 경우가 있음.

```sql
SELECT *
  FROM order
 WHERE ordno BETWEEN 1 AND 1000
   AND custno LIKE 'DN%'
```

- 가정
  - ORDER 테이블에는 위 ordno 조건을 만족하는 로우가 1000건이 있고, custno 조건을 만족하는 로우가 10건이 있음.
  - ordno, custno 별도의 인덱스가 있음.
  - 인덱스 조인으로 실행되지 않아 두개의 인덱스를 머지하는 실행계획은 수립되지 않음.
  
- ordno 인덱스를 사용한 경우 : 최악의 경우 모든 범위 (1000건)을 완료해야만 멈춤.
- custno 인덱스를 사용한 경우 : 최대 10회만 처리하면 됨. ordno 의 조건을 만족하는 로우는 1000건 이므로 custno 인덱스를 경유해 엑세스한 로우들은 대부분 이 조건을 만족하게 됨.

- 이와같이 엑세스를 주관하는 조건은 범위가 적을수록 유리함.

- 결론

  1. 엑세스를 주관하는 컬럼의 처리범위는 좁을수록 유리하다. 이러한 경우는 나머지 컬럼들의 처리범위에 영향을 적게 받는다. 즉, 언제나 빠른 수행속도를 보장받을 수 있다.
  2. 엑세스를 주관하는 컬럼의 범위가 넓더라도 그 외의 조건을 만족하는 범위가 넓다면 역시 빠른 수행속도를 보장받을 수 있다.
  3. 엑세스 주관 컬럼의 범위가 넓고, 그외 컬럼의 범위가 좁아서 늦어지는 경우는 처리범위가 좁은 컬럼이 엑세스 주관 컬럼이 되도록하면 해결된다. 이런 경우 힌트나 사용제한 기능을 이용하여 옵티마이져를 도와주면 됨.


```sql
SELECT *
  FROM order
 WHERE RTRIM(ordno) BETWEEN 1 AND 1000
   AND custno LIKE 'DN%'
   
SELECT  /* + INDEX(ORDER custno_index) */   
  FROM order
 WHERE ordno BETWEEN 1 AND 1000
   AND custno LIKE 'DN%'
```

### 1.4 부분범위처리로의 유도 
RDBMS 가 가지고 있는 기능을 최대한 활용한다면 보다 많은 처리를 부분범위처리 방법으로 유도할 수 있음.

#### 1.4.1 엑세스 경로를 이용한 SORT 의 대체
- 인덱스를 이용하여 ORDER BY 를 대체함. 그림 2-1-7.
- 원하는 정렬이 item_cd DESC 가 아닌, item_cd DESC, category DESC 였으면 이런 방식으로는 유도 불가능.
- 인덱스가 item_cd + category 로 결합되어 있다면 가능.
- 이처럼 부분범위 처리로 유도하기 위해서는 인덱스를 구성하고 있는 컬럼의 순서가 매우 중요한 결정요소가 됨.

```sql
SELECT ord_dept, ordqty * 1000
  FROM order
 WHERE ord_date LIKE '2017%'
 ORDER BY ord_dept DESC 
```

- 위 SQL은 엑세스를 주관하는 컬럼은 ord_date 이지만, 다른 컬럼인 ord_dept 의 역순으로 정렬되기를 원하고 있음.

```sql
SELECT /* + INDEX_DESC(a ord_dept_index) */
      ord_dept, ordqty * 1000
  FROM order a 
 WHERE ord_date LIKE '2017%'
   AND ord_dept > ' '
```

- AND ord_dept > ' ' 을 추가함으로써 액세스를 주관하는 컬럼과 ORDER BY할 컬럼이 같아졌음.
- 엑세스 주관 컬럼의 처리범위가 넓어도 다른 조건의 처리범위와 같이 넓으면 빠르다는 원리를 적용.

#### 1.4.2 인덱스만 액세스하는 부분범위처리
- 인덱스만 엑세스하도록 하는 실행계획으로 유도하기 위한 후보컬럼 선정도 중요한 전략중 하나임을 강조. 그림 2-1-8.
- 하지만 인덱스에 함부로 컬럼을 추가하면 나쁜 실행계획이 나타날 수 있음을 유의.

```sql
SELECT ord_date, SUM(qty)
  FROM order
 WHERE ord_date LIKE '201705%'
 GROUP BY ord_date
```

- 위처럼 WHERE 절에 사용하지 않은 컬럼도 인덱스만 사용하도록 유도할 목적으로 결합 인덱스에 추가할 수 있음.

```sql
// ORD_DATE + AGENT_CD 로 결합된 인덱스가 있다고 가정 

SELECT ord_dept, COUNT(*)
  FROM order
 WHERE ord_date LIKE '2017%'
 GROUP BY ord_dept
 
SELECT agent_cd, COUNT(*)
  FROM order
 WHERE ord_date LIKE '2017%'
 GROUP BY agent_cd
```

#### 1.4.3 MIN, MAX의 처리

- 기본키의 마지막 일련번호를 찾아 새로운 번호를 부여하는 처리는 실무에서 아주 많이 사용되고 있지만 대부분이 좋지 못한 방법을 사용함.
- 시퀀스를 사용하면 수행속도에 매우 유리한데 실전에서는 그다지 많이 활용 되고 있지 않음(?).
  1. 하나의 컬럼으로 인조 식별자를 만들 때 임의의 값으로 생성되는것을 매우 싫어하는 경향이 있음.
  2. 중간에 누락이 되는 번호가 생김.
  3. 특정 분류단위로 일련번호가 증가하는 경우에만 처리할 수가 없음.

- FIRST ROW, RANGE SCAN(MIN/MAX) : 인덱스를 순차적으로 스캔하여 첫 번째 로우만 추출하고 멈춤. 완벽한 부분범위 처리를 함.
- 위와 같은 실행계획이 나타나지 않는다면 다음과 같이 처리

```sql
SELECT /* + INDEX_DESC(order pk_order) */
      NVL(MAX(SEQ), 0) + 1
  FROM order
 WHERE dept_no = '12300'
   AND ROWNUM = 1
```

- ROWNUM 을 이용하여 FIRST ROW와 같은 효과를 내게 함.

#### 1.4.4 FILTER형 부분범위 처리
- 애플리케이션을 작성하는 과정에서 우리는 어떤 조건을 만족하는 집합의 존재여부만 확인하는 경우를 자주 접하게 됨.
- 존재의 여부를 확인하기 위해 별생각없이 함부로 사용한 SQL 때문에 전체범위를 모두 처리하는 우를 범하게 됨. 그림 2-1-9.
- EXSISTS를 사용한 실행계획은 FILTER로 나타남.
- MINUS <==> NOT EXISTS 

#### 1.4.5 ROWNUM의 활용
- ROWNUM이란 : 모든 SQL에 그냥 삽입해서 사용할 수 있는 일종의 가상컬럼. SQL이 실행되는 과정에서 발생하는 일련번호이므로 각 SQL 수행시마다 같은 로우라 하더라도 서로 다른 ROWNUM을 가질 수 있음.
- ROWNUM이 결정되는 과정을 정확히 알아야 원하는 결과를 추출할 수 있음. 그림 2-1-10.
- ROWNUM은 엑세스되는 로우의 번호가 아니라 조건을 만족한 '결과에 대한 일련번호' 이므로 우리가 10건만 요구하였더라도 내부적으로 훨씬 많은 로우가 엑세스 될 수 있음.
- 그러므로 추출되는 로우중에 10번째 로우를 찾기위해 WHERE 절에 ROWNUM = 10을 요구했다면 이 조건을 만족하는 로우는 결코 찾을 수 없음.
- ROWNUM의 실행계획 : COUNT(STOPKEY)
- 그러므로 추출되는 로우중에 10번째 로우를 찾기위해 WHERE 절에 ROWNUM = 10을 요구했다면 이 조건을 만족하는 로우는 결코 찾을 수 없음.
- 테이블을 부분범위 처리로 엑세스하여 ROWNUM을 COUNT하다가 주어진 ROWNUM 조건에 도달하면 멈춤(skopkey) 하겠다는 의미. 확실히 일정범위만 스캔함을 확인 가능.


#### 1.4.6 인라인뷰를 이용한 부분범위 처리 
- TO BE CONTINUED...

## 최선영 : 454 ~ 528 (2부 2.2.2절)
