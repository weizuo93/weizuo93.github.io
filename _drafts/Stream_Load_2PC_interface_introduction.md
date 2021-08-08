### Stream Load 2PC接口说明

#### 1. 数据写入(第一阶段)

1）访问接口：
```
curl --location-trusted -u root: -H "two_phase_commit:true" -H "label:xxx" -H "column_separator:|" -T ' + file +' http://fe_host:http_port/api/{db}/{table}/_stream_load;
```
或
```
curl --location-trusted -u root: -H "two_phase_commit:true" -H "label:xxx" -H "column_separator:|" -T ' + file +' http://be_host:webserver_port/api/{db}/{table}/_stream_load;
```
数据写入兼容原始的Stream Load接口，通过将header参数two_phase_commit的值设置为true来启动2PC（默认为false，即使用原始的Stream Load方式导入数据）。

2）返回信息
数据写入阶段的返回信息兼容了原始Stream Load的返回信息，在返回信息中增加了TwoPhaseCommit字段，来表示是否通过2PC的方式写入。
```
{
    "TxnId": xxx,
    "Label": "xxx",
    "TwoPhaseCommit": "true",
    "Status": "Success",
    "Message": "OK",
    "NumberTotalRows": xxx,
    "NumberLoadedRows": xxx,
    "NumberFilteredRows": xxx,
    "NumberUnselectedRows": xxx,
    "LoadBytes": xxx,
    "LoadTimeMs": xxx,
    "BeginTxnTimeMs": xxx,
    "StreamLoadPutTimeMs": xxx,
    "ReadDataTimeMs": xxx,
    "WriteDataTimeMs": xxx,
    "CommitAndPublishTimeMs": xxx
}
```

如果发生重复导入，Label已经被使用，则返回信息中会携带相关的txn id信息：
```
{
    "TxnId": -1,
    "Label": "xxxx",
    "TwoPhaseCommit": "true",
    "Status": "Label Already Exists",
    "ExistingJobStatus": "PRECOMMITTED",
    "Message": "errCode = 2, detailMessage = Label [xxx] has already been used, relate to txn [xxx]",
    "NumberTotalRows": 0,
    "NumberLoadedRows": 0,
    "NumberFilteredRows": 0,
    "NumberUnselectedRows": 0,
    "LoadBytes": 0,
    "LoadTimeMs": 0,
    "BeginTxnTimeMs": 0,
    "StreamLoadPutTimeMs": 0,
    "ReadDataTimeMs": 0,
    "WriteDataTimeMs": 0,
    "CommitAndPublishTimeMs": 0
}
```
如果Stream Load 2PC功能被关闭，会有如下报错信息，需要联系doris运维同学开启该功能。
```
{
    "TxnId": -1,
    "Label": "xxx",
    "TwoPhaseCommit": "true",
    "Status": "Fail",
    "Message": "Two phase commit (2PC) for stream load was disabled",
    "NumberTotalRows": 0,
    "NumberLoadedRows": 0,
    "NumberFilteredRows": 0,
    "NumberUnselectedRows": 0,
    "LoadBytes": 0,
    "LoadTimeMs": 0,
    "BeginTxnTimeMs": 0,
    "StreamLoadPutTimeMs": 0,
    "ReadDataTimeMs": 0,
    "WriteDataTimeMs": 0,
    "CommitAndPublishTimeMs": 0
}
```
#### 2. Commit(第二阶段)

1）访问接口：
```
 curl -X PUT --location-trusted -u user:passwd -H "txn_id:txnId" -H "txn_operation:commit" http://fe_host:http_port/api/{db}/_stream_load_2pc
 ```
 或
 ```
 curl -X PUT --location-trusted -u user:passwd -H "txn_id:txnId" -H "txn_operation:commit" http://be_host:webserver_port/api/{db}/_stream_load_2pc
```

2）返回信息
* Commit成功
```
{
    "status": "Success",
    "msg": "transaction [txnId] commit successfully."
}
```

* Commit失败
```
{
    "status": "Fail",
    "msg": "xxxxxx"
}
```

失败信息说明如下：
| 失败信息 | 说明 | 
| :-----| :---- |
|Two phase commit (2PC) for stream load was disabled|2PC功能是关闭的，需要联系运维同学开启|
|errCode = 2, detailMessage = Access denied for default_cluster:xxx@xxx|用户权限有误|
|errCode = 2, detailMessage = unknown database, database=default_cluster:test|DB不存在|
| convert txn_id [xxx] failed | header中txn_id信息缺失，或txn_id信息格式有误（非数值） | 
| transaction operation should be 'commit' or 'abort' | header中txn_operation信息缺失，或txn_operation的值有误（非commit和abort）| 
|errCode = 2, detailMessage = transaction [xxx] not found|Transaction不存在|
|errCode = 2, detailMessage = transaction [xxx] is already aborted. abort reason: User Abort|Transaction已经被用户abort|
|errCode = 2, detailMessage = transaction [xxx] is already aborted. abort reason: timeout by txn manager|Transaction因为超时已经被系统abort|
|errCode = 2, detailMessage = transaction [xxx] is prepare, not pre-committed.|Transaction处于Prepare状态，数据还未写入完成，不能执行Commit|
|errCode = 2, detailMessage = transaction [xxx] is already committed, not pre-committed.|Transaction已经被Committed|
|errCode = 2, detailMessage = transaction [xxx] is already visible, not pre-committed.|Transaction已经是可见状态|



#### 3. Abort(第二阶段)
1）访问接口
```
curl -X PUT --location-trusted -u user:passwd -H "txn_id:txnId" -H "txn_operation:abort" http://fe_host:http_port/api/{db}/_stream_load_2pc
```
或
```
 curl -X PUT --location-trusted -u user:passwd -H "txn_id:txnId" -H "txn_operation:abort" http://be_host:webserver_port/api/{db}/_stream_load_2pc
```

2）返回信息
* Abort成功
```
{
    "status": "Success",
    "msg": "transaction [xxx] abort successfully."
}
```

* Abort失败
```
{
    "status": "Fail",
    "msg": "xxx"
}
```

失败信息说明如下：
| 失败信息 | 说明 | 
| :-----| :---- |
|Two phase commit (2PC) for stream load was disabled|2PC功能是关闭的，需要联系运维同学开启|
|errCode = 2, detailMessage = Access denied for default_cluster:xxx@xxx|用户权限有误|
|errCode = 2, detailMessage = unknown database, database=default_cluster:test|DB不存在|
| convert txn_id [xxx] failed | header中txn_id信息缺失，或txn_id信息格式有误（非数值） | 
| transaction operation should be 'commit' or 'abort' | header中txn_operation信息缺失，或txn_operation的值有误（非commit和abort）| 
|errCode = 2, detailMessage = transaction [xxx] not found|Transaction不存在|
|errCode = 2, detailMessage = transaction [xxx] is already aborted. abort reason: User Abort|Transaction已经被用户abort|
|errCode = 2, detailMessage = transaction [xxx] is already aborted. abort reason: timeout by txn manager|Transaction因为超时已经被系统abort|
|errCode = 2, detailMessage = transaction [2] is already COMMITTED, could not abort.|Transaction已经是Committed状态，不能被Abort|
|errCode = 2, detailMessage = transaction [2] is already VISIBLE, could not abort.|Transaction已经是可见状态，不能被Abort|

