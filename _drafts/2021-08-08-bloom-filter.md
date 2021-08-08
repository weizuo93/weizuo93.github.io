---
layout: post
title: Apache Doris 的Bloom Filter索引原理 
categories: Doris原理解析
description: Apache Doris 的布隆过滤器原理。
keywords: 布隆过滤器、Apache Doris
---

### 引言

Apache Doris 支持布隆过滤器（Bloom Filter）索引，可以对取值区分度比较大的字段添加布隆过滤器（Bloom Filter）索引，布隆过滤器（Bloom Filter）索引按照data page的粒度生成。数据写入时，会记录每一个写入data page的值，当一个data page写满之后，会根据该data page的所有取值为该data page生成布隆过滤器（Bloom Filter）索引。数据查询时，查询条件在设置有布隆过滤器（Bloom Filter）索引的字段进行过滤，当某个data page的Bloom Filter没有命中时表示该data page中没有需要的数据，这样可以对data page进行快速过滤，减少不必要的数据读取。

### 布隆过滤器（Bloom Filter）索引原理

Apache Doris中采用了基于block的布隆过滤器算法，主要实现了Putze et al.的论文《Cache-, Hash- and Space-Efficient Bloom filters》。每一个data page对应的布隆过滤器（Bloom Filter）索引数据会被划分为多个block，每一个block的数据长度为BYTES_PER_BLOCK（32字节，共256bit），block中的每一个bit会被初始化为0。向data page中写入数据时，每一个取值value都会将一个block中的BITS_SET_PER_BLOCK（默认值为8）个bit置位为1。判断某个value是否命中Bloom Filter时，计算该value在Bloom Filter中对应的block，并判断该value在block中对应的BITS_SET_PER_BLOCK个bit是否都已经被置位为1，如果该value在block中对应的BITS_SET_PER_BLOCK个bit中存在0，则表示Bloom Filter未命中，可以跳过该Bloom Filter对应的data page。

Apache Doris中布隆过滤器（Bloom Filter）的Hash策略为HASH_MURMUR3_X64_64。单个data page的布隆过滤器（Bloom Filter）索引数据长度BLOOM_FILTER_BIT通过如下公式计算：

```
        BLOOM_FILTER_BIT = -N * ln(FPP) / (ln(2) ^ 2)
```

其中，N表示当前data page中的不同取值的个数；FPP（False Positive Probablity） 取值为0.05。计算出的Bloom Filter数据长度（单位为bit）一定是2的整数次幂。

布隆过滤器（Bloom Filter）中，每一个block的长度为BYTES_PER_BLOCK（32字节），因此，单个data page的布隆过滤器（Bloom Filter）中的block数量通过如下公式计算：

```
BLOCK_NUM = (BLOOM_FILTER_BIT / 8)  / BYTES_PER_BLOCK;
```

#### 索引数据生成

针对data page中的每一个取值value，都基于HASH_MURMUR3_X64_64计算出64位的HASH_CODE。

取HASH_CODE的高32位计算出当前value在布隆过滤器（Bloom Filter）中对应的BLOCK_INDEX：

```
BLOCK_INDEX = (HASH_CODE >> 32) & (BLOCK_NUM - 1)
```

其中，BLOCK_NUM为2的整数次幂，则BLOCK_INDEX一定小于BLOCK_NUM。

取HASH_CODE的低32位计算出当前value会将对应的256 bit的block中的哪些bit置位为1，方法如下：

```
uint32_t key = (uint32_t)HASH_CODE

uint32_t SALT[8] = {0x47b6137b, 0x44974d91, 0x8824ad5b, 0xa2b7289d,  0x705495c7, 0x2df1424b, 0x9efc4947, 0x5c6bfb31};

uint32_t masks[BITS_SET_PER_BLOCK];
for (int i = 0; i < BITS_SET_PER_BLOCK; ++i) {
        masks[i] = key * SALT[i]; 
        masks[i] = masks[i] >> 27; 
        masks[i] = 0x1 << masks[i]; 
}
```

masks[i]中包含32个bit，其中只有1个bit值为1，其他31个bit值均为0。masks[i]会与block中第i个32bit按位取或。

```
uint32_t* BLOCK_OFFSET =  BLOOM_FILTER_OFFSET + BYTES_PER_BLOCK * BLOCK_INDEX

for (int i = 0; i < BITS_SET_PER_BLOCK; ++i) {
*(BLOCK_OFFSET + i) |= masks[i];
}
```

其中，BLOOM_FILTER_OFFSET表示Bloom Filter的偏置，BLOCK_OFFSET表示当前block的偏置。
<center>（图）</center>

#### 数据查询

数据查询时，查询条件（"="、"is"或"in"语句）在设置有布隆过滤器（Bloom Filter）索引的列依次对每一个data page进行过滤。首先，基于HASH_MURMUR3_X64_64方法对过滤条件的值value计算出64位的HASH_CODE；然后，采用与生成Bloom Filter索引数据相同的方法计算出该value值在Bloom Filter中对应的block，以及在block中对应的BITS_SET_PER_BLOCK个bit位。如果block中的这BITS_SET_PER_BLOCK个bit值均为1，则表示Bloom Filter命中，该value值在Bloom Filter对应的data page中可能存在；否则，表示Bloom Filter未命中，该value值在Bloom Filter对应的data page中一定不存在。

### 适用场景

Apache Doris支持在建表时对指定的列创建Bloom Filter索引，也可以对已经创建的表执行Alter Table命令添加Bloom Filter索引。

```
ALTER TABLE example_db.my_table SET ("bloom_filter_columns"="k1,k2,k3");
```

目前只支持对smallint、int、unsigned int、 bigint、largeint、char、 varchar、date、datetime和decimal类型的字段创建Bloom Filter索引，其他类型的字段均不支持Bloom Filter索引。对于创建了Bloom Filter索引的字段，查询条件是"="、"is"或"in"语句时，才会使用Bloom Filter索引进行data page的过滤。