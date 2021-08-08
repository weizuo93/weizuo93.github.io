---
layout: post
title: 基于SSB的ClickHouse性能测试
categories: ClickHouse
description: ClickHouse性能测试
keywords: ClickHouse
---

## 基于SSB的ClickHouse性能测试

### 1. 数据生成

编译dbgen：
```
$ git clone git@github.com:vadimtk/ssb-dbgen.git
$ cd ssb-dbgen
$ make
```

生成数据：
```
$ ./dbgen -s 100 -T c
$ ./dbgen -s 100 -T l
$ ./dbgen -s 100 -T p
$ ./dbgen -s 100 -T s
$ ./dbgen -s 100 -T d
```
使用`-s 100`将生成 6 亿行数据(67GB), 如果使用`-s 1000`会生成 60 亿行数据。

### 2. 建表

lineorder表
```
CREATE TABLE lineorder on cluster cluster01
(
    LOORDERKEY             UInt32,
    LOLINENUMBER           UInt8,
    LOCUSTKEY              UInt32,
    LOPARTKEY              UInt32,
    LOSUPPKEY              UInt32,
    LOORDERDATE            Date,
    LOORDERPRIORITY        LowCardinality(String),
    LOSHIPPRIORITY         UInt8,
    LOQUANTITY             UInt8,
    LOEXTENDEDPRICE        UInt32,
    LOORDTOTALPRICE        UInt32,
    LODISCOUNT             UInt8,
    LOREVENUE              UInt32,
    LOSUPPLYCOST           UInt32,
    LOTAX                  UInt8,
    LOCOMMITDATE           Date,
    LOSHIPMODE             LowCardinality(String)
)
ENGINE = MergeTree PARTITION BY toYear(LOORDERDATE) ORDER BY (LOORDERDATE, LOORDERKEY);



CREATE TABLE ssb.lineorder_all  on cluster cluster01 AS ssb.lineorder 
ENGINE = Distributed(cluster01, ssb, lineorder, rand());
```

customer表
```
CREATE TABLE customer on cluster cluster01
(
        CCUSTKEY       UInt32,
        CNAME          String,
        CADDRESS       String,
        CCITY          LowCardinality(String),
        CNATION        LowCardinality(String),
        CREGION        LowCardinality(String),
        CPHONE         String,
        CMKTSEGMENT    LowCardinality(String)
)
ENGINE = MergeTree ORDER BY (CCUSTKEY);



CREATE TABLE ssb.customer_all  on cluster cluster01 AS ssb.customer 
ENGINE = Distributed(cluster01, ssb, customer, rand());
```

supplier表
```
CREATE TABLE supplier on cluster cluster01
(
        SSUPPKEY       UInt32,
        SNAME          String,
        SADDRESS       String,
        SCITY          LowCardinality(String),
        SNATION        LowCardinality(String),
        SREGION        LowCardinality(String),
        SPHONE         String
)
ENGINE = MergeTree ORDER BY SSUPPKEY;


CREATE TABLE ssb.supplier_all  on cluster cluster01 AS ssb.supplier 
ENGINE = Distributed(cluster01, ssb, supplier, rand());
```

part表
```
CREATE TABLE part on cluster cluster01
(
        PPARTKEY       UInt32,
        PNAME          String,
        PMFGR          LowCardinality(String),
        PCATEGORY      LowCardinality(String),
        PBRAND         LowCardinality(String),
        PCOLOR         LowCardinality(String),
        PTYPE          LowCardinality(String),
        PSIZE          UInt8,
        PCONTAINER     LowCardinality(String)
)
ENGINE = MergeTree ORDER BY PPARTKEY;


CREATE TABLE ssb.part_all  on cluster cluster01 AS ssb.part 
ENGINE = Distributed(cluster01, ssb, part, rand());
```

date表
```
CREATE TABLE date on cluster cluster01
(
        DDATEKEY             UInt32,
        DDATE                String,
        DDAYOFWEEK           String,
        DMONTH               String,
        DYEAR                UInt32,
        DYEARMONTHNUM        UInt32,
        DYEARMONTH           String,
        DDAYNUMINWEEK        UInt32,
        DDAYNUMINMONTH       UInt32,
        DDAYNUMINYEAR        UInt32,
        DMONTHNUMINYEAR      UInt32,
        DWEEKNUMINYEAR       UInt32,
        DSELLINGSEASON       String,
        DLASTDAYINWEEKFL     UInt32,
        DLASTDAYINMONTHFL    UInt32,
        DHOLIDAYFL           UInt32,
        DWEEKDAYFL           UInt32
) 
ENGINE = MergeTree ORDER BY DDATEKEY;


CREATE TABLE ssb.date_all  on cluster cluster01 AS ssb.date 
ENGINE = Distributed(cluster01, ssb, date, rand());
```

### 3. 数据导入

```
[weizuo@PC ClickHouse]$ ./clickhouse client -h host -u xxx --password xxx --time --query "INSERT INTO ssb.customer_all FORMAT CSV" < ~/ssb-dbgen/customer.tbl
3.517
[weizuo@PC ClickHouse]$ ./clickhouse client -h host -u xxx --password xxx --time --query "INSERT INTO ssb.part_all FORMAT CSV" < ~/ssb-dbgen/part.tbl
2.589
[weizuo@PC ClickHouse]$ ./clickhouse client -h host -u xxx --password xxx --time --query "INSERT INTO ssb.supplier_all FORMAT CSV" < ~/ssb-dbgen/supplier.tbl
0.937
[weizuo@PC ClickHouse]$ ./clickhouse client -h host -u xxx --password xxx --time --query "INSERT INTO ssb.lineorder_all FORMAT CSV" < ~/ssb-dbgen/lineorder.tbl
355.095
[weizuo@PC ClickHouse]$ ./clickhouse client -h host -u xxx --password xxx --time --query "INSERT INTO ssb.date_all FORMAT CSV" < ~/ssb-dbgen/date.tbl
0.192
[weizuo@PC ClickHouse]$
[weizuo@PC ClickHouse]$ ll ~/ssb-dbgen/*.tbl
-rw---S--T 1 weizuo weizuo   331529327 Dec 29 12:00 /home/weizuo/ssb-dbgen/customer.tbl
-rw-r--r-- 1 weizuo weizuo      275973 Dec 29 14:00 /home/weizuo/ssb-dbgen/date.tbl
-rw-r--r-- 1 weizuo weizuo 70969689336 Dec 29 13:36 /home/weizuo/ssb-dbgen/lineorder.tbl
-rw-r--r-- 1 weizuo weizuo   140642413 Dec 29 13:58 /home/weizuo/ssb-dbgen/part.tbl
-rw-r--r-- 1 weizuo weizuo    19462852 Dec 29 13:59 /home/weizuo/ssb-dbgen/supplier.tbl
```

### 4. 多表Join查询

查看系统并发：
```
host :) show settings like 'max_threads';

SHOW SETTINGS LIKE 'max_threads'

Query id: 088f7232-1d9e-4c1c-89d0-69bc475065be

┌─name────────┬─type───────┬─value──────┐
│ max_threads │ MaxThreads │ 'auto(16)' │
└─────────────┴────────────┴────────────┘

1 rows in set. Elapsed: 0.005 sec.
```

Q1.1
```
host :) select sum(LOREVENUE) as revenue
                          from lineorder_all 
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                          where DYEAR = 1993 and LODISCOUNT between 1 and 3 and LOQUANTITY < 25;

SELECT sum(LOREVENUE) AS revenue
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
WHERE (DYEAR = 1993) AND ((LODISCOUNT >= 1) AND (LODISCOUNT <= 3)) AND (LOQUANTITY < 25)

Query id: 4e01fdf1-d437-4943-ad1c-f50f0ac90486

┌────────revenue─┐
│ 21881848590256 │
└────────────────┘

1 rows in set. Elapsed: 0.467 sec. Processed 600.04 million rows, 4.80 GB (1.28 billion rows/s., 10.28 GB/s.)
```

Q1.2
```
host :) select sum(LOREVENUE) as revenue
                          from lineorder_all 
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                          where DYEARMONTHNUM = 199401 and LODISCOUNT between 4 and 6 and LOQUANTITY between 26 and 35;

SELECT sum(LOREVENUE) AS revenue
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
WHERE (DYEARMONTHNUM = 199401) AND ((LODISCOUNT >= 4) AND (LODISCOUNT <= 6)) AND ((LOQUANTITY >= 26) AND (LOQUANTITY <= 35))

Query id: 7f90b183-0492-45c8-9b10-7fc1836a85f5

┌───────revenue─┐
│ 1829705342413 │
└───────────────┘

1 rows in set. Elapsed: 0.447 sec. Processed 600.04 million rows, 4.80 GB (1.34 billion rows/s., 10.73 GB/s.)
```

Q1.3
```
host :) select sum(LOREVENUE) as revenue
                          from lineorder_all 
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                          where DWEEKNUMINYEAR = 6 and DYEAR = 1994 and LODISCOUNT between 5 and 7 and LOQUANTITY between 26 and 35;

SELECT sum(LOREVENUE) AS revenue
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
WHERE (DWEEKNUMINYEAR = 6) AND (DYEAR = 1994) AND ((LODISCOUNT >= 5) AND (LODISCOUNT <= 7)) AND ((LOQUANTITY >= 26) AND (LOQUANTITY <= 35))

Query id: 4bee8f99-cf55-4ef2-a706-28b85c391067

┌──────revenue─┐
│ 407995993835 │
└──────────────┘

1 rows in set. Elapsed: 0.449 sec. Processed 600.04 million rows, 4.80 GB (1.34 billion rows/s., 10.69 GB/s.)
```

Q2.1
```
host :) select sum(LOREVENUE) as lorevenue, DYEAR, PBRAND
                          from lineorder_all
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                               global join part_all on LOPARTKEY = PPARTKEY
                               global join supplier_all on LOSUPPKEY = SSUPPKEY
                          where PCATEGORY = 'MFGR#12' and SREGION = 'AMERICA'
                          group by DYEAR, PBRAND
                          order by DYEAR, PBRAND;

SELECT
    sum(LOREVENUE) AS lorevenue,
    DYEAR,
    PBRAND
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
WHERE (PCATEGORY = 'MFGR#12') AND (SREGION = 'AMERICA')
GROUP BY
    DYEAR,
    PBRAND
ORDER BY
    DYEAR ASC,
    PBRAND ASC

Query id: d7b46c9a-5f7c-4430-84ca-f9351cee2af1

┌───lorevenue─┬─DYEAR─┬─PBRAND────┐
│ 64420005618 │  1992 │ MFGR#121  │
│ 63389346096 │  1992 │ MFGR#1210 │
│ 68416605637 │  1992 │ MFGR#1211 │
│ 64135723264 │  1992 │ MFGR#1212 │
│ 66331370055 │  1992 │ MFGR#1213 │
            ...
│ 37043563027 │  1998 │ MFGR#126  │
│ 38499217759 │  1998 │ MFGR#127  │
│ 39679892915 │  1998 │ MFGR#128  │
│ 35300513083 │  1998 │ MFGR#129  │
└─────────────┴───────┴───────────┘

280 rows in set. Elapsed: 20.981 sec. Processed 601.65 million rows, 8.41 GB (28.68 million rows/s., 400.92 MB/s.)
```

Q2.2
```
host :) select sum(LOREVENUE) as lorevenue, DYEAR, PBRAND
                          from lineorder_all
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                               global join part_all on LOPARTKEY = PPARTKEY
                               global join supplier_all on LOSUPPKEY = SSUPPKEY
                          where PBRAND between 'MFGR#2221' and 'MFGR#2228' and SREGION = 'ASIA'
                          group by DYEAR, PBRAND
                          order by DYEAR, PBRAND;

SELECT
    sum(LOREVENUE) AS lorevenue,
    DYEAR,
    PBRAND
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
WHERE ((PBRAND >= 'MFGR#2221') AND (PBRAND <= 'MFGR#2228')) AND (SREGION = 'ASIA')
GROUP BY
    DYEAR,
    PBRAND
ORDER BY
    DYEAR ASC,
    PBRAND ASC

Query id: 1a66ed43-709d-439b-9dba-84331bd0a424

┌───lorevenue─┬─DYEAR─┬─PBRAND────┐
│ 66450349438 │  1992 │ MFGR#2221 │
│ 65423264312 │  1992 │ MFGR#2222 │
│ 66936772687 │  1992 │ MFGR#2223 │
│ 64047191934 │  1992 │ MFGR#2224 │
│ 65744559138 │  1992 │ MFGR#2225 │
│ 66993045668 │  1992 │ MFGR#2226 │
│ 67411226147 │  1992 │ MFGR#2227 │
│ 69390885970 │  1992 │ MFGR#2228 │
│ 66819757447 │  1993 │ MFGR#2221 │
│ 67805601887 │  1993 │ MFGR#2222 │
│ 67208412655 │  1993 │ MFGR#2223 │
│ 64222070981 │  1993 │ MFGR#2224 │
│ 66159498618 │  1993 │ MFGR#2225 │
│ 68387963965 │  1993 │ MFGR#2226 │
│ 68470823598 │  1993 │ MFGR#2227 │
│ 70176992353 │  1993 │ MFGR#2228 │
│ 66201929022 │  1994 │ MFGR#2221 │
│ 66601352347 │  1994 │ MFGR#2222 │
│ 67149651412 │  1994 │ MFGR#2223 │
│ 64508853727 │  1994 │ MFGR#2224 │
│ 66008726728 │  1994 │ MFGR#2225 │
│ 66358870053 │  1994 │ MFGR#2226 │
│ 67912269895 │  1994 │ MFGR#2227 │
│ 69071503806 │  1994 │ MFGR#2228 │
│ 65269818712 │  1995 │ MFGR#2221 │
│ 65566595895 │  1995 │ MFGR#2222 │
│ 65980940491 │  1995 │ MFGR#2223 │
│ 63741007905 │  1995 │ MFGR#2224 │
│ 64701224302 │  1995 │ MFGR#2225 │
│ 67771832811 │  1995 │ MFGR#2226 │
│ 67898964035 │  1995 │ MFGR#2227 │
│ 70977692364 │  1995 │ MFGR#2228 │
│ 67169465617 │  1996 │ MFGR#2221 │
│ 67121666299 │  1996 │ MFGR#2222 │
│ 66485923436 │  1996 │ MFGR#2223 │
│ 64410797788 │  1996 │ MFGR#2224 │
│ 65788692665 │  1996 │ MFGR#2225 │
│ 68193662121 │  1996 │ MFGR#2226 │
│ 67904649725 │  1996 │ MFGR#2227 │
│ 69705348599 │  1996 │ MFGR#2228 │
│ 66839293911 │  1997 │ MFGR#2221 │
│ 65623735495 │  1997 │ MFGR#2222 │
│ 66608624781 │  1997 │ MFGR#2223 │
│ 64127451073 │  1997 │ MFGR#2224 │
│ 66071861556 │  1997 │ MFGR#2225 │
│ 68517706654 │  1997 │ MFGR#2226 │
│ 67632192229 │  1997 │ MFGR#2227 │
│ 70029520291 │  1997 │ MFGR#2228 │
│ 39646973602 │  1998 │ MFGR#2221 │
│ 38969579899 │  1998 │ MFGR#2222 │
│ 38767988496 │  1998 │ MFGR#2223 │
│ 38020572188 │  1998 │ MFGR#2224 │
│ 38328423898 │  1998 │ MFGR#2225 │
│ 38705033272 │  1998 │ MFGR#2226 │
│ 39907545239 │  1998 │ MFGR#2227 │
│ 40654201840 │  1998 │ MFGR#2228 │
└─────────────┴───────┴───────────┘

56 rows in set. Elapsed: 12.453 sec. Processed 601.65 million rows, 8.41 GB (48.31 million rows/s., 675.37 MB/s.)
```

Q2.3
```
host :) select sum(LOREVENUE) as lorevenue, DYEAR, PBRAND
                          from lineorder_all
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                               global join part_all on LOPARTKEY = PPARTKEY
                               global join supplier_all on LOSUPPKEY = SSUPPKEY
                          where PBRAND = 'MFGR#2239' and SREGION = 'EUROPE'
                          group by DYEAR, PBRAND
                          order by DYEAR, PBRAND;

SELECT
    sum(LOREVENUE) AS lorevenue,
    DYEAR,
    PBRAND
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
WHERE (PBRAND = 'MFGR#2239') AND (SREGION = 'EUROPE')
GROUP BY
    DYEAR,
    PBRAND
ORDER BY
    DYEAR ASC,
    PBRAND ASC

Query id: 0c0181e8-6a02-4225-83df-0d224e010c39

┌───lorevenue─┬─DYEAR─┬─PBRAND────┐
│ 65751589723 │  1992 │ MFGR#2239 │
│ 64532844801 │  1993 │ MFGR#2239 │
│ 64722599002 │  1994 │ MFGR#2239 │
│ 65616432683 │  1995 │ MFGR#2239 │
│ 64802884686 │  1996 │ MFGR#2239 │
│ 64485541165 │  1997 │ MFGR#2239 │
│ 37276536361 │  1998 │ MFGR#2239 │
└─────────────┴───────┴───────────┘

7 rows in set. Elapsed: 9.868 sec. Processed 601.65 million rows, 8.41 GB (60.97 million rows/s., 852.31 MB/s.)
```

Q3.1
```
host :) select CNATION, SNATION, DYEAR, sum(LOREVENUE) as lorevenue
                          from lineorder_all
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                               global join customer_all on LOCUSTKEY = CCUSTKEY
                               global join supplier_all on LOSUPPKEY = SSUPPKEY
                          where CREGION = 'ASIA' and SREGION = 'ASIA'and DYEAR >= 1992 and DYEAR <= 1997
                          group by CNATION, SNATION, DYEAR
                          order by DYEAR asc, lorevenue desc;

SELECT
    CNATION,
    SNATION,
    DYEAR,
    sum(LOREVENUE) AS lorevenue
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN customer_all ON LOCUSTKEY = CCUSTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
WHERE (CREGION = 'ASIA') AND (SREGION = 'ASIA') AND (DYEAR >= 1992) AND (DYEAR <= 1997)
GROUP BY
    CNATION,
    SNATION,
    DYEAR
ORDER BY
    DYEAR ASC,
    lorevenue DESC

Query id: a0d9f646-6364-4827-b7f1-e2ea93918cc3

┌─CNATION───┬─SNATION───┬─DYEAR─┬────lorevenue─┐
│ INDIA     │ INDIA     │  1992 │ 537778456208 │
│ INDONESIA │ INDIA     │  1992 │ 536684093041 │
│ VIETNAM   │ INDIA     │  1992 │ 536483529614 │
│ INDIA     │ JAPAN     │  1992 │ 535663357352 │
│ INDONESIA │ JAPAN     │  1992 │ 535044240518 │
                ...
│ JAPAN     │ CHINA     │  1997 │ 526303212379 │
│ CHINA     │ VIETNAM   │  1997 │ 525867745074 │
│ CHINA     │ CHINA     │  1997 │ 525562838002 │
│ JAPAN     │ VIETNAM   │  1997 │ 525495763677 │
└───────────┴───────────┴───────┴──────────────┘

150 rows in set. Elapsed: 44.290 sec. Processed 603.24 million rows, 8.42 GB (13.62 million rows/s., 190.11 MB/s.)
```

Q3.2
```
host :) select CCITY, SCITY, DYEAR, sum(LOREVENUE) as lorevenue
                          from lineorder_all
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                               global join customer_all on LOCUSTKEY = CCUSTKEY
                               global join supplier_all on LOSUPPKEY = SSUPPKEY
                          where CNATION = 'UNITED STATES' and SNATION = 'UNITED STATES'
                            and DYEAR >= 1992 and DYEAR <= 1997
                          group by CCITY, SCITY, DYEAR
                          order by DYEAR asc, lorevenue desc;

SELECT
    CCITY,
    SCITY,
    DYEAR,
    sum(LOREVENUE) AS lorevenue
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN customer_all ON LOCUSTKEY = CCUSTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
WHERE (CNATION = 'UNITED STATES') AND (SNATION = 'UNITED STATES') AND (DYEAR >= 1992) AND (DYEAR <= 1997)
GROUP BY
    CCITY,
    SCITY,
    DYEAR
ORDER BY
    DYEAR ASC,
    lorevenue DESC

Query id: 15e42dcc-2d9f-4c21-bd86-8be0b2a23491

┌─CCITY──────┬─SCITY──────┬─DYEAR─┬──lorevenue─┐
│ UNITED ST6 │ UNITED ST6 │  1992 │ 5694246807 │
│ UNITED ST0 │ UNITED ST0 │  1992 │ 5676049026 │
│ UNITED ST1 │ UNITED ST1 │  1992 │ 5652630617 │
│ UNITED ST8 │ UNITED ST6 │  1992 │ 5649039075 │
│ UNITED ST4 │ UNITED ST1 │  1992 │ 5618014301 │
                ...
│ UNITED ST2 │ UNITED ST9 │  1997 │ 4928308312 │
│ UNITED ST9 │ UNITED ST6 │  1997 │ 4877577655 │
│ UNITED ST3 │ UNITED ST2 │  1997 │ 4866105481 │
│ UNITED ST9 │ UNITED ST9 │  1997 │ 4836163349 │
│ UNITED ST9 │ UNITED ST5 │  1997 │ 4769919410 │
└────────────┴────────────┴───────┴────────────┘

600 rows in set. Elapsed: 17.421 sec. Processed 603.24 million rows, 8.42 GB (34.63 million rows/s., 483.33 MB/s.)
```

Q3.3
```
host :) select CCITY, SCITY, DYEAR, sum(LOREVENUE) as lorevenue
                          from lineorder_all
                                   global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                                   global join customer_all on LOCUSTKEY = CCUSTKEY
                                   global join supplier_all on LOSUPPKEY = SSUPPKEY
                          where (CCITY='UNITED KI1' or CCITY='UNITED KI5')
                            and (SCITY='UNITED KI1' or SCITY='UNITED KI5')
                            and DYEAR >= 1992 and DYEAR <= 1997
                          group by CCITY, SCITY, DYEAR
                          order by DYEAR asc, lorevenue desc;

SELECT
    CCITY,
    SCITY,
    DYEAR,
    sum(LOREVENUE) AS lorevenue
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN customer_all ON LOCUSTKEY = CCUSTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
WHERE ((CCITY = 'UNITED KI1') OR (CCITY = 'UNITED KI5')) AND ((SCITY = 'UNITED KI1') OR (SCITY = 'UNITED KI5')) AND (DYEAR >= 1992) AND (DYEAR <= 1997)
GROUP BY
    CCITY,
    SCITY,
    DYEAR
ORDER BY
    DYEAR ASC,
    lorevenue DESC

Query id: 3698eb80-8bcc-4dc3-8b9f-0d9f6dcd6d89

┌─CCITY──────┬─SCITY──────┬─DYEAR─┬──lorevenue─┐
│ UNITED KI1 │ UNITED KI1 │  1992 │ 5776096629 │
│ UNITED KI5 │ UNITED KI1 │  1992 │ 5555883901 │
│ UNITED KI5 │ UNITED KI5 │  1992 │ 5348705805 │
│ UNITED KI1 │ UNITED KI5 │  1992 │ 5326870427 │
│ UNITED KI1 │ UNITED KI1 │  1993 │ 5892974670 │
│ UNITED KI1 │ UNITED KI5 │  1993 │ 5490859451 │
│ UNITED KI5 │ UNITED KI1 │  1993 │ 5468354303 │
│ UNITED KI5 │ UNITED KI5 │  1993 │ 5089909647 │
│ UNITED KI5 │ UNITED KI1 │  1994 │ 5437315108 │
│ UNITED KI1 │ UNITED KI1 │  1994 │ 5348775917 │
│ UNITED KI5 │ UNITED KI5 │  1994 │ 5310936695 │
│ UNITED KI1 │ UNITED KI5 │  1994 │ 5237461110 │
│ UNITED KI1 │ UNITED KI1 │  1995 │ 5737551920 │
│ UNITED KI5 │ UNITED KI5 │  1995 │ 5657584590 │
│ UNITED KI5 │ UNITED KI1 │  1995 │ 5260093556 │
│ UNITED KI1 │ UNITED KI5 │  1995 │ 5213763257 │
│ UNITED KI5 │ UNITED KI1 │  1996 │ 5522325005 │
│ UNITED KI1 │ UNITED KI1 │  1996 │ 5451244409 │
│ UNITED KI5 │ UNITED KI5 │  1996 │ 5231759057 │
│ UNITED KI1 │ UNITED KI5 │  1996 │ 5203962897 │
│ UNITED KI1 │ UNITED KI1 │  1997 │ 5340760807 │
│ UNITED KI5 │ UNITED KI1 │  1997 │ 5295685214 │
│ UNITED KI1 │ UNITED KI5 │  1997 │ 5188428156 │
│ UNITED KI5 │ UNITED KI5 │  1997 │ 5024634475 │
└────────────┴────────────┴───────┴────────────┘

24 rows in set. Elapsed: 11.124 sec. Processed 603.24 million rows, 8.42 GB (54.23 million rows/s., 756.65 MB/s.)
```

Q3.4
```
host :) select CCITY, SCITY, DYEAR, sum(LOREVENUE) as lorevenue
                          from lineorder_all
                                   global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                                   global join customer_all on LOCUSTKEY = CCUSTKEY
                                   global join supplier_all on LOSUPPKEY = SSUPPKEY
                          where (CCITY='UNITED KI1' or CCITY='UNITED KI5')
                            and (SCITY='UNITED KI1' or SCITY='UNITED KI5')
                            and DYEARMONTH = 'Dec1997'
                          group by CCITY, SCITY, DYEAR
                          order by DYEAR asc, lorevenue desc;

SELECT
    CCITY,
    SCITY,
    DYEAR,
    sum(LOREVENUE) AS lorevenue
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN customer_all ON LOCUSTKEY = CCUSTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
WHERE ((CCITY = 'UNITED KI1') OR (CCITY = 'UNITED KI5')) AND ((SCITY = 'UNITED KI1') OR (SCITY = 'UNITED KI5')) AND (DYEARMONTH = 'Dec1997')
GROUP BY
    CCITY,
    SCITY,
    DYEAR
ORDER BY
    DYEAR ASC,
    lorevenue DESC

Query id: 0e497e95-fdc2-47fd-8424-fdf7e215506b

┌─CCITY──────┬─SCITY──────┬─DYEAR─┬─lorevenue─┐
│ UNITED KI1 │ UNITED KI1 │  1997 │ 481119563 │
│ UNITED KI5 │ UNITED KI5 │  1997 │ 386477033 │
│ UNITED KI5 │ UNITED KI1 │  1997 │ 378048353 │
│ UNITED KI1 │ UNITED KI5 │  1997 │ 366630529 │
└────────────┴────────────┴───────┴───────────┘

4 rows in set. Elapsed: 0.737 sec. Processed 603.24 million rows, 8.42 GB (818.86 million rows/s., 11.43 GB/s.)
```

Q4.1
```
host :) select DYEAR, CNATION, sum(LOREVENUE) - sum(LOSUPPLYCOST) as profit
                          from lineorder_all
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                               global join customer_all on LOCUSTKEY = CCUSTKEY
                               global join supplier_all on LOSUPPKEY = SSUPPKEY
                               global join part_all on LOPARTKEY = PPARTKEY
                          where CREGION = 'AMERICA' and SREGION = 'AMERICA' and (PMFGR = 'MFGR#1' or PMFGR = 'MFGR#2')
                          group by DYEAR, CNATION
                          order by DYEAR, CNATION;

SELECT
    DYEAR,
    CNATION,
    sum(LOREVENUE) - sum(LOSUPPLYCOST) AS profit
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN customer_all ON LOCUSTKEY = CCUSTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
GROUP BY
    DYEAR,
    CNATION
ORDER BY
    DYEAR ASC,
    CNATION ASC

Query id: 2f35e7c2-f208-4f66-8188-ec53940126ff

┌─DYEAR─┬─CNATION───────┬────────profit─┐
│  1992 │ ARGENTINA     │ 1041983042066 │
│  1992 │ BRAZIL        │ 1031193572794 │
│  1992 │ CANADA        │ 1032094614252 │
│  1992 │ PERU          │ 1037331491440 │
│  1992 │ UNITED STATES │ 1031593944407 │
│  1993 │ ARGENTINA     │ 1034515265588 │
│  1993 │ BRAZIL        │ 1028249774691 │
│  1993 │ CANADA        │ 1030633924190 │
│  1993 │ PERU          │ 1032888811548 │
│  1993 │ UNITED STATES │ 1030241613033 │
│  1994 │ ARGENTINA     │ 1035059347804 │
│  1994 │ BRAZIL        │ 1029788284729 │
│  1994 │ CANADA        │ 1028314868119 │
│  1994 │ PERU          │ 1025406236588 │
│  1994 │ UNITED STATES │ 1035441439980 │
│  1995 │ ARGENTINA     │ 1036878482604 │
│  1995 │ BRAZIL        │ 1032846705883 │
│  1995 │ CANADA        │ 1031488804141 │
│  1995 │ PERU          │ 1034460048487 │
│  1995 │ UNITED STATES │ 1034988860577 │
│  1996 │ ARGENTINA     │ 1041240509801 │
│  1996 │ BRAZIL        │ 1030467525021 │
│  1996 │ CANADA        │ 1035089775664 │
│  1996 │ PERU          │ 1029765730730 │
│  1996 │ UNITED STATES │ 1032384751840 │
│  1997 │ ARGENTINA     │ 1036752881505 │
│  1997 │ BRAZIL        │ 1036482571346 │
│  1997 │ CANADA        │ 1025775840777 │
│  1997 │ PERU          │ 1031380143878 │
│  1997 │ UNITED STATES │ 1030570488847 │
│  1998 │ ARGENTINA     │  607618915600 │
│  1998 │ BRAZIL        │  603999739074 │
│  1998 │ CANADA        │  601462066533 │
│  1998 │ PERU          │  603980044827 │
│  1998 │ UNITED STATES │  605069471323 │
└───────┴───────────────┴───────────────┘

35 rows in set. Elapsed: 55.995 sec. Processed 604.65 million rows, 13.23 GB (10.80 million rows/s., 236.22 MB/s.)
```

Q4.2
```
host :) select DYEAR, SNATION, PCATEGORY, sum(LOREVENUE) - sum(LOSUPPLYCOST) as profit
                          from lineorder_all
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                               global join customer_all on LOCUSTKEY = CCUSTKEY
                               global join supplier_all on LOSUPPKEY = SSUPPKEY
                               global join part_all on LOPARTKEY = PPARTKEY
                          where CREGION = 'AMERICA' and SREGION = 'AMERICA'
                            and (DYEAR = 1997 or DYEAR = 1998)
                            and (PMFGR = 'MFGR#1' or PMFGR = 'MFGR#2')
                          group by DYEAR, SNATION, PCATEGORY
                          order by DYEAR, SNATION, PCATEGORY;

SELECT
    DYEAR,
    SNATION,
    PCATEGORY,
    sum(LOREVENUE) - sum(LOSUPPLYCOST) AS profit
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN customer_all ON LOCUSTKEY = CCUSTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((DYEAR = 1997) OR (DYEAR = 1998)) AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
GROUP BY
    DYEAR,
    SNATION,
    PCATEGORY
ORDER BY
    DYEAR ASC,
    SNATION ASC,
    PCATEGORY ASC

Query id: ef233305-b3b4-4866-9bed-c3440e17c0ae

┌─DYEAR─┬─SNATION───────┬─PCATEGORY─┬───────profit─┐
│  1997 │ ARGENTINA     │ MFGR#11   │ 102369950215 │
│  1997 │ ARGENTINA     │ MFGR#12   │ 103052774082 │
│  1997 │ ARGENTINA     │ MFGR#13   │ 103202870567 │
│  1997 │ ARGENTINA     │ MFGR#14   │ 101773547534 │
│  1997 │ ARGENTINA     │ MFGR#15   │ 103728199319 │
                 ...
│  1998 │ UNITED STATES │ MFGR#22   │  59636033602 │
│  1998 │ UNITED STATES │ MFGR#23   │  61533867289 │
│  1998 │ UNITED STATES │ MFGR#24   │  60779388345 │
│  1998 │ UNITED STATES │ MFGR#25   │  60042710566 │
└───────┴───────────────┴───────────┴──────────────┘

100 rows in set. Elapsed: 12.749 sec. Processed 604.64 million rows, 13.23 GB (47.43 million rows/s., 1.04 GB/s.)
```

Q4.3
```
host :) select DYEAR, SCITY, PBRAND, sum(LOREVENUE) - sum(LOSUPPLYCOST) as profit
                          from lineorder_all
                               global join date_all on toYYYYMMDD(LOORDERDATE) = DDATEKEY
                               global join customer_all on LOCUSTKEY = CCUSTKEY
                               global join supplier_all on LOSUPPKEY = SSUPPKEY
                               global join part_all on LOPARTKEY = PPARTKEY
                          where CREGION = 'AMERICA'and SNATION = 'UNITED STATES'
                            and (DYEAR = 1997 or DYEAR = 1998)
                            and PCATEGORY = 'MFGR#14'
                          group by DYEAR, SCITY, PBRAND
                          order by DYEAR, SCITY, PBRAND;

SELECT
    DYEAR,
    SCITY,
    PBRAND,
    sum(LOREVENUE) - sum(LOSUPPLYCOST) AS profit
FROM lineorder_all
GLOBAL INNER JOIN date_all ON toYYYYMMDD(LOORDERDATE) = DDATEKEY
GLOBAL INNER JOIN customer_all ON LOCUSTKEY = CCUSTKEY
GLOBAL INNER JOIN supplier_all ON LOSUPPKEY = SSUPPKEY
GLOBAL INNER JOIN part_all ON LOPARTKEY = PPARTKEY
WHERE (CREGION = 'AMERICA') AND (SNATION = 'UNITED STATES') AND ((DYEAR = 1997) OR (DYEAR = 1998)) AND (PCATEGORY = 'MFGR#14')
GROUP BY
    DYEAR,
    SCITY,
    PBRAND
ORDER BY
    DYEAR ASC,
    SCITY ASC,
    PBRAND ASC

Query id: 7d28df7f-2a63-4279-beda-bf588899ddb8

┌─DYEAR─┬─SCITY──────┬─PBRAND────┬────profit─┐
│  1997 │ UNITED ST0 │ MFGR#141  │ 273402537 │
│  1997 │ UNITED ST0 │ MFGR#1410 │ 301190220 │
│  1997 │ UNITED ST0 │ MFGR#1411 │ 238636747 │
│  1997 │ UNITED ST0 │ MFGR#1412 │ 282770314 │
│  1997 │ UNITED ST0 │ MFGR#1413 │ 230053267 │
│  1997 │ UNITED ST0 │ MFGR#1414 │ 245407278 │
                     ...
│  1998 │ UNITED ST9 │ MFGR#146  │ 134899214 │
│  1998 │ UNITED ST9 │ MFGR#147  │ 140734743 │
│  1998 │ UNITED ST9 │ MFGR#148  │ 130780691 │
│  1998 │ UNITED ST9 │ MFGR#149  │  90067340 │
└───────┴────────────┴───────────┴───────────┘

800 rows in set. Elapsed: 10.308 sec. Processed 604.64 million rows, 13.23 GB (58.66 million rows/s., 1.28 GB/s.)
```

### 5. 单表查询

生成分布式大宽表：
```
SET max_memory_usage = 20000000000;


CREATE TABLE ssb.flat_table
ENGINE = MergeTree
PARTITION BY toYear(LOORDERDATE)
ORDER BY (LOORDERDATE, LOORDERKEY) AS
SELECT
    l.LOORDERKEY AS LOORDERKEY,
    l.LOLINENUMBER AS LOLINENUMBER,
    l.LOCUSTKEY AS LOCUSTKEY,
    l.LOPARTKEY AS LOPARTKEY,
    l.LOSUPPKEY AS LOSUPPKEY,
    l.LOORDERDATE AS LOORDERDATE,
    l.LOORDERPRIORITY AS LOORDERPRIORITY,
    l.LOSHIPPRIORITY AS LOSHIPPRIORITY,
    l.LOQUANTITY AS LOQUANTITY,
    l.LOEXTENDEDPRICE AS LOEXTENDEDPRICE,
    l.LOORDTOTALPRICE AS LOORDTOTALPRICE,
    l.LODISCOUNT AS LODISCOUNT,
    l.LOREVENUE AS LOREVENUE,
    l.LOSUPPLYCOST AS LOSUPPLYCOST,
    l.LOTAX AS LOTAX,
    l.LOCOMMITDATE AS LOCOMMITDATE,
    l.LOSHIPMODE AS LOSHIPMODE,
    c.CNAME AS CNAME,
    c.CADDRESS AS CADDRESS,
    c.CCITY AS CCITY,
    c.CNATION AS CNATION,
    c.CREGION AS CREGION,
    c.CPHONE AS CPHONE,
    c.CMKTSEGMENT AS CMKTSEGMENT,
    s.SNAME AS SNAME,
    s.SADDRESS AS SADDRESS,
    s.SCITY AS SCITY,
    s.SNATION AS SNATION,
    s.SREGION AS SREGION,
    s.SPHONE AS SPHONE,
    p.PNAME AS PNAME,
    p.PMFGR AS PMFGR,
    p.PCATEGORY AS PCATEGORY,
    p.PBRAND AS PBRAND,
    p.PCOLOR AS PCOLOR,
    p.PTYPE AS PTYPE,
    p.PSIZE AS PSIZE,
    p.PCONTAINER AS PCONTAINER
FROM ssb.lineorder_all AS l
INNER JOIN ssb.customer_all AS c ON c.CCUSTKEY = l.LOCUSTKEY
INNER JOIN ssb.supplier_all AS s ON s.SSUPPKEY = l.LOSUPPKEY
INNER JOIN ssb.part_all AS p ON p.PPARTKEY = l.LOPARTKEY;

时间：
0 rows in set. Elapsed: 2079.527 sec. Processed 610.64 million rows, 26.63 GB (293.64 thousand rows/s., 12.80 MB/s.)




host :) show create table flat_table;

SHOW CREATE TABLE flat_table

Query id: a07cd78c-bb77-46f9-bca7-44fd067197ce

┌─statement──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ CREATE TABLE ssb.flat_table
(
  `LOORDERKEY` UInt32,
  `LOLINENUMBER` UInt8,
  `LOCUSTKEY` UInt32,
  `LOPARTKEY` UInt32,
  `LOSUPPKEY` UInt32,
  `LOORDERDATE` Date,
  `LOORDERPRIORITY` LowCardinality(String),
  `LOSHIPPRIORITY` UInt8,
  `LOQUANTITY` UInt8,
  `LOEXTENDEDPRICE` UInt32,
  `LOORDTOTALPRICE` UInt32,
  `LODISCOUNT` UInt8,
  `LOREVENUE` UInt32,
  `LOSUPPLYCOST` UInt32,
  `LOTAX` UInt8,
  `LOCOMMITDATE` Date,
  `LOSHIPMODE` LowCardinality(String),
  `CNAME` String,
  `CADDRESS` String,
  `CCITY` LowCardinality(String),
  `CNATION` LowCardinality(String),
  `CREGION` LowCardinality(String),
  `CPHONE` String,
  `CMKTSEGMENT` LowCardinality(String),
  `SNAME` String,
  `SADDRESS` String,
  `SCITY` LowCardinality(String),
  `SNATION` LowCardinality(String),
  `SREGION` LowCardinality(String),
  `SPHONE` String,
  `PNAME` String,
  `PMFGR` LowCardinality(String),
  `PCATEGORY` LowCardinality(String),
  `PBRAND` LowCardinality(String),
  `PCOLOR` LowCardinality(String),
  `PTYPE` LowCardinality(String),
  `PSIZE` UInt8,
  `PCONTAINER` LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toYear(LOORDERDATE)
ORDER BY (LOORDERDATE, LOORDERKEY)
SETTINGS index_granularity = 8192 │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

1 rows in set. Elapsed: 0.004 sec. 



CREATE TABLE ssb.lineorder_flat on cluster cluster01
(
  `LOORDERKEY` UInt32,
  `LOLINENUMBER` UInt8,
  `LOCUSTKEY` UInt32,
  `LOPARTKEY` UInt32,
  `LOSUPPKEY` UInt32,
  `LOORDERDATE` Date,
  `LOORDERPRIORITY` LowCardinality(String),
  `LOSHIPPRIORITY` UInt8,
  `LOQUANTITY` UInt8,
  `LOEXTENDEDPRICE` UInt32,
  `LOORDTOTALPRICE` UInt32,
  `LODISCOUNT` UInt8,
  `LOREVENUE` UInt32,
  `LOSUPPLYCOST` UInt32,
  `LOTAX` UInt8,
  `LOCOMMITDATE` Date,
  `LOSHIPMODE` LowCardinality(String),
  `CNAME` String,
  `CADDRESS` String,
  `CCITY` LowCardinality(String),
  `CNATION` LowCardinality(String),
  `CREGION` LowCardinality(String),
  `CPHONE` String,
  `CMKTSEGMENT` LowCardinality(String),
  `SNAME` String,
  `SADDRESS` String,
  `SCITY` LowCardinality(String),
  `SNATION` LowCardinality(String),
  `SREGION` LowCardinality(String),
  `SPHONE` String,
  `PNAME` String,
  `PMFGR` LowCardinality(String),
  `PCATEGORY` LowCardinality(String),
  `PBRAND` LowCardinality(String),
  `PCOLOR` LowCardinality(String),
  `PTYPE` LowCardinality(String),
  `PSIZE` UInt8,
  `PCONTAINER` LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toYear(LOORDERDATE)
ORDER BY (LOORDERDATE, LOORDERKEY)
SETTINGS index_granularity = 8192;


CREATE TABLE ssb.lineorder_flat_all  on cluster cluster01 AS ssb.lineorder_flat 
ENGINE = Distributed(cluster01, ssb, lineorder_flat, rand());


insert into lineorder_flat_all select * from flat_table;
```

查看系统并发：
```
host :) show settings like 'max_threads';

SHOW SETTINGS LIKE 'max_threads'

Query id: 064b3a00-ae91-4052-8003-744f7c45b031

┌─name────────┬─type───────┬─value──────┐
│ max_threads │ MaxThreads │ 'auto(16)' │
└─────────────┴────────────┴────────────┘

1 rows in set. Elapsed: 0.005 sec. 
```

Q1.1
```
host :) SELECT sum(LOEXTENDEDPRICE * LODISCOUNT) AS revenue
             FROM lineorder_flat_all
             WHERE (toYear(LOORDERDATE) = 1993) AND ((LODISCOUNT >= 1) AND (LODISCOUNT <= 3)) AND (LOQUANTITY < 25);

SELECT sum(LOEXTENDEDPRICE * LODISCOUNT) AS revenue
FROM lineorder_flat_all
WHERE (toYear(LOORDERDATE) = 1993) AND ((LODISCOUNT >= 1) AND (LODISCOUNT <= 3)) AND (LOQUANTITY < 25)

Query id: b231bdb9-bc15-4079-81ca-7ede7d53b99e

┌────────revenue─┐
│ 44652567249651 │
└────────────────┘

1 rows in set. Elapsed: 0.105 sec. Processed 91.01 million rows, 728.06 MB (869.69 million rows/s., 6.96 GB/s.)
```

Q1.2
```
host :) SELECT sum(LOEXTENDEDPRICE * LODISCOUNT) AS revenue
             FROM lineorder_flat_all
             WHERE (toYYYYMM(LOORDERDATE) = 199401) AND ((LODISCOUNT >= 4) AND (LODISCOUNT <= 6)) AND ((LOQUANTITY >= 26) AND (LOQUANTITY <= 35));

SELECT sum(LOEXTENDEDPRICE * LODISCOUNT) AS revenue
FROM lineorder_flat_all
WHERE (toYYYYMM(LOORDERDATE) = 199401) AND ((LODISCOUNT >= 4) AND (LODISCOUNT <= 6)) AND ((LOQUANTITY >= 26) AND (LOQUANTITY <= 35))

Query id: d872737d-ef71-4b8f-ba82-ab157870b7ac

┌───────revenue─┐
│ 9624332170119 │
└───────────────┘

1 rows in set. Elapsed: 0.035 sec. Processed 7.75 million rows, 62.02 MB (224.22 million rows/s., 1.79 GB/s.)
```

Q1.3
```
host :) SELECT sum(LOEXTENDEDPRICE * LODISCOUNT) AS revenue
             FROM lineorder_flat_all
             WHERE (toISOWeek(LOORDERDATE) = 6) AND (toYear(LOORDERDATE) = 1994) AND ((LODISCOUNT >= 5) AND (LODISCOUNT <= 7)) AND ((LOQUANTITY >= 26) AND (LOQUANTITY <= 35));

SELECT sum(LOEXTENDEDPRICE * LODISCOUNT) AS revenue
FROM lineorder_flat_all
WHERE (toISOWeek(LOORDERDATE) = 6) AND (toYear(LOORDERDATE) = 1994) AND ((LODISCOUNT >= 5) AND (LODISCOUNT <= 7)) AND ((LOQUANTITY >= 26) AND (LOQUANTITY <= 35))

Query id: 9963fdee-fa09-433a-b784-a9887158219c

┌───────revenue─┐
│ 2611093671163 │
└───────────────┘

1 rows in set. Elapsed: 0.031 sec. Processed 1.79 million rows, 14.32 MB (57.83 million rows/s., 462.62 MB/s.)
```

Q2.1
```
host :) SELECT
               sum(LOREVENUE),
               toYear(LOORDERDATE) AS year,
               PBRAND
             FROM lineorder_flat_all
             WHERE (PCATEGORY = 'MFGR#12') AND (SREGION = 'AMERICA')
             GROUP BY
               year,
               PBRAND
             ORDER BY
               year ASC,
               PBRAND ASC;

SELECT
  sum(LOREVENUE),
  toYear(LOORDERDATE) AS year,
  PBRAND
FROM lineorder_flat_all
WHERE (PCATEGORY = 'MFGR#12') AND (SREGION = 'AMERICA')
GROUP BY
  year,
  PBRAND
ORDER BY
  year ASC,
  PBRAND ASC

Query id: c681f091-790d-4e72-b906-fc3f1fe2f558

┌─sum(LOREVENUE)─┬─year─┬─PBRAND────┐
│  64420005618 │ 1992 │ MFGR#121 │
│  63389346096 │ 1992 │ MFGR#1210 │
│  68416605637 │ 1992 │ MFGR#1211 │
            ...

│  38499217759 │ 1998 │ MFGR#127 │
│  39679892915 │ 1998 │ MFGR#128 │
│  35300513083 │ 1998 │ MFGR#129 │
└────────────────┴──────┴───────────┘

280 rows in set. Elapsed: 0.520 sec. Processed 600.04 million rows, 6.19 GB (1.15 billion rows/s., 11.91 GB/s.)
```

Q2.2
```
host :) SELECT
               sum(LOREVENUE),
               toYear(LOORDERDATE) AS year,
               PBRAND
             FROM lineorder_flat_all
             WHERE (PBRAND >= 'MFGR#2221') AND (PBRAND <= 'MFGR#2228') AND (SREGION = 'ASIA')
             GROUP BY
               year,
               PBRAND
             ORDER BY
               year ASC,
               PBRAND ASC;

SELECT
  sum(LOREVENUE),
  toYear(LOORDERDATE) AS year,
  PBRAND
FROM lineorder_flat_all
WHERE (PBRAND >= 'MFGR#2221') AND (PBRAND <= 'MFGR#2228') AND (SREGION = 'ASIA')
GROUP BY
  year,
  PBRAND
ORDER BY
  year ASC,
  PBRAND ASC

Query id: dae832aa-da11-4f8b-a718-e882bb3fe9b7

┌─sum(LOREVENUE)─┬─year─┬─PBRAND────┐
│  66450349438 │ 1992 │ MFGR#2221 │
│  65423264312 │ 1992 │ MFGR#2222 │
│  66936772687 │ 1992 │ MFGR#2223 │
│  64047191934 │ 1992 │ MFGR#2224 │
            ...

│  38328423898 │ 1998 │ MFGR#2225 │
│  38705033272 │ 1998 │ MFGR#2226 │
│  39907545239 │ 1998 │ MFGR#2227 │
│  40654201840 │ 1998 │ MFGR#2228 │
└────────────────┴──────┴───────────┘

56 rows in set. Elapsed: 0.448 sec. Processed 600.04 million rows, 5.59 GB (1.34 billion rows/s., 12.48 GB/s.)
```

Q2.3
```
host :) SELECT
               sum(LOREVENUE),
               toYear(LOORDERDATE) AS year,
               PBRAND
             FROM lineorder_flat_all
             WHERE (PBRAND = 'MFGR#2239') AND (SREGION = 'EUROPE')
             GROUP BY
               year,
               PBRAND
             ORDER BY
               year ASC,
               PBRAND ASC;

SELECT
  sum(LOREVENUE),
  toYear(LOORDERDATE) AS year,
  PBRAND
FROM lineorder_flat_all
WHERE (PBRAND = 'MFGR#2239') AND (SREGION = 'EUROPE')
GROUP BY
  year,
  PBRAND
ORDER BY
  year ASC,
  PBRAND ASC

Query id: 025f1d8f-d149-4c44-9b3e-cface3f41d54

┌─sum(LOREVENUE)─┬─year─┬─PBRAND────┐
│  65751589723 │ 1992 │ MFGR#2239 │
│  64532844801 │ 1993 │ MFGR#2239 │
│  64722599002 │ 1994 │ MFGR#2239 │
│  65616432683 │ 1995 │ MFGR#2239 │
│  64802884686 │ 1996 │ MFGR#2239 │
│  64485541165 │ 1997 │ MFGR#2239 │
│  37276536361 │ 1998 │ MFGR#2239 │
└────────────────┴──────┴───────────┘

7 rows in set. Elapsed: 0.418 sec. Processed 600.04 million rows, 5.59 GB (1.43 billion rows/s., 13.37 GB/s.)
```

Q3.1
```
host :) SELECT
               CNATION,
               SNATION,
               toYear(LOORDERDATE) AS year,
               sum(LOREVENUE) AS revenue
             FROM lineorder_flat_all
             WHERE (CREGION = 'ASIA') AND (SREGION = 'ASIA') AND (year >= 1992) AND (year <= 1997)
             GROUP BY
               CNATION,
               SNATION,
               year
             ORDER BY
               year ASC,
               revenue DESC;

SELECT
  CNATION,
  SNATION,
  toYear(LOORDERDATE) AS year,
  sum(LOREVENUE) AS revenue
FROM lineorder_flat_all
WHERE (CREGION = 'ASIA') AND (SREGION = 'ASIA') AND (year >= 1992) AND (year <= 1997)
GROUP BY
  CNATION,
  SNATION,
  year
ORDER BY
  year ASC,
  revenue DESC

Query id: 876dbbe4-430a-4ce7-ab4f-53e3818676ec

┌─CNATION───┬─SNATION───┬─year─┬──────revenue─┐
│ INDIA   │ INDIA   │ 1992 │ 537778456208 │
│ INDONESIA │ INDIA   │ 1992 │ 536684093041 │
│ VIETNAM  │ INDIA   │ 1992 │ 536483529614 │
│ INDIA   │ JAPAN   │ 1992 │ 535663357352 │
            ...

│ INDONESIA │ VIETNAM  │ 1997 │ 528663138160 │
│ JAPAN   │ CHINA   │ 1997 │ 526303212379 │
│ CHINA   │ VIETNAM  │ 1997 │ 525867745074 │
│ CHINA   │ CHINA   │ 1997 │ 525562838002 │
│ JAPAN   │ VIETNAM  │ 1997 │ 525495763677 │
└───────────┴───────────┴──────┴──────────────┘

150 rows in set. Elapsed: 0.857 sec. Processed 546.67 million rows, 5.48 GB (638.24 million rows/s., 6.39 GB/s.)
```

Q3.2
```
host :) SELECT
               CCITY,
               SCITY,
               toYear(LOORDERDATE) AS year,
               sum(LOREVENUE) AS revenue
             FROM lineorder_flat_all
             WHERE (CNATION = 'UNITED STATES') AND (SNATION = 'UNITED STATES') AND (year >= 1992) AND (year <= 1997)
             GROUP BY
               CCITY,
               SCITY,
               year
             ORDER BY
               year ASC,
               revenue DESC;

SELECT
  CCITY,
  SCITY,
  toYear(LOORDERDATE) AS year,
  sum(LOREVENUE) AS revenue
FROM lineorder_flat_all
WHERE (CNATION = 'UNITED STATES') AND (SNATION = 'UNITED STATES') AND (year >= 1992) AND (year <= 1997)
GROUP BY
  CCITY,
  SCITY,
  year
ORDER BY
  year ASC,
  revenue DESC

Query id: db4c9e88-39e5-4728-a954-8cf37a2d0ff2

┌─CCITY──────┬─SCITY──────┬─year─┬────revenue─┐
│ UNITED ST6 │ UNITED ST6 │ 1992 │ 5694246807 │
│ UNITED ST0 │ UNITED ST0 │ 1992 │ 5676049026 │
│ UNITED ST1 │ UNITED ST1 │ 1992 │ 5652630617 │
│ UNITED ST8 │ UNITED ST6 │ 1992 │ 5649039075 │
             ...
             
│ UNITED ST3 │ UNITED ST2 │ 1997 │ 4866105481 │
│ UNITED ST9 │ UNITED ST9 │ 1997 │ 4836163349 │
│ UNITED ST9 │ UNITED ST5 │ 1997 │ 4769919410 │
└────────────┴────────────┴──────┴────────────┘

600 rows in set. Elapsed: 0.518 sec. Processed 546.67 million rows, 5.57 GB (1.06 billion rows/s., 10.75 GB/s.)
```

Q3.3
```
host :) SELECT
               CCITY,
               SCITY,
               toYear(LOORDERDATE) AS year,
               sum(LOREVENUE) AS revenue
             FROM lineorder_flat_all
             WHERE ((CCITY = 'UNITED KI1') OR (CCITY = 'UNITED KI5')) AND ((SCITY = 'UNITED KI1') OR (SCITY = 'UNITED KI5')) AND (year >= 1992) AND (year <= 1997)
             GROUP BY
               CCITY,
               SCITY,
               year
             ORDER BY
               year ASC,
               revenue DESC;

SELECT
  CCITY,
  SCITY,
  toYear(LOORDERDATE) AS year,
  sum(LOREVENUE) AS revenue
FROM lineorder_flat_all
WHERE ((CCITY = 'UNITED KI1') OR (CCITY = 'UNITED KI5')) AND ((SCITY = 'UNITED KI1') OR (SCITY = 'UNITED KI5')) AND (year >= 1992) AND (year <= 1997)
GROUP BY
  CCITY,
  SCITY,
  year
ORDER BY
  year ASC,
  revenue DESC

Query id: de107a40-9573-4d8d-9528-336b10ad58c3

┌─CCITY──────┬─SCITY──────┬─year─┬────revenue─┐
│ UNITED KI1 │ UNITED KI1 │ 1992 │ 5776096629 │
│ UNITED KI5 │ UNITED KI1 │ 1992 │ 5555883901 │
│ UNITED KI5 │ UNITED KI5 │ 1992 │ 5348705805 │
│ UNITED KI1 │ UNITED KI5 │ 1992 │ 5326870427 │
│ UNITED KI1 │ UNITED KI1 │ 1993 │ 5892974670 │
│ UNITED KI1 │ UNITED KI5 │ 1993 │ 5490859451 │
│ UNITED KI5 │ UNITED KI1 │ 1993 │ 5468354303 │
│ UNITED KI5 │ UNITED KI5 │ 1993 │ 5089909647 │
│ UNITED KI5 │ UNITED KI1 │ 1994 │ 5437315108 │
│ UNITED KI1 │ UNITED KI1 │ 1994 │ 5348775917 │
│ UNITED KI5 │ UNITED KI5 │ 1994 │ 5310936695 │
│ UNITED KI1 │ UNITED KI5 │ 1994 │ 5237461110 │
│ UNITED KI1 │ UNITED KI1 │ 1995 │ 5737551920 │
│ UNITED KI5 │ UNITED KI5 │ 1995 │ 5657584590 │
│ UNITED KI5 │ UNITED KI1 │ 1995 │ 5260093556 │
│ UNITED KI1 │ UNITED KI5 │ 1995 │ 5213763257 │
│ UNITED KI5 │ UNITED KI1 │ 1996 │ 5522325005 │
│ UNITED KI1 │ UNITED KI1 │ 1996 │ 5451244409 │
│ UNITED KI5 │ UNITED KI5 │ 1996 │ 5231759057 │
│ UNITED KI1 │ UNITED KI5 │ 1996 │ 5203962897 │
│ UNITED KI1 │ UNITED KI1 │ 1997 │ 5340760807 │
│ UNITED KI5 │ UNITED KI1 │ 1997 │ 5295685214 │
│ UNITED KI1 │ UNITED KI5 │ 1997 │ 5188428156 │
│ UNITED KI5 │ UNITED KI5 │ 1997 │ 5024634475 │
└────────────┴────────────┴──────┴────────────┘

24 rows in set. Elapsed: 0.480 sec. Processed 546.67 million rows, 4.47 GB (1.14 billion rows/s., 9.30 GB/s.)
```

Q3.4
```
host :) SELECT
               CCITY,
               SCITY,
               toYear(LOORDERDATE) AS year,
               sum(LOREVENUE) AS revenue
             FROM lineorder_flat_all
             WHERE ((CCITY = 'UNITED KI1') OR (CCITY = 'UNITED KI5')) AND ((SCITY = 'UNITED KI1') OR (SCITY = 'UNITED KI5')) AND (toYYYYMM(LOORDERDATE) = 199712)
             GROUP BY
               CCITY,
               SCITY,
               year
             ORDER BY
               year ASC,
               revenue DESC;

SELECT
  CCITY,
  SCITY,
  toYear(LOORDERDATE) AS year,
  sum(LOREVENUE) AS revenue
FROM lineorder_flat_all
WHERE ((CCITY = 'UNITED KI1') OR (CCITY = 'UNITED KI5')) AND ((SCITY = 'UNITED KI1') OR (SCITY = 'UNITED KI5')) AND (toYYYYMM(LOORDERDATE) = 199712)
GROUP BY
  CCITY,
  SCITY,
  year
ORDER BY
  year ASC,
  revenue DESC

Query id: a2423d74-be9e-4eeb-9c98-212f145fd311

┌─CCITY──────┬─SCITY──────┬─year─┬───revenue─┐
│ UNITED KI1 │ UNITED KI1 │ 1997 │ 481119563 │
│ UNITED KI5 │ UNITED KI5 │ 1997 │ 386477033 │
│ UNITED KI5 │ UNITED KI1 │ 1997 │ 378048353 │
│ UNITED KI1 │ UNITED KI5 │ 1997 │ 366630529 │
└────────────┴────────────┴──────┴───────────┘

4 rows in set. Elapsed: 0.036 sec. Processed 7.80 million rows, 63.69 MB (218.75 million rows/s., 1.79 GB/s.)
```

Q4.1
```
host :) SELECT
               toYear(LOORDERDATE) AS year,
               CNATION,
               sum(LOREVENUE - LOSUPPLYCOST) AS profit
             FROM lineorder_flat_all
             WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
             GROUP BY
               year,
               CNATION
             ORDER BY
               year ASC,
               CNATION ASC;

SELECT
  toYear(LOORDERDATE) AS year,
  CNATION,
  sum(LOREVENUE - LOSUPPLYCOST) AS profit
FROM lineorder_flat_all
WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
GROUP BY
  year,
  CNATION
ORDER BY
  year ASC,
  CNATION ASC

Query id: f32e37c7-51fc-48f4-8f1f-3ef7516a6274

┌─year─┬─CNATION───────┬────────profit─┐
│ 1992 │ ARGENTINA   │ 1041983042066 │
│ 1992 │ BRAZIL    │ 1031193572794 │
│ 1992 │ CANADA    │ 1032094614252 │
│ 1992 │ PERU     │ 1037331491440 │
│ 1992 │ UNITED STATES │ 1031593944407 │
          ...

│ 1998 │ BRAZIL    │ 603999739074 │
│ 1998 │ CANADA    │ 601462066533 │
│ 1998 │ PERU     │ 603980044827 │
│ 1998 │ UNITED STATES │ 605069471323 │
└──────┴───────────────┴───────────────┘

35 rows in set. Elapsed: 0.784 sec. Processed 600.04 million rows, 8.41 GB (765.08 million rows/s., 10.72 GB/s.)
```

Q4.2
```
host :) SELECT
               toYear(LOORDERDATE) AS year,
               SNATION,
               PCATEGORY,
               sum(LOREVENUE - LOSUPPLYCOST) AS profit
             FROM lineorder_flat_all
             WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((year = 1997) OR (year = 1998)) AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
             GROUP BY
               year,
               SNATION,
               PCATEGORY
             ORDER BY
               year ASC,
               SNATION ASC,
               PCATEGORY ASC;

SELECT
  toYear(LOORDERDATE) AS year,
  SNATION,
  PCATEGORY,
  sum(LOREVENUE - LOSUPPLYCOST) AS profit
FROM lineorder_flat_all
WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((year = 1997) OR (year = 1998)) AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
GROUP BY
  year,
  SNATION,
  PCATEGORY
ORDER BY
  year ASC,
  SNATION ASC,
  PCATEGORY ASC

Query id: a8bc1bd0-aff7-437b-8990-f2b7eee5c600

┌─year─┬─SNATION───────┬─PCATEGORY─┬───────profit─┐
│ 1997 │ ARGENTINA   │ MFGR#11  │ 102369950215 │
│ 1997 │ ARGENTINA   │ MFGR#12  │ 103052774082 │
│ 1997 │ ARGENTINA   │ MFGR#13  │ 103202870567 │
│ 1997 │ ARGENTINA   │ MFGR#14  │ 101773547534 │
│ 1997 │ ARGENTINA   │ MFGR#15  │ 103728199319 │
│ 1997 │ ARGENTINA   │ MFGR#21  │ 101209234002 │
               ...
               
│ 1998 │ UNITED STATES │ MFGR#23  │ 61533867289 │
│ 1998 │ UNITED STATES │ MFGR#24  │ 60779388345 │
│ 1998 │ UNITED STATES │ MFGR#25  │ 60042710566 │
└──────┴───────────────┴───────────┴──────────────┘

100 rows in set. Elapsed: 0.322 sec. Processed 144.42 million rows, 2.17 GB (449.12 million rows/s., 6.75 GB/s.)
```

Q4.3
```
host :) SELECT
               toYear(LOORDERDATE) AS year,
               SCITY,
               PBRAND,
               sum(LOREVENUE - LOSUPPLYCOST) AS profit
             FROM lineorder_flat_all
             WHERE (SNATION = 'UNITED STATES') AND ((year = 1997) OR (year = 1998)) AND (PCATEGORY = 'MFGR#14')
             GROUP BY
               year,
               SCITY,
               PBRAND
             ORDER BY
               year ASC,
               SCITY ASC,
               PBRAND ASC;

SELECT
  toYear(LOORDERDATE) AS year,
  SCITY,
  PBRAND,
  sum(LOREVENUE - LOSUPPLYCOST) AS profit
FROM lineorder_flat_all
WHERE (SNATION = 'UNITED STATES') AND ((year = 1997) OR (year = 1998)) AND (PCATEGORY = 'MFGR#14')
GROUP BY
  year,
  SCITY,
  PBRAND
ORDER BY
  year ASC,
  SCITY ASC,
  PBRAND ASC

Query id: fccbd079-8b9a-4dea-81de-e9e39fe68090

┌─year─┬─SCITY──────┬─PBRAND────┬─────profit─┐
│ 1997 │ UNITED ST0 │ MFGR#141 │ 1402715668 │
│ 1997 │ UNITED ST0 │ MFGR#1410 │ 1309800423 │
│ 1997 │ UNITED ST0 │ MFGR#1411 │ 1252501939 │
│ 1997 │ UNITED ST0 │ MFGR#1412 │ 1295391924 │
                ...

│ 1998 │ UNITED ST9 │ MFGR#147 │ 640908455 │
│ 1998 │ UNITED ST9 │ MFGR#148 │ 811919859 │
│ 1998 │ UNITED ST9 │ MFGR#149 │ 603099066 │
└──────┴────────────┴───────────┴────────────┘

800 rows in set. Elapsed: 0.266 sec. Processed 144.42 million rows, 2.24 GB (543.01 million rows/s., 8.41 GB/s.)
```

### 6. 并发测试
Q4.1
```
SELECT
  toYear(LOORDERDATE) AS year,
  CNATION,
  sum(LOREVENUE - LOSUPPLYCOST) AS profit
FROM lineorder_flat_all
WHERE (CREGION = 'AMERICA') AND (SREGION = 'AMERICA') AND ((PMFGR = 'MFGR#1') OR (PMFGR = 'MFGR#2'))
GROUP BY
  year,
  CNATION
ORDER BY
  year ASC,
  CNATION ASC;
```

```
并发数：1
[weizuo@PC ClickHouse]$ ./clickhouse benchmark --host=host --port=9000 --user=xxx --password=xxx --database=ssb --concurrency=1 --iterations=1 < /home/weizuo/test-ck.sql
Loaded 1 queries.

Queries executed: 1.

host:9000, queries 1, QPS: 1.219, RPS: 731564655.275, MiB/s: 9775.986, result RPS: 42.672, result MiB/s: 0.001.

0.000%                0.820 sec.        
10.000%                0.820 sec.        
20.000%                0.820 sec.        
30.000%                0.820 sec.        
40.000%                0.820 sec.        
50.000%                0.820 sec.        
60.000%                0.820 sec.        
70.000%                0.820 sec.        
80.000%                0.820 sec.        
90.000%                0.820 sec.        
95.000%                0.820 sec.        
99.000%                0.820 sec.        
99.900%                0.820 sec.        
99.990%                0.820 sec.


并发数：10
[weizuo@PC ClickHouse]$ ./clickhouse benchmark --host=host --port=9000 --user=xxxx --password=xxxx --database=ssb --concurrency=10 --iterations=10 < /home/weizuo/test-ck.sql
Loaded 1 queries.

Queries executed: 10.

host:9000, queries 10, QPS: 2.286, RPS: 1371408144.333, MiB/s: 18326.290, result RPS: 79.994, result MiB/s: 0.001.

0.000%                4.297 sec.        
10.000%                4.366 sec.        
20.000%                4.373 sec.        
30.000%                4.382 sec.        
40.000%                4.385 sec.        
50.000%                4.386 sec.        
60.000%                4.386 sec.        
70.000%                4.389 sec.        
80.000%                4.390 sec.        
90.000%                4.392 sec.        
95.000%                4.393 sec.        
99.000%                4.393 sec.        
99.900%                4.393 sec.        
99.990%                4.393 sec.



并发数：20
[weizuo@PC ClickHouse]$ ./clickhouse benchmark --host=host --port=9000 --user=xxx --password=xxx --database=ssb --concurrency=20 --iterations=20 < /home/weizuo/test-ck.sql
Loaded 1 queries.

Queries executed: 20.

host:9000, queries 20, QPS: 2.247, RPS: 1348011911.681, MiB/s: 18013.644, result RPS: 78.629, result MiB/s: 0.001.

0.000%                8.014 sec.        
10.000%                8.671 sec.        
20.000%                8.793 sec.        
30.000%                8.823 sec.        
40.000%                8.959 sec.        
50.000%                8.982 sec.        
60.000%                8.985 sec.        
70.000%                9.053 sec.        
80.000%                9.063 sec.        
90.000%                9.127 sec.        
95.000%                9.129 sec.        
99.000%                9.210 sec.        
99.900%                9.210 sec.        
99.990%                9.210 sec.  

 
 
 并发数：30
 [weizuo@PC ClickHouse]$ ./clickhouse benchmark --host=host --port=9000 --user=xxx --password=xxxx --database=ssb --concurrency=30 --iterations=30 < /home/weizuo/test-ck.sql
Loaded 1 queries.

Queries executed: 30.

host:9000, queries 30, QPS: 2.329, RPS: 1397284276.805, MiB/s: 18672.076, result RPS: 81.503, result MiB/s: 0.001.

0.000%                11.942 sec.        
10.000%                12.565 sec.        
20.000%                12.651 sec.        
30.000%                12.761 sec.        
40.000%                12.870 sec.        
50.000%                12.898 sec.        
60.000%                12.904 sec.        
70.000%                12.973 sec.        
80.000%                13.048 sec.        
90.000%                13.289 sec.        
95.000%                13.554 sec.        
99.000%                13.580 sec.        
99.900%                13.580 sec.        
99.990%                13.580 sec.      


并发数：50
[weizuo@PC ClickHouse]$ ./clickhouse benchmark --host=host --port=9000 --user=xxx --password=xxx --database=ssb --concurrency=50 --iterations=50 < /home/weizuo/test-ck.sql
Loaded 1 queries.

Queries executed: 50.

host:9000, queries 50, QPS: 2.513, RPS: 1508176983.836, MiB/s: 20153.949, result RPS: 87.971, result MiB/s: 0.001.

0.000%                17.935 sec.        
10.000%                18.492 sec.        
20.000%                18.642 sec.        
30.000%                18.711 sec.        
40.000%                19.131 sec.        
50.000%                19.350 sec.        
60.000%                19.735 sec.        
70.000%                20.100 sec.        
80.000%                21.456 sec.        
90.000%                22.507 sec.        
95.000%                22.845 sec.        
99.000%                22.892 sec.        
99.900%                22.892 sec.        
99.990%                22.892 sec.


并发数：80
[weizuo@PC ClickHouse]$ ./clickhouse benchmark --host=host --port=9000 --user=xxx --password=xxx --database=ssb --concurrency=80 --iterations=80 < /home/weizuo/test-ck.sql
Loaded 1 queries.

Queries executed: 80.

host:9000, queries 80, QPS: 2.379, RPS: 1427479913.215, MiB/s: 19075.584, result RPS: 83.264, result MiB/s: 0.001.

0.000%                29.276 sec.        
10.000%                30.861 sec.        
20.000%                31.820 sec.        
30.000%                32.307 sec.        
40.000%                32.569 sec.        
50.000%                32.983 sec.        
60.000%                33.275 sec.        
70.000%                34.012 sec.        
80.000%                36.055 sec.        
90.000%                38.144 sec.        
95.000%                38.331 sec.        
99.000%                38.356 sec.        
99.900%                38.378 sec.        
99.990%                38.378 sec.        


并发数：100
[weizuo@PC ClickHouse]$ ./clickhouse benchmark --host=host --port=9000 --user=xxx --password=xxx--database=ssb --concurrency=100 --iterations=100 < /home/weizuo/test-ck.sql
Loaded 1 queries.

Queries executed: 100.

host:9000, queries 100, QPS: 2.304, RPS: 1382268748.568, MiB/s: 18471.422, result RPS: 80.627, result MiB/s: 0.001.

0.000%                37.619 sec.        
10.000%                39.425 sec.        
20.000%                41.509 sec.        
30.000%                42.049 sec.        
40.000%                42.357 sec.        
50.000%                42.655 sec.        
60.000%                42.988 sec.        
70.000%                45.166 sec.        
80.000%                46.268 sec.        
90.000%                48.397 sec.        
95.000%                48.865 sec.        
99.000%                48.887 sec.        
99.900%                48.887 sec.        
99.990%                48.887 sec.        
```