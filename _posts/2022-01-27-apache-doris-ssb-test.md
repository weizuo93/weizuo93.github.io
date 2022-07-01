---
layout: post
title: 基于SSB的Apache Doris性能测试
date: 2022-01-27 20:30:32 +0800
categories: [Apache Doris]
---

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

数据格式转化`data_format_convert.py` :
```
def data_format_convert(source, dest):                           
    sf = open(source, "r")
    df = open(dest, "w")
    sline = sf.readline()
    while sline:
        dline = ""
        count = 0 
        i = 0 

        for ch in sline:
            i = i + 1 
            if ch == "\"":
                count = count + 1 
                continue

            if ch == "," and count % 2 == 0 and i == len(sline) - 1:
                continue

            if ch == "," and count % 2 == 0:
                dline += "|" 
                continue

            dline += ch

        df.write(dline)

        sline = sf.readline()

    sf.close()
    df.close()


if __name__ == "__main__":
    data_format_convert("customer.tbl", "customer-doris.tbl")
    data_format_convert("date.tbl", "date-doris.tbl")
    data_format_convert("part.tbl", "part-doris.tbl")
    data_format_convert("supplier.tbl", "supplier-doris.tbl")
    data_format_convert("lineorder.tbl", "lineorder-doris.tbl")
```

### 2. 建表

lineorder表
```
CREATE TABLE IF NOT EXISTS `lineorder`(
  `lo_orderkey` int(11) NOT NULL COMMENT "",
  `lo_linenumber` int(11) NOT NULL COMMENT "",
  `lo_custkey` int(11) NOT NULL COMMENT "",
  `lo_partkey` int(11) NOT NULL COMMENT "",
  `lo_suppkey` int(11) NOT NULL COMMENT "",
  `lo_orderdate` date NOT NULL COMMENT "",
  `lo_orderpriority` varchar(16) NOT NULL COMMENT "",
  `lo_shippriority` int(11) NOT NULL COMMENT "",
  `lo_quantity` int(11) NOT NULL COMMENT "",
  `lo_extendedprice` int(11) NOT NULL COMMENT "",
  `lo_ordtotalprice` int(11) NOT NULL COMMENT "",
  `lo_discount` int(11) NOT NULL COMMENT "",
  `lo_revenue` int(11) NOT NULL COMMENT "",
  `lo_supplycost` int(11) NOT NULL COMMENT "",
  `lo_tax` int(11) NOT NULL COMMENT "",
  `lo_commitdate` date NOT NULL COMMENT "",
  `lo_shipmode` varchar(11) NOT NULL COMMENT ""
) ENGINE=OLAP
DUPLICATE KEY(`lo_orderkey`)
COMMENT "OLAP"
DISTRIBUTED BY HASH(`lo_orderkey`) BUCKETS 192
PROPERTIES (
"replication_num" = "1",
"in_memory" = "false",
"storage_format" = "V2"
);
```

customer表
```
CREATE TABLE IF NOT EXISTS `customer` (
  `c_custkey` int(11) NOT NULL COMMENT "",
  `c_name` varchar(26) NOT NULL COMMENT "",
  `c_address` varchar(41) NOT NULL COMMENT "",
  `c_city` varchar(11) NOT NULL COMMENT "",
  `c_nation` varchar(16) NOT NULL COMMENT "", 
  `c_region` varchar(13) NOT NULL COMMENT "",
  `c_phone` varchar(16) NOT NULL COMMENT "",
  `c_mktsegment` varchar(11) NOT NULL COMMENT ""
) ENGINE=OLAP
DUPLICATE KEY(`c_custkey`)
COMMENT "OLAP"
DISTRIBUTED BY HASH(`c_custkey`) BUCKETS 12
PROPERTIES (
"replication_num" = "1",
"in_memory" = "false",
"storage_format" = "V2"
);
```

supplier表
```
 CREATE TABLE IF NOT EXISTS `supplier` (
  `s_suppkey` int(11) NOT NULL COMMENT "",
  `s_name` varchar(26) NOT NULL COMMENT "",
  `s_address` varchar(26) NOT NULL COMMENT "",
  `s_city` varchar(11) NOT NULL COMMENT "",
  `s_nation` varchar(16) NOT NULL COMMENT "",
  `s_region` varchar(13) NOT NULL COMMENT "",
  `s_phone` varchar(16) NOT NULL COMMENT ""
) ENGINE=OLAP
DUPLICATE KEY(`s_suppkey`)
COMMENT "OLAP"
DISTRIBUTED BY HASH(`s_suppkey`) BUCKETS 12
PROPERTIES ( 
"replication_num" = "1",
"in_memory" = "false",
"storage_format" = "V2"
);
```

part表
```
CREATE TABLE IF NOT EXISTS `part` (
  `p_partkey` int(11) NOT NULL COMMENT "",
  `p_name` varchar(23) NOT NULL COMMENT "",
  `p_mfgr` varchar(7) NOT NULL COMMENT "",
  `p_category` varchar(8) NOT NULL COMMENT "",
  `p_brand` varchar(10) NOT NULL COMMENT "",
  `p_color` varchar(12) NOT NULL COMMENT "",
  `p_type` varchar(26) NOT NULL COMMENT "",
  `p_size` int(11) NOT NULL COMMENT "",
  `p_container` varchar(11) NOT NULL COMMENT ""
) ENGINE=OLAP
DUPLICATE KEY(`p_partkey`)
COMMENT "OLAP"
DISTRIBUTED BY HASH(`p_partkey`) BUCKETS 12
PROPERTIES (
"replication_num" = "1",
"in_memory" = "false",
"storage_format" = "V2"
); 
```

lineorder_flat表
```
CREATE TABLE `lineorder_flat` (
 `LO_ORDERDATE` date NOT NULL COMMENT "", 
 `LO_ORDERKEY` int(11) NOT NULL COMMENT "", 
 `LO_LINENUMBER` tinyint(4) NOT NULL COMMENT "", 
 `LO_CUSTKEY` int(11) NOT NULL COMMENT "", 
 `LO_PARTKEY` int(11) NOT NULL COMMENT "", 
 `LO_SUPPKEY` int(11) NOT NULL COMMENT "", 
 `LO_ORDERPRIORITY` varchar(100) NOT NULL COMMENT "", 
 `LO_SHIPPRIORITY` tinyint(4) NOT NULL COMMENT "", 
 `LO_QUANTITY` tinyint(4) NOT NULL COMMENT "", 
 `LO_EXTENDEDPRICE` int(11) NOT NULL COMMENT "", 
 `LO_ORDTOTALPRICE` int(11) NOT NULL COMMENT "", 
 `LO_DISCOUNT` tinyint(4) NOT NULL COMMENT "", 
 `LO_REVENUE` int(11) NOT NULL COMMENT "", 
 `LO_SUPPLYCOST` int(11) NOT NULL COMMENT "", 
 `LO_TAX` tinyint(4) NOT NULL COMMENT "", 
 `LO_COMMITDATE` date NOT NULL COMMENT "", 
 `LO_SHIPMODE` varchar(100) NOT NULL COMMENT "", 
 `C_NAME` varchar(100) NOT NULL COMMENT "", 
 `C_ADDRESS` varchar(100) NOT NULL COMMENT "", 
 `C_CITY` varchar(100) NOT NULL COMMENT "", 
 `C_NATION` varchar(100) NOT NULL COMMENT "", 
 `C_REGION` varchar(100) NOT NULL COMMENT "", 
 `C_PHONE` varchar(100) NOT NULL COMMENT "", 
 `C_MKTSEGMENT` varchar(100) NOT NULL COMMENT "", 
 `S_NAME` varchar(100) NOT NULL COMMENT "", 
 `S_ADDRESS` varchar(100) NOT NULL COMMENT "", 
 `S_CITY` varchar(100) NOT NULL COMMENT "", 
 `S_NATION` varchar(100) NOT NULL COMMENT "", 
 `S_REGION` varchar(100) NOT NULL COMMENT "", 
 `S_PHONE` varchar(100) NOT NULL COMMENT "", 
 `P_NAME` varchar(100) NOT NULL COMMENT "", 
 `P_MFGR` varchar(100) NOT NULL COMMENT "", 
 `P_CATEGORY` varchar(100) NOT NULL COMMENT "", 
 `P_BRAND` varchar(100) NOT NULL COMMENT "", 
 `P_COLOR` varchar(100) NOT NULL COMMENT "", 
 `P_TYPE` varchar(100) NOT NULL COMMENT "", 
 `P_SIZE` tinyint(4) NOT NULL COMMENT "", 
 `P_CONTAINER` varchar(100) NOT NULL COMMENT ""
) ENGINE=OLAP
DUPLICATE KEY(`LO_ORDERDATE`, `LO_ORDERKEY`)
COMMENT "OLAP"
PARTITION BY RANGE(`LO_ORDERDATE`)
(PARTITION p1 VALUES [('0000-01-01'), ('1993-01-01')),
PARTITION p2 VALUES [('1993-01-01'), ('1994-01-01')),
PARTITION p3 VALUES [('1994-01-01'), ('1995-01-01')),
PARTITION p4 VALUES [('1995-01-01'), ('1996-01-01')),
PARTITION p5 VALUES [('1996-01-01'), ('1997-01-01')),
PARTITION p6 VALUES [('1997-01-01'), ('1998-01-01')),
PARTITION p7 VALUES [('1998-01-01'), ('1999-01-01')))
DISTRIBUTED BY HASH(`LO_ORDERKEY`) BUCKETS 48
PROPERTIES (
"replication_num" = "1",
"in_memory" = "false",
"storage_format" = "V2"
);
```

dates表
```
CREATE TABLE IF NOT EXISTS `dates` (
 `d_datekey` int(11) NOT NULL COMMENT "", 
 `d_date` varchar(20) NOT NULL COMMENT "", 
 `d_dayofweek` varchar(10) NOT NULL COMMENT "", 
 `d_month` varchar(11) NOT NULL COMMENT "", 
 `d_year` int(11) NOT NULL COMMENT "", 
 `d_yearmonthnum` int(11) NOT NULL COMMENT "", 
 `d_yearmonth` varchar(9) NOT NULL COMMENT "", 
 `d_daynuminweek` int(11) NOT NULL COMMENT "", 
 `d_daynuminmonth` int(11) NOT NULL COMMENT "", 
 `d_daynuminyear` int(11) NOT NULL COMMENT "", 
 `d_monthnuminyear` int(11) NOT NULL COMMENT "", 
 `d_weeknuminyear` int(11) NOT NULL COMMENT "", 
 `d_sellingseason` varchar(14) NOT NULL COMMENT "", 
 `d_lastdayinweekfl` int(11) NOT NULL COMMENT "", 
 `d_lastdayinmonthfl` int(11) NOT NULL COMMENT "", 
 `d_holidayfl` int(11) NOT NULL COMMENT "", 
 `d_weekdayfl` int(11) NOT NULL COMMENT ""
) ENGINE=OLAP
DUPLICATE KEY(`d_datekey`)
COMMENT "OLAP"
DISTRIBUTED BY HASH(`d_datekey`) BUCKETS 1
PROPERTIES (
"replication_num" = "1",
"in_memory" = "false", 
"storage_format" = "V2"
);
```

### 3. 数据导入

导入数据到customer表：
```
[weizuo@PC ssb-dbgen]$ curl --location-trusted -u xxx:xxx -H "label:load_label_customer" -H "column_separator:|" -T ./customer-doris.tbl http://host:8030/api/ssb_dbgen/customer/_stream_load;
{
    "TxnId": 1040,
    "Label": "load_label_customer",
    "Status": "Success",
    "Message": "OK",
    "NumberTotalRows": 3000000,
    "NumberLoadedRows": 3000000,
    "NumberFilteredRows": 0,
    "NumberUnselectedRows": 0,
    "LoadBytes": 286529327,
    "LoadTimeMs": 10510,
    "BeginTxnTimeMs": 1,
    "StreamLoadPutTimeMs": 3,
    "ReadDataTimeMs": 7076,
    "WriteDataTimeMs": 10448,
    "CommitAndPublishTimeMs": 56
}
```

导入数据到part表：
```
[weizuo@PC ssb-dbgen]$ curl --location-trusted -u xxx:xxx -H "label:load_label_part" -H "column_separator:|" -T ./part-doris.tbl http://host:8030/api/ssb_dbgen/part/_stream_load;
{
    "TxnId": 1041,
    "Label": "load_label_part",
    "Status": "Success",
    "Message": "OK",
    "NumberTotalRows": 1400000,
    "NumberLoadedRows": 1400000,
    "NumberFilteredRows": 0,
    "NumberUnselectedRows": 0,
    "LoadBytes": 119642413,
    "LoadTimeMs": 4843,
    "BeginTxnTimeMs": 0,
    "StreamLoadPutTimeMs": 1,
    "ReadDataTimeMs": 3215,
    "WriteDataTimeMs": 4781,
    "CommitAndPublishTimeMs": 59
}
```

导入数据到supplier表：
```
[weizuo@PC ssb-dbgen]$ curl --location-trusted -u xxx:xxx -H "label:load_label_supplier" -H "column_separator:|" -T ./supplier-doris.tbl http://host:8030/api/ssb_dbgen/supplier/_stream_load;
{
    "TxnId": 1042,
    "Label": "load_label_supplier",
    "Status": "Success",
    "Message": "OK",
    "NumberTotalRows": 200000,
    "NumberLoadedRows": 200000,
    "NumberFilteredRows": 0,
    "NumberUnselectedRows": 0,
    "LoadBytes": 16862852,
    "LoadTimeMs": 821,
    "BeginTxnTimeMs": 0,
    "StreamLoadPutTimeMs": 1,
    "ReadDataTimeMs": 157,
    "WriteDataTimeMs": 751,
    "CommitAndPublishTimeMs": 67
}
```

导入数据到lineorder表：
```
[weizuo@PC data_dir]$ curl --location-trusted -u xxx:xxx -H "label:load_label_lineorder" -H "column_separator:|" -T ./lineorder-doris.tbl http://host:8030/api/ssb_gen/lineorder/_stream_load;
{
  "TxnId": 1053,
  "Label": "load_label_lineorder",
  "Status": "Success",
  "Message": "OK",
  "NumberTotalRows": 600037902,
  "NumberLoadedRows": 600037902,
  "NumberFilteredRows": 0,
  "NumberUnselectedRows": 0,
  "LoadBytes": 65569348218,
  "LoadTimeMs": 2019999,
  "BeginTxnTimeMs": 1,
  "StreamLoadPutTimeMs": 3,
  "ReadDataTimeMs": 1904038,
  "WriteDataTimeMs": 2019964,
  "CommitAndPublishTimeMs": 30
}
```

导入数据到dates表：
```
[weizuo@PC data_dir]$ curl --location-trusted -u xxx:xxx -H "label:load_label_dates" -H "column_separator:|" -T ./dates-doris.tbl http://host:8030/api/ssb_dbgen/dates/_stream_load;
{
    "TxnId": 1054,
    "Label": "load_label_dates",
    "Status": "Success",
    "Message": "OK",
    "NumberTotalRows": 2556,
    "NumberLoadedRows": 2556,
    "NumberFilteredRows": 0,
    "NumberUnselectedRows": 0,
    "LoadBytes": 227409,
    "LoadTimeMs": 149,
    "BeginTxnTimeMs": 1,
    "StreamLoadPutTimeMs": 2,
    "ReadDataTimeMs": 0,
    "WriteDataTimeMs": 119,
    "CommitAndPublishTimeMs": 25
}
```

### 4. 多表Join查询

设置并发
```
set parallel_fragment_exec_instance_num=16;
```

Q1.1
```
MySQL [ssb_dbgen]> select sum(lo_revenue) as revenue from lineorder join dates on lo_orderdate = d_datekey where d_year = 1993 and lo_discount between 1 and 3 and lo_quantity < 25;
+----------------+
| revenue        |
+----------------+
| 21881848590256 |
+----------------+
1 row in set (0.67 sec)
```

Q1.2
```
MySQL [ssb_dbgen]> select sum(lo_revenue) as revenue from lineorder join dates on lo_orderdate = d_datekey where d_yearmonthnum = 199401 and lo_discount between 4 and 6 and lo_quantity between 26 and 35;
+---------------+
| revenue       |
+---------------+
| 1829705342413 |
+---------------+
1 row in set (0.39 sec)
```

Q1.3
```
MySQL [ssb_dbgen]> select sum(lo_revenue) as revenue from lineorder join dates on lo_orderdate = d_datekey where d_weeknuminyear = 6 and d_year = 1994 and lo_discount between 5 and 7 and lo_quantity between 26 and 35;
+--------------+
| revenue      |
+--------------+
| 407995993835 |
+--------------+
1 row in set (0.38 sec)
```

Q2.1
```
MySQL [ssb_dbgen]> select sum(lo_revenue) as lo_revenue, d_year, p_brand from lineorder join dates on lo_orderdate = d_datekey join part on lo_partkey = p_partkey join supplier on lo_suppkey = s_suppkey where p_category = 'MFGR#12' and s_region = 'AMERICA' group by d_year, p_brand order by d_year, p_brand;
+-------------+--------+-----------+
| lo_revenue  | d_year | p_brand   |
+-------------+--------+-----------+
| 64420005618 |   1992 | MFGR#121  |
| 63389346096 |   1992 | MFGR#1210 |
| 68416605637 |   1992 | MFGR#1211 |
| 64135723264 |   1992 | MFGR#1212 |
            ...
| 38499217759 |   1998 | MFGR#127  |
| 39679892915 |   1998 | MFGR#128  |
| 35300513083 |   1998 | MFGR#129  |
+-------------+--------+-----------+
280 rows in set (10.85 sec)
```

Q2.2
```
MySQL [ssb_dbgen]> select sum(lo_revenue) as lo_revenue, d_year, p_brand from lineorder join dates on lo_orderdate = d_datekey join part on lo_partkey = p_partkey join supplier on lo_suppkey = s_suppkey where p_brand between 'MFGR#2221' and 'MFGR#2228' and s_region = 'ASIA' group by d_year, p_brand order by d_year, p_brand;
+-------------+--------+-----------+
| lo_revenue  | d_year | p_brand   |
+-------------+--------+-----------+
| 66450349438 |   1992 | MFGR#2221 |
| 65423264312 |   1992 | MFGR#2222 |
| 66936772687 |   1992 | MFGR#2223 |
               ...
| 38705033272 |   1998 | MFGR#2226 |
| 39907545239 |   1998 | MFGR#2227 |
| 40654201840 |   1998 | MFGR#2228 |
+-------------+--------+-----------+
56 rows in set (9.45 sec) 
```

Q2.3
```
MySQL [ssb_dbgen]> select sum(lo_revenue) as lo_revenue, d_year, p_brand from lineorder join dates on lo_orderdate = d_datekey join part on lo_partkey = p_partkey join supplier on lo_suppkey = s_suppkey where p_brand = 'MFGR#2239' and s_region = 'EUROPE' group by d_year, p_brand order by d_year, p_brand;
+-------------+--------+-----------+
| lo_revenue  | d_year | p_brand   |
+-------------+--------+-----------+
| 65751589723 |   1992 | MFGR#2239 |
| 64532844801 |   1993 | MFGR#2239 |
| 64722599002 |   1994 | MFGR#2239 |
| 65616432683 |   1995 | MFGR#2239 |
| 64802884686 |   1996 | MFGR#2239 |
| 64485541165 |   1997 | MFGR#2239 |
| 37276536361 |   1998 | MFGR#2239 |
+-------------+--------+-----------+
7 rows in set (9.59 sec)
```

Q3.1
```
MySQL [ssb_dbgen]> select c_nation, s_nation, d_year, sum(lo_revenue) as lo_revenue from lineorder join dates on lo_orderdate = d_datekey join customer on lo_custkey = c_custkey join supplier on lo_suppkey = s_suppkey where c_region = 'ASIA' and s_region = 'ASIA'and d_year >= 1992 and d_year <= 1997 group by c_nation, s_nation, d_year order by d_year asc, lo_revenue desc;
+-----------+-----------+--------+--------------+
| c_nation  | s_nation  | d_year | lo_revenue   |
+-----------+-----------+--------+--------------+
| INDIA     | INDIA     |   1992 | 537778456208 |
| INDONESIA | INDIA     |   1992 | 536684093041 |
| VIETNAM   | INDIA     |   1992 | 536483529614 |
                 ...
| JAPAN     | CHINA     |   1997 | 526303212379 |
| CHINA     | VIETNAM   |   1997 | 525867745074 |
| CHINA     | CHINA     |   1997 | 525562838002 |
| JAPAN     | VIETNAM   |   1997 | 525495763677 |
+-----------+-----------+--------+--------------+
150 rows in set (11.38 sec)  
```

Q3.2
```
MySQL [ssb_dbgen]> select c_city, s_city, d_year, sum(lo_revenue) as lo_revenue from lineorder join dates on lo_orderdate = d_datekey join customer on lo_custkey = c_custkey join supplier on lo_suppkey = s_suppkey where c_nation = 'UNITED STATES' and s_nation = 'UNITED STATES' and d_year >= 1992 and d_year <= 1997 group by c_city, s_city, d_year order by d_year asc, lo_revenue desc;
+------------+------------+--------+------------+
| c_city     | s_city     | d_year | lo_revenue |
+------------+------------+--------+------------+
| UNITED ST6 | UNITED ST6 |   1992 | 5694246807 |
| UNITED ST0 | UNITED ST0 |   1992 | 5676049026 |
| UNITED ST1 | UNITED ST1 |   1992 | 5652630617 |
| UNITED ST8 | UNITED ST6 |   1992 | 5649039075 |
                   ...
| UNITED ST3 | UNITED ST2 |   1997 | 4866105481 |
| UNITED ST9 | UNITED ST9 |   1997 | 4836163349 |
| UNITED ST9 | UNITED ST5 |   1997 | 4769919410 |
+------------+------------+--------+------------+
600 rows in set (9.20 sec)
```

Q3.3
```
MySQL [ssb_dbgen]> select c_city, s_city, d_year, sum(lo_revenue) as lo_revenue from lineorder join dates on lo_orderdate = d_datekey join customer on lo_custkey = c_custkey join supplier on lo_suppkey = s_suppkey where (c_city='UNITED KI1' or c_city='UNITED KI5') and (s_city='UNITED KI1' or s_city='UNITED KI5') and d_year >= 1992 and d_year <= 1997 group by c_city, s_city, d_year order by d_year asc, lo_revenue desc;
+------------+------------+--------+------------+
| c_city     | s_city     | d_year | lo_revenue |
+------------+------------+--------+------------+
| UNITED KI1 | UNITED KI1 |   1992 | 5776096629 |
| UNITED KI5 | UNITED KI1 |   1992 | 5555883901 |
| UNITED KI5 | UNITED KI5 |   1992 | 5348705805 |
| UNITED KI1 | UNITED KI5 |   1992 | 5326870427 |
| UNITED KI1 | UNITED KI1 |   1993 | 5892974670 |
| UNITED KI1 | UNITED KI5 |   1993 | 5490859451 |
| UNITED KI5 | UNITED KI1 |   1993 | 5468354303 |
| UNITED KI5 | UNITED KI5 |   1993 | 5089909647 |
| UNITED KI5 | UNITED KI1 |   1994 | 5437315108 |
| UNITED KI1 | UNITED KI1 |   1994 | 5348775917 |
| UNITED KI5 | UNITED KI5 |   1994 | 5310936695 |
| UNITED KI1 | UNITED KI5 |   1994 | 5237461110 |
| UNITED KI1 | UNITED KI1 |   1995 | 5737551920 |
| UNITED KI5 | UNITED KI5 |   1995 | 5657584590 |
| UNITED KI5 | UNITED KI1 |   1995 | 5260093556 |
| UNITED KI1 | UNITED KI5 |   1995 | 5213763257 |
| UNITED KI5 | UNITED KI1 |   1996 | 5522325005 |
| UNITED KI1 | UNITED KI1 |   1996 | 5451244409 |
| UNITED KI5 | UNITED KI5 |   1996 | 5231759057 |
| UNITED KI1 | UNITED KI5 |   1996 | 5203962897 |
| UNITED KI1 | UNITED KI1 |   1997 | 5340760807 |
| UNITED KI5 | UNITED KI1 |   1997 | 5295685214 |
| UNITED KI1 | UNITED KI5 |   1997 | 5188428156 |
| UNITED KI5 | UNITED KI5 |   1997 | 5024634475 |
+------------+------------+--------+------------+
24 rows in set (9.00 sec)
```

Q3.4
```
MySQL [ssb_dbgen]> select c_city, s_city, d_year, sum(lo_revenue) as lo_revenue from lineorder join dates on lo_orderdate = d_datekey join customer on lo_custkey = c_custkey join supplier on lo_suppkey = s_suppkey where (c_city='UNITED KI1' or c_city='UNITED KI5') and (s_city='UNITED KI1' or s_city='UNITED KI5') and d_yearmonth = 'Dec1997' group by c_city, s_city, d_year order by d_year asc, lo_revenue desc;
+------------+------------+--------+------------+
| c_city     | s_city     | d_year | lo_revenue |
+------------+------------+--------+------------+
| UNITED KI1 | UNITED KI1 |   1997 |  481119563 |
| UNITED KI5 | UNITED KI5 |   1997 |  386477033 |
| UNITED KI5 | UNITED KI1 |   1997 |  378048353 |
| UNITED KI1 | UNITED KI5 |   1997 |  366630529 |
+------------+------------+--------+------------+
4 rows in set (0.66 sec)
```

Q4.1
```
MySQL [ssb_dbgen]> select d_year, c_nation, sum(lo_revenue) - sum(lo_supplycost) as profit from lineorder join dates on lo_orderdate = d_datekey join customer on lo_custkey = c_custkey join supplier on lo_suppkey = s_suppkey join part on lo_partkey = p_partkey where c_region = 'AMERICA' and s_region = 'AMERICA' and (p_mfgr = 'MFGR#1' or p_mfgr = 'MFGR#2') group by d_year, c_nation order by d_year, c_nation;
+--------+---------------+---------------+
| d_year | c_nation      | profit        |
+--------+---------------+---------------+
|   1992 | ARGENTINA     | 1041983042066 |
|   1992 | BRAZIL        | 1031193572794 |
|   1992 | CANADA        | 1032094614252 |
|   1992 | PERU          | 1037331491440 |
|   1992 | UNITED STATES | 1031593944407 |
|   1993 | ARGENTINA     | 1034515265588 |
|   1993 | BRAZIL        | 1028249774691 |
|   1993 | CANADA        | 1030633924190 |
|   1993 | PERU          | 1032888811548 |
|   1993 | UNITED STATES | 1030241613033 |
|   1994 | ARGENTINA     | 1035059347804 |
|   1994 | BRAZIL        | 1029788284729 |
|   1994 | CANADA        | 1028314868119 |
|   1994 | PERU          | 1025406236588 |
|   1994 | UNITED STATES | 1035441439980 |
|   1995 | ARGENTINA     | 1036878482604 |
|   1995 | BRAZIL        | 1032846705883 |
|   1995 | CANADA        | 1031488804141 |
|   1995 | PERU          | 1034460048487 |
|   1995 | UNITED STATES | 1034988860577 |
|   1996 | ARGENTINA     | 1041240509801 |
|   1996 | BRAZIL        | 1030467525021 |
|   1996 | CANADA        | 1035089775664 |
|   1996 | PERU          | 1029765730730 |
|   1996 | UNITED STATES | 1032384751840 |
|   1997 | ARGENTINA     | 1036752881505 |
|   1997 | BRAZIL        | 1036482571346 |
|   1997 | CANADA        | 1025775840777 |
|   1997 | PERU          | 1031380143878 |
|   1997 | UNITED STATES | 1030570488847 |
|   1998 | ARGENTINA     |  607618915600 |
|   1998 | BRAZIL        |  603999739074 |
|   1998 | CANADA        |  601462066533 |
|   1998 | PERU          |  603980044827 |
|   1998 | UNITED STATES |  605069471323 |
+--------+---------------+---------------+
35 rows in set (12.61 sec)
```

Q4.2
```
MySQL [ssb_dbgen]> select d_year, s_nation, p_category, sum(lo_revenue) - sum(lo_supplycost) as profit from lineorder join dates on lo_orderdate = d_datekey join customer on lo_custkey = c_custkey join supplier on lo_suppkey = s_suppkey join part on lo_partkey = p_partkey where c_region = 'AMERICA'and s_region = 'AMERICA' and (d_year = 1997 or d_year = 1998) and (p_mfgr = 'MFGR#1' or p_mfgr = 'MFGR#2') group by d_year, s_nation, p_category order by d_year, s_nation, p_category;
+--------+---------------+------------+--------------+
| d_year | s_nation      | p_category | profit       |
+--------+---------------+------------+--------------+
|   1997 | ARGENTINA     | MFGR#11    | 102369950215 |
|   1997 | ARGENTINA     | MFGR#12    | 103052774082 |
                  ...
|   1998 | UNITED STATES | MFGR#23    |  61533867289 |
|   1998 | UNITED STATES | MFGR#24    |  60779388345 |
|   1998 | UNITED STATES | MFGR#25    |  60042710566 |
+--------+---------------+------------+--------------+
100 rows in set (4.42 sec)
```

Q4.3
```
MySQL [ssb_dbgen]> select d_year, s_city, p_brand, sum(lo_revenue) - sum(lo_supplycost) as profit from lineorder join dates on lo_orderdate = d_datekey join customer on lo_custkey = c_custkey join supplier on lo_suppkey = s_suppkey join part on lo_partkey = p_partkey where c_region = 'AMERICA'and s_nation = 'UNITED STATES' and (d_year = 1997 or d_year = 1998) and p_category = 'MFGR#14' group by d_year, s_city, p_brand order by d_year, s_city, p_brand;
+--------+------------+-----------+-----------+
| d_year | s_city     | p_brand   | profit    |
+--------+------------+-----------+-----------+
|   1997 | UNITED ST0 | MFGR#141  | 273402537 |
             ...
|   1998 | UNITED ST9 | MFGR#148  | 130780691 |
|   1998 | UNITED ST9 | MFGR#149  |  90067340 |
+--------+------------+-----------+-----------+
800 rows in set (3.68 sec)
```

### 5. 单表查询

将多表数据写入单表：
```
MySQL [ssb_dbgen]> insert into ssb_large.lineorder_flat SELECT `LO_ORDERDATE` , `LO_ORDERKEY` , `LO_LINENUMBER` , `LO_CUSTKEY` , `LO_PARTKEY` , `LO_SUPPKEY` , `LO_ORDERPRIORITY` , `LO_SHIPPRIORITY` , `LO_QUANTITY` , `LO_EXTENDEDPRICE` , `LO_ORDTOTALPRICE` , `LO_DISCOUNT` , `LO_REVENUE` , `LO_SUPPLYCOST` , `LO_TAX` , `LO_COMMITDATE` , `LO_SHIPMODE` , `C_NAME` , `C_ADDRESS` , `C_CITY` , `C_NATION` , `C_REGION` , `C_PHONE` , `C_MKTSEGMENT` , `S_NAME` , `S_ADDRESS` , `S_CITY` , `S_NATION` , `S_REGION` , `S_PHONE` , `P_NAME` , `P_MFGR` , `P_CATEGORY` , `P_BRAND` , `P_COLOR` , `P_TYPE` , `P_SIZE` , `P_CONTAINER` FROM lineorder l INNER JOIN customer c ON (c.C_CUSTKEY = l.LO_CUSTKEY) INNER JOIN supplier s ON (s.S_SUPPKEY = l.LO_SUPPKEY) INNER JOIN part p ON (p.P_PARTKEY = l.LO_PARTKEY) ;
Query OK, 600037902 rows affected (1 hour 1 min 28.10 sec)
{'label':'insert_48a7f254b72e4829-a5fbaf74a2a446d6', 'status':'VISIBLE', 'txnId':'1060'}   
```

设置系统并发：
```
set parallel_fragment_exec_instance_num=16;
```

Q1.1
```
MySQL [ssb_dbgen]> SELECT sum(LO_EXTENDEDPRICE * LO_DISCOUNT) AS revenue FROM lineorder_flat WHERE LO_ORDERDATE >= 19930101 and LO_ORDERDATE <= 19931231 AND LO_DISCOUNT BETWEEN 1 AND 3 AND LO_QUANTITY < 25;
+----------------+
| revenue    |
+----------------+
| 44652567249651 |
+----------------+
1 row in set (0.28 sec)
```

Q1.2
```
MySQL [ssb_dbgen]> SELECT sum(LO_EXTENDEDPRICE * LO_DISCOUNT) AS revenue FROM lineorder_flat WHERE LO_ORDERDATE >= 19940101 and LO_ORDERDATE <= 19940131 AND LO_DISCOUNT BETWEEN 4 AND 6 AND LO_QUANTITY BETWEEN 26 AND 35;
+---------------+
| revenue    |
+---------------+
| 9624332170119 |
+---------------+
1 row in set (0.05 sec)
```

Q1.3
```
MySQL [ssb_dbgen]> SELECT sum(LO_EXTENDEDPRICE * LO_DISCOUNT) AS revenue FROM lineorder_flat WHERE weekofyear(LO_ORDERDATE) = 6 AND LO_ORDERDATE >= 19940101 and LO_ORDERDATE <= 19941231  AND LO_DISCOUNT BETWEEN 5 AND 7 AND LO_QUANTITY BETWEEN 26 AND 35;
+---------------+
| revenue    |
+---------------+
| 2611093671163 |
+---------------+
1 row in set (0.16 sec)
```

Q2.1
```
MySQL [ssb_dbgen]> SELECT   sum(LO_REVENUE),   (LO_ORDERDATE DIV 10000) AS year,   P_BRAND FROM lineorder_flat WHERE P_CATEGORY = 'MFGR#12' AND S_REGION = 'AMERICA' GROUP BY   year,   P_BRAND ORDER BY   year,   P_BRAND;
+-------------------+------+-----------+
| sum(`LO_REVENUE`) | year | P_BRAND  |
+-------------------+------+-----------+
|    64420005618 | 1992 | MFGR#121 |
|    63389346096 | 1992 | MFGR#1210 |
|    68416605637 | 1992 | MFGR#1211 |
             ......

|    39679892915 | 1998 | MFGR#128 |
|    35300513083 | 1998 | MFGR#129 |
+-------------------+------+-----------+
280 rows in set (0.92 sec)
```

Q2.2
```
MySQL [ssb_dbgen]> SELECT   sum(LO_REVENUE),   (LO_ORDERDATE DIV 10000) AS year,   P_BRAND FROM lineorder_flat WHERE P_BRAND >= 'MFGR#2221' AND P_BRAND <= 'MFGR#2228' AND S_REGION = 'ASIA' GROUP BY   year,   P_BRAND ORDER BY   year,   P_BRAND;
+-------------------+------+-----------+
| sum(`LO_REVENUE`) | year | P_BRAND  |
+-------------------+------+-----------+
|    66450349438 | 1992 | MFGR#2221 |
|    65423264312 | 1992 | MFGR#2222 |
             ...

|    39907545239 | 1998 | MFGR#2227 |
|    40654201840 | 1998 | MFGR#2228 |
+-------------------+------+-----------+
56 rows in set (0.99 sec)
```

Q2.3
```
MySQL [ssb_dbgen]> SELECT   sum(LO_REVENUE),   (LO_ORDERDATE DIV 10000) AS year,   P_BRAND FROM lineorder_flat WHERE P_BRAND = 'MFGR#2239' AND S_REGION = 'EUROPE' GROUP BY   year,   P_BRAND ORDER BY   year,   P_BRAND;
+-------------------+------+-----------+
| sum(`LO_REVENUE`) | year | P_BRAND  |
+-------------------+------+-----------+
|    65751589723 | 1992 | MFGR#2239 |
|    64532844801 | 1993 | MFGR#2239 |
|    64722599002 | 1994 | MFGR#2239 |
|    65616432683 | 1995 | MFGR#2239 |
|    64802884686 | 1996 | MFGR#2239 |
|    64485541165 | 1997 | MFGR#2239 |
|    37276536361 | 1998 | MFGR#2239 |
+-------------------+------+-----------+
7 rows in set (0.87 sec)
```

Q3.1
```
MySQL [ssb_dbgen]> SELECT   C_NATION,   S_NATION,   (LO_ORDERDATE DIV 10000) AS year,   sum(LO_REVENUE) AS revenue FROM lineorder_flat WHERE C_REGION = 'ASIA' AND S_REGION = 'ASIA' AND LO_ORDERDATE >= 19920101 AND LO_ORDERDATE  <= 19971231 GROUP BY   C_NATION,   S_NATION,   year ORDER BY   year ASC,   revenue DESC;
+-----------+-----------+------+--------------+
| C_NATION | S_NATION | year | revenue   |
+-----------+-----------+------+--------------+
| INDIA   | INDIA   | 1992 | 537778456208 |
| INDONESIA | INDIA   | 1992 | 536684093041 |
           ...

| CHINA   | VIETNAM  | 1997 | 525867745074 |
| CHINA   | CHINA   | 1997 | 525562838002 |
| JAPAN   | VIETNAM  | 1997 | 525495763677 |
+-----------+-----------+------+--------------+
150 rows in set (1.36 sec)
```

Q3.2
```
MySQL [ssb_dbgen]> SELECT   C_CITY,   S_CITY,   (LO_ORDERDATE DIV 10000) AS year,   sum(LO_REVENUE) AS revenue FROM lineorder_flat WHERE C_NATION = 'UNITED STATES' AND S_NATION = 'UNITED STATES' AND LO_ORDERDATE >= 19920101 AND LO_ORDERDATE <= 19971231 GROUP BY   C_CITY,   S_CITY,   year ORDER BY   year ASC,   revenue DESC;
+------------+------------+------+------------+
| C_CITY   | S_CITY   | year | revenue  |
+------------+------------+------+------------+
| UNITED ST6 | UNITED ST6 | 1992 | 5694246807 |
| UNITED ST0 | UNITED ST0 | 1992 | 5676049026 |
                   ...

| UNITED ST9 | UNITED ST9 | 1997 | 4836163349 |
| UNITED ST9 | UNITED ST5 | 1997 | 4769919410 |
+------------+------------+------+------------+
600 rows in set (0.78 sec)
```

Q3.3
```
MySQL [ssb_dbgen]> SELECT   C_CITY,   S_CITY,   (LO_ORDERDATE DIV 10000) AS year,   sum(LO_REVENUE) AS revenue FROM lineorder_flat WHERE C_CITY in ( 'UNITED KI1' ,'UNITED KI5') AND S_CITY in ( 'UNITED KI1' ,'UNITED KI5') AND LO_ORDERDATE >= 19920101 AND LO_ORDERDATE <= 19971231 GROUP BY   C_CITY,   S_CITY,   year ORDER BY   year ASC,   revenue DESC;
+------------+------------+------+------------+
| C_CITY   | S_CITY   | year | revenue  |
+------------+------------+------+------------+
| UNITED KI1 | UNITED KI1 | 1992 | 5776096629 |
| UNITED KI5 | UNITED KI1 | 1992 | 5555883901 |
| UNITED KI5 | UNITED KI5 | 1992 | 5348705805 |
| UNITED KI1 | UNITED KI5 | 1992 | 5326870427 |
| UNITED KI1 | UNITED KI1 | 1993 | 5892974670 |
| UNITED KI1 | UNITED KI5 | 1993 | 5490859451 |
| UNITED KI5 | UNITED KI1 | 1993 | 5468354303 |
| UNITED KI5 | UNITED KI5 | 1993 | 5089909647 |
| UNITED KI5 | UNITED KI1 | 1994 | 5437315108 |
| UNITED KI1 | UNITED KI1 | 1994 | 5348775917 |
| UNITED KI5 | UNITED KI5 | 1994 | 5310936695 |
| UNITED KI1 | UNITED KI5 | 1994 | 5237461110 |
| UNITED KI1 | UNITED KI1 | 1995 | 5737551920 |
| UNITED KI5 | UNITED KI5 | 1995 | 5657584590 |
| UNITED KI5 | UNITED KI1 | 1995 | 5260093556 |
| UNITED KI1 | UNITED KI5 | 1995 | 5213763257 |
| UNITED KI5 | UNITED KI1 | 1996 | 5522325005 |
| UNITED KI1 | UNITED KI1 | 1996 | 5451244409 |
| UNITED KI5 | UNITED KI5 | 1996 | 5231759057 |
| UNITED KI1 | UNITED KI5 | 1996 | 5203962897 |
| UNITED KI1 | UNITED KI1 | 1997 | 5340760807 |
| UNITED KI5 | UNITED KI1 | 1997 | 5295685214 |
| UNITED KI1 | UNITED KI5 | 1997 | 5188428156 |
| UNITED KI5 | UNITED KI5 | 1997 | 5024634475 |
+------------+------------+------+------------+
24 rows in set (0.84 sec)
```

Q3.4
```
MySQL [ssb_dbgen]> SELECT   C_CITY,   S_CITY,   (LO_ORDERDATE DIV 10000) AS year,   sum(LO_REVENUE) AS revenue FROM lineorder_flat WHERE C_CITY in ('UNITED KI1', 'UNITED KI5') AND S_CITY in ( 'UNITED KI1', 'UNITED KI5') AND LO_ORDERDATE >= 19971201 AND LO_ORDERDATE <= 19971231 GROUP BY   C_CITY,   S_CITY,   year ORDER BY   year ASC,   revenue DESC;
+------------+------------+------+-----------+
| C_CITY   | S_CITY   | year | revenue  |
+------------+------------+------+-----------+
| UNITED KI1 | UNITED KI1 | 1997 | 481119563 |
| UNITED KI5 | UNITED KI5 | 1997 | 386477033 |
| UNITED KI5 | UNITED KI1 | 1997 | 378048353 |
| UNITED KI1 | UNITED KI5 | 1997 | 366630529 |
+------------+------------+------+-----------+
4 rows in set (0.06 sec)
```

Q4.1
```
MySQL [ssb_dbgen]> SELECT  (LO_ORDERDATE DIV 10000) AS year,   C_NATION,   sum(LO_REVENUE - LO_SUPPLYCOST) AS profit FROM lineorder_flat WHERE C_REGION = 'AMERICA' AND S_REGION = 'AMERICA' AND P_MFGR in ( 'MFGR#1' , 'MFGR#2') GROUP BY   year,   C_NATION ORDER BY   year ASC,   C_NATION ASC;
+------+---------------+---------------+
| year | C_NATION   | profit    |
+------+---------------+---------------+
| 1992 | ARGENTINA   | 1041983042066 |
| 1992 | BRAZIL    | 1031193572794 |
| 1992 | CANADA    | 1032094614252 |
| 1992 | PERU     | 1037331491440 |
| 1992 | UNITED STATES | 1031593944407 |
| 1993 | ARGENTINA   | 1034515265588 |
| 1993 | BRAZIL    | 1028249774691 |
| 1993 | CANADA    | 1030633924190 |
| 1993 | PERU     | 1032888811548 |
| 1993 | UNITED STATES | 1030241613033 |
| 1994 | ARGENTINA   | 1035059347804 |
| 1994 | BRAZIL    | 1029788284729 |
| 1994 | CANADA    | 1028314868119 |
| 1994 | PERU     | 1025406236588 |
| 1994 | UNITED STATES | 1035441439980 |
| 1995 | ARGENTINA   | 1036878482604 |
| 1995 | BRAZIL    | 1032846705883 |
| 1995 | CANADA    | 1031488804141 |
| 1995 | PERU     | 1034460048487 |
| 1995 | UNITED STATES | 1034988860577 |
| 1996 | ARGENTINA   | 1041240509801 |
| 1996 | BRAZIL    | 1030467525021 |
| 1996 | CANADA    | 1035089775664 |
| 1996 | PERU     | 1029765730730 |
| 1996 | UNITED STATES | 1032384751840 |
| 1997 | ARGENTINA   | 1036752881505 |
| 1997 | BRAZIL    | 1036482571346 |
| 1997 | CANADA    | 1025775840777 |
| 1997 | PERU     | 1031380143878 |
| 1997 | UNITED STATES | 1030570488847 |
| 1998 | ARGENTINA   | 607618915600 |
| 1998 | BRAZIL    | 603999739074 |
| 1998 | CANADA    | 601462066533 |
| 1998 | PERU     | 603980044827 |
| 1998 | UNITED STATES | 605069471323 |
+------+---------------+---------------+
35 rows in set (1.35 sec)
```

Q4.2
```
MySQL [ssb_dbgen]> SELECT  (LO_ORDERDATE DIV 10000) AS year,   S_NATION,   P_CATEGORY,   sum(LO_REVENUE - LO_SUPPLYCOST) AS profit FROM lineorder_flat WHERE C_REGION = 'AMERICA' AND S_REGION = 'AMERICA' AND LO_ORDERDATE >= 19970101 and LO_ORDERDATE <= 19981231 AND P_MFGR in ( 'MFGR#1' , 'MFGR#2') GROUP BY   year,   S_NATION,   P_CATEGORY ORDER BY   year ASC,   S_NATION ASC,   P_CATEGORY ASC;
+------+---------------+------------+--------------+
| year | S_NATION   | P_CATEGORY | profit    |
+------+---------------+------------+--------------+
| 1997 | ARGENTINA   | MFGR#11  | 102369950215 |
| 1997 | ARGENTINA   | MFGR#12  | 103052774082 |
            ...

| 1998 | UNITED STATES | MFGR#23  | 61533867289 |
| 1998 | UNITED STATES | MFGR#24  | 60779388345 |
| 1998 | UNITED STATES | MFGR#25  | 60042710566 |
+------+---------------+------------+--------------+
100 rows in set (0.48 sec)
```

Q4.3
```
MySQL [ssb_dbgen]> SELECT   (LO_ORDERDATE DIV 10000) AS year,   S_CITY,   P_BRAND,   sum(LO_REVENUE - LO_SUPPLYCOST) AS profit FROM lineorder_flat WHERE S_NATION = 'UNITED STATES' AND LO_ORDERDATE >= 19970101 and LO_ORDERDATE <= 19981231 AND P_CATEGORY = 'MFGR#14' GROUP BY   year,   S_CITY,   P_BRAND ORDER BY   year ASC,   S_CITY ASC,   P_BRAND ASC;
+------+------------+-----------+------------+
| year | S_CITY   | P_BRAND  | profit   |
+------+------------+-----------+------------+
| 1997 | UNITED ST0 | MFGR#141 | 1402715668 |
| 1997 | UNITED ST0 | MFGR#1410 | 1309800423 |
| 1997 | UNITED ST0 | MFGR#1411 | 1252501939 |
| 1997 | UNITED ST0 | MFGR#1412 | 1295391924 |
            ...

| 1998 | UNITED ST9 | MFGR#146 | 770184207 |
| 1998 | UNITED ST9 | MFGR#147 | 640908455 |
| 1998 | UNITED ST9 | MFGR#148 | 811919859 |
| 1998 | UNITED ST9 | MFGR#149 | 603099066 |
+------+------------+-----------+------------+
800 rows in set (0.37 sec)
```

### 6. 并发测试

Q4.1
```
SELECT  (LO_ORDERDATE DIV 10000) AS year,   C_NATION,   sum(LO_REVENUE - LO_SUPPLYCOST) AS profit FROM lineorder_flat WHERE C_REGION = 'AMERICA' AND S_REGION = 'AMERICA' AND P_MFGR in ( 'MFGR#1' , 'MFGR#2') GROUP BY   year,   C_NATION ORDER BY   year ASC,   C_NATION ASC;
```

```
并发数：1
[weizuo@PC ~]$ mysqlslap -hhost -P9030 -uxxx --concurrency=1 --iterations=1 --query=/home/weizuo/test.sql --create-schema='ssb_dbgen' --number-of-queries=1
Benchmark
        Average number of seconds to run all queries: 1.542 seconds
        Minimum number of seconds to run all queries: 1.542 seconds
        Maximum number of seconds to run all queries: 1.542 seconds
        Number of clients running queries: 1
        Average number of queries per client: 1

并发数：10
[weizuo@PC ~]$ mysqlslap -hhost -P9030 -uxxx --concurrency=10 --iterations=1 --query=/home/weizuo/test.sql --create-schema='ssb_dbgen' --number-of-queries=10
Benchmark
        Average number of seconds to run all queries: 12.968 seconds
        Minimum number of seconds to run all queries: 12.968 seconds
        Maximum number of seconds to run all queries: 12.968 seconds
        Number of clients running queries: 10
        Average number of queries per client: 1

并发数：20        
[weizuo@PC ~]$ mysqlslap -hhost -P9030 -uxxx --concurrency=20 --iterations=1 --query=/home/weizuo/test.sql --create-schema='ssb_dbgen' --number-of-queries=20
Benchmark
        Average number of seconds to run all queries: 25.506 seconds
        Minimum number of seconds to run all queries: 25.506 seconds
        Maximum number of seconds to run all queries: 25.506 seconds
        Number of clients running queries: 20
        Average number of queries per client: 1
        
并发数：30
[weizuo@PC ~]$ mysqlslap -hhost -P9030 -uxxx --concurrency=30 --iterations=1 --query=/home/weizuo/test.sql --create-schema='ssb_dbgen' --number-of-queries=30
Benchmark
        Average number of seconds to run all queries: 37.832 seconds
        Minimum number of seconds to run all queries: 37.832 seconds
        Maximum number of seconds to run all queries: 37.832 seconds
        Number of clients running queries: 30
        Average number of queries per client: 1
        
并发数：50      
[weizuo@PC ~]$ mysqlslap -hhost -P9030 -uxxx --concurrency=50 --iterations=1 --query=/home/weizuo/test.sql --create-schema='ssb_dbgen' --number-of-queries=50
    查询超时
```