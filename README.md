# DB
DB 내용 정리

1. 실행 된 쿼리에 따른 SQL_ID 조회
2. 조회된 SQL_ID를 통해서 XPLAN 확인

```sql
select SQL_ID,SQL_TEXT from v$sqltext where 1 = 1 AND SQL_TEXT LIKE '%KKK%';

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('brqyxb2mnbtx4', 0, 'ALLSTATS LAST'));
```

XPLAN 확인 

### **AS-IS**

1) 원본 쿼리

```sql
SELECT 
SUM (NVL (STOP_CNT, 0))           AS STOP_CNT,        
SUM (NVL (FORCE_STOP_CNT, 0))     AS FORCE_STOP_CNT   
FROM ( 
SELECT decode(gdh.sale_gb, '11', 1) AS stop_cnt,
decode(gdh.sale_gb,'19', 1) AS force_stop_cnt
FROM
    tgoods        gd,
    tgoodshistory gdh,
    tenterprise   c
WHERE
        gd.goods_code = gdh.goods_code
    AND gd.entp_code = c.entp_code
        AND c.entp_code = gdh.entp_code
            AND gdh.rowid = (
            SELECT /*+ INDEX_DESC( D PK_TGOODSHISTORY) */
                ROWID
            FROM
                tgoodshistory d
            WHERE
                    d.goods_code = gd.goods_code
                AND d.insert_date <= sysdate
                    AND ROWNUM = 1
        )
                AND gd.modify_date >= trunc
```

2) XPLAN

```sql
-----------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name               | E-Rows |E-Bytes| Cost (%CPU)| E-Time   |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                    |        |       |   316 (100)|          |       |       |          |
|   1 |  SORT AGGREGATE                  |                    |      1 |    69 |            |          |       |       |          |
|   2 |   NESTED LOOPS                   |                    |      1 |    69 |   309   (0)| 00:00:01 |       |       |          |
|   3 |    NESTED LOOPS                  |                    |     35 |  1330 |   274   (0)| 00:00:01 |       |       |          |
|*  4 |     INDEX RANGE SCAN             | IDX_TENTERPRISE_01 |      1 |    14 |     2   (0)| 00:00:01 |  1025K|  1025K|          |
|*  5 |     TABLE ACCESS BY INDEX ROWID  | TGOODS             |     70 |  1680 |   272   (0)| 00:00:01 |       |       |          |
|*  6 |      INDEX RANGE SCAN            | IDX_TGOODS_03      |    677 |       |     4   (0)| 00:00:01 |  1025K|  1025K|          |
|*  7 |    TABLE ACCESS BY USER ROWID    | TGOODSHISTORY      |      1 |    31 |     1   (0)| 00:00:01 |       |       |          |
|*  8 |     COUNT STOPKEY                |                    |        |       |            |          |       |       |          |
|*  9 |      TABLE ACCESS BY INDEX ROWID | TGOODSHISTORY      |      2 |    58 |     7   (0)| 00:00:01 |       |       |          |
|* 10 |       INDEX RANGE SCAN DESCENDING| PK_TGOODSHISTORY   |     17 |       |     4   (0)| 00:00:01 |  1025K|  1025K|          |
-----------------------------------------------------------------------------------------------------------------------------------

   4 - access("C"."MASTER_ENTP_CODE"='100001')
   5 - filter("GD"."MODIFY_DATE">=TRUNC(SYSDATE@!-31))
   6 - access("GD"."ENTP_CODE"="C"."ENTP_CODE")
   7 - filter(("GD"."GOODS_CODE"="GDH"."GOODS_CODE" AND "C"."ENTP_CODE"="GDH"."ENTP_CODE"))
   8 - filter(ROWNUM=1)
   9 - filter("D"."INSERT_DATE"<=SYSDATE@!)
  10 - access("D"."GOODS_CODE"=:B1)
```

실행 내용 

1) IDX_TENTERPRISE_01 인덱스를 통해서 tenterprise  테이블을 조회

2) USE_NL 방식으로 TGOODS 테이블 조회(TGOODS 테이블을 IDX_TGOODS_03 을 통해서 조회)

3) TGOODSHISTORY 를 PK_TGOODSHISTORY 를 통해서 줄인 다음에 USE_NL 방식으로 조입한다.

여기서 추가적으로 필터를 보면

4번 → tenterprise 업체코드로 필터   

5번 → tgoods modify_date로 필터

6번 → tgoods 업체코드로 필터

7번 → tgoodshistory 테이블 필터 조건 적용’

8번 → ?

9번 → tgoodshistory 테이블 조건으로 필터 적용

→ 정리를 해보면 tenterprise 테이블과 tgoods를 use_nl(루프) 처리하고 tgoodshistory를 use_nl(루프 처리해서 데이터를 가지고 오고 있다.

(* 힌트 아래 부분을 보면 조인 하기 전에 조건으로 테이블 range 감소 시킴)

- **어떤 식으로 튜닝할 것인가**

→ tenterprise 테이블과 tgoods를 조인 할때 use_hash(메모리에 올리고 합침) 방식을 사용한다.

(* 단 tgoods의 양이 적은 경우 use_hash가 무리가 될 수 있으므로 상품 수에 따라서 use_nl, use_hash 를 분기 처리 한다.

### T**O-BE**

1) 원본 쿼리

```sql
SELECT 
SUM (NVL (STOP_CNT, 0))           AS STOP_CNT,        
SUM (NVL (FORCE_STOP_CNT, 0))     AS FORCE_STOP_CNT   
FROM ( 
SELECT 
/*+ LEADING(C GD GDH) USE_HASH(GD) */
decode(gdh.sale_gb, '11', 1) AS stop_cnt,
decode(gdh.sale_gb,'19', 1) AS force_stop_cnt
FROM
    tgoods        gd,
    tgoodshistory gdh,
    tenterprise   c
WHERE
        gd.goods_code = gdh.goods_code
    AND gd.entp_code = c.entp_code
        AND c.entp_code = gdh.entp_code
            AND gdh.rowid = (
            SELECT /*+ INDEX_DESC( D PK_TGOODSHISTORY) */
                ROWID
            FROM
                tgoodshistory d
            WHERE
                    d.goods_code = gd.goods_code
                AND d.insert_date <= sysdate
                    AND ROWNUM = 1
        )
                AND gd.modify_date >= trunc
```

→ LEADING 으로 조인 순서를 tenterprise  tgoods  tgoodshistory  으로 정의 하고 TGOODS는 HASH 방식으로 JOIN 되도록 처리한다.

  2) XPLAN

```sql
-----------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name               | E-Rows |E-Bytes| Cost (%CPU)| E-Time   |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                    |        |       |   409K(100)|          |       |       |          |
|   1 |  SORT AGGREGATE                  |                    |      1 |    72 |            |          |       |       |          |
|   2 |   NESTED LOOPS                   |                    |     24 |  1728 |   408K  (1)| 00:00:16 |       |       |          |
|*  3 |    HASH JOIN                     |                    |    175K|  7014K|   233K  (1)| 00:00:10 |  4212K|  4212K| 4367K (0)|
|   4 |     JOIN FILTER CREATE           | :BF0000            |   4318 | 60452 |    26   (0)| 00:00:01 |       |       |          |
|*  5 |      INDEX RANGE SCAN            | IDX_TENTERPRISE_01 |   4318 | 60452 |    26   (0)| 00:00:01 |  1025K|  1025K|          |
|   6 |     JOIN FILTER USE              | :BF0000            |    934K|    24M|   233K  (1)| 00:00:10 |       |       |          |
|*  7 |      TABLE ACCESS STORAGE FULL   | TGOODS             |    934K|    24M|   233K  (1)| 00:00:10 |  1025K|  1025K| 5142K (0)|
|*  8 |    TABLE ACCESS BY USER ROWID    | TGOODSHISTORY      |      1 |    31 |     1   (0)| 00:00:01 |       |       |          |
|*  9 |     COUNT STOPKEY                |                    |        |       |            |          |       |       |          |
|* 10 |      TABLE ACCESS BY INDEX ROWID | TGOODSHISTORY      |      2 |    58 |     7   (0)| 00:00:01 |       |       |          |
|* 11 |       INDEX RANGE SCAN DESCENDING| PK_TGOODSHISTORY   |     17 |       |     4   (0)| 00:00:01 |  1025K|  1025K|          |
-----------------------------------------------------------------------------------------------------------------------------------

3 - access("GD"."ENTP_CODE"="C"."ENTP_CODE")
   5 - access("C"."MASTER_ENTP_CODE"='415497')
   7 - storage((INTERNAL_FUNCTION("GD"."SALE_GB") AND "GD"."MODIFY_DATE">=TRUNC(SYSDATE@!-31) AND 
              SYS_OP_BLOOM_FILTER(:BF0000,"GD"."ENTP_CODE")))
       filter((INTERNAL_FUNCTION("GD"."SALE_GB") AND "GD"."MODIFY_DATE">=TRUNC(SYSDATE@!-31) AND 
              SYS_OP_BLOOM_FILTER(:BF0000,"GD"."ENTP_CODE")))
   8 - filter((INTERNAL_FUNCTION("GDH"."SALE_GB") AND "GD"."GOODS_CODE"="GDH"."GOODS_CODE" AND 
              "C"."ENTP_CODE"="GDH"."ENTP_CODE"))
   9 - filter(ROWNUM=1)
  10 - filter("D"."INSERT_DATE"<=SYSDATE@!)
  11 - access("D"."GOODS_CODE"=:B1)

```

실행 내용 

→ 실행 내용 앞에 내용과 동일하지만 tgoods는 use_hash 로 처리하여서 성능 개선이 되었다.

## **USE_NL VS USE_HASH**

![image](https://user-images.githubusercontent.com/56577599/216822799-178bf273-5a58-4237-803d-2e96b130f215.png)


1. 고객 테이블에서 고객명이 ‘홍길동’인 고객을 구한다.(선행 테이블 결정)
2. ‘홍길동’ 고객의 수 만큼 순차적으로 주문 테이블을 고객번호 컬럼으로 접근한다.(순차적 접근)
3. 주문 테이블에서 주민일자가 ‘20141201’인 정보만 필터한다.

(* 여기서 궁금한 거는 주문을 주문일자로 필터하고 순차적 처리를 하는 게 좋을 까? 아니면 순차적으로 JOIN 하고 나서 “주문일자”로 데이터를 줄이는 것이 좋을까?)

**→ 여기서 봤을 때는 JOIN 이 인덱스 컬럼이 있어서 빠를 것처럼 보여서 JOIN을 먼저 하면 좋을 거 같아보이는 데 PUSH_PRED 옵션을 통해서 조절 가능 한 것 인가?**

![image](https://user-images.githubusercontent.com/56577599/216822760-97cb6c5b-b16d-4bc6-a553-34633e45ef05.png)


1. 조직 테이블에서 사업부가 ‘강원사업부’인 조직들을 수한 후,  조인절 컬럼인 조직코드를 통해 해시 함수로 분류한 다음, 해시 테이블을 생성한ㄷ.

1. 집계 테이블에서 집계년원이 ‘201412’ 인 자료를 구한 후, 조인절 컬럼인 조직코드를 해시 함수로 변환 후 해시 테이블에 순차적으로 접근한다.

(* 여기서 2번 3번 인덱스는 사용 하지 않는다.

**→ 여기서 봤을 때는 조인 조건에 인덱스 조회 조건이 있으면 USE_NL 을 써도 나쁘지 않을 수 도 있다는 생각이든다.**
