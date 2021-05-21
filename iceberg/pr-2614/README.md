1000W records

ORC format

like below:
```
tt.id	tt.name	tt.age	tt.addr	tt.tel
1	name	age	addr	tel
2	name	age	addr	tel
3	name	age	addr	tel
4	name	age	addr	tel
5	name	age	addr	tel
6	name	age	addr	tel
7	name	age	addr	tel
8	name	age	addr	tel
9	name	age	addr	tel
10	name	age	addr	tel
```


test step:

1.open hive shell: hive-shell

2.prepare hive env:
```
create database iceberg_test;

use iceberg_test;

add jar /app/iceberg_lib/iceberg-hive-runtime-0.11.0.jar;
```

3.prepare a source table with data
```
CREATE EXTERNAL TABLE `iceberg_test.tt`(
  `id` string,
  `name` string,
  `age` string,
  `addr` string,
  `tel` string)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.ql.io.orc.OrcSerde'
WITH SERDEPROPERTIES (
  'field.delim'='\u0001',
  'line.delim'='\n',
  'serialization.format'='\u0001',
  'serialization.null.format'='')
STORED AS INPUTFORMAT
  'org.apache.hadoop.hive.ql.io.orc.OrcInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat'
LOCATION
  'hdfs://icebergHadoop/user/hive/warehouse/iceberg_test.db/tt/';
```
upload data file to hadoop: `hadoop fs -put orc_000000_0 hdfs://icebergHadoop/user/hive/warehouse/iceberg_test.db/tt/`

4.create 2 iceberg tables
```
CREATE EXTERNAL TABLE `iceberg_test.a`(
  `id` string,
  `name` string,
  `age` string,
  `addr` string)
ROW FORMAT SERDE
  'org.apache.iceberg.mr.hive.HiveIcebergSerDe'
STORED BY
  'org.apache.iceberg.mr.hive.HiveIcebergStorageHandler';

CREATE EXTERNAL TABLE `iceberg_test.b`(
  `id` string,
  `name` string,
  `age` string,
  `tel` string)
ROW FORMAT SERDE
  'org.apache.iceberg.mr.hive.HiveIcebergSerDe'
STORED BY
  'org.apache.iceberg.mr.hive.HiveIcebergStorageHandler';
```

5.insert data into 2 iceberg tables

table 1: `insert into table iceberg_test.a select id,name,age,addr from iceberg_test.tt limit 1;`
Ps: left table 1 only need 1 record. 

table 2: `insert into table iceberg_test.b select id,name,age,tel from iceberg_test.tt;`

insert into table2 again: `insert into table iceberg_test.b select id,name,age,tel from iceberg_test.tt;`

Ps: right table 2 must insert twice, Make sure to generate two large data files.

6.exec query SQL
```
select * from
(select * from iceberg_test.a where age='age')p1
left join
(select id,name,age from iceberg_test.b where name='name') p2
on p1.id=p2.id limit 10;
```

And then we will get error:
```
Error: java.lang.RuntimeException: Error in configuring object
	at org.apache.hadoop.util.ReflectionUtils.setJobConf(ReflectionUtils.java:113)
	at org.apache.hadoop.util.ReflectionUtils.setConf(ReflectionUtils.java:79)
	at org.apache.hadoop.util.ReflectionUtils.newInstance(ReflectionUtils.java:137)
	at org.apache.hadoop.mapred.MapTask.runOldMapper(MapTask.java:450)
	at org.apache.hadoop.mapred.MapTask.run(MapTask.java:343)
	at org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:164)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1893)
	at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:158)
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.util.ReflectionUtils.setJobConf(ReflectionUtils.java:110)
	... 9 more
Caused by: java.lang.RuntimeException: Error in configuring object
	at org.apache.hadoop.util.ReflectionUtils.setJobConf(ReflectionUtils.java:113)
	at org.apache.hadoop.util.ReflectionUtils.setConf(ReflectionUtils.java:79)
	at org.apache.hadoop.util.ReflectionUtils.newInstance(ReflectionUtils.java:137)
	at org.apache.hadoop.mapred.MapRunner.configure(MapRunner.java:38)
	... 14 more
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.util.ReflectionUtils.setJobConf(ReflectionUtils.java:110)
	... 17 more
Caused by: java.lang.RuntimeException: Map operator initialization failed
	at org.apache.hadoop.hive.ql.exec.mr.ExecMapper.configure(ExecMapper.java:125)
	... 22 more
Caused by: java.lang.RuntimeException: cannot find field addr from [org.apache.iceberg.mr.hive.serde.objectinspector.IcebergRecordObjectInspector$IcebergRecordStructField@e77fd7de, org.apache.iceberg.mr.hive.serde.objectinspector.IcebergRecordObjectInspector$IcebergRecordStructField@49e10d0e, org.apache.iceberg.mr.hive.serde.objectinspector.IcebergRecordObjectInspector$IcebergRecordStructField@9053e2ba]
	at org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorUtils.getStandardStructFieldRef(ObjectInspectorUtils.java:523)
	at org.apache.iceberg.mr.hive.serde.objectinspector.IcebergRecordObjectInspector.getStructFieldRef(IcebergRecordObjectInspector.java:70)
	at org.apache.hadoop.hive.ql.exec.ExprNodeColumnEvaluator.initialize(ExprNodeColumnEvaluator.java:56)
	at org.apache.hadoop.hive.ql.exec.Operator.initEvaluators(Operator.java:1033)
	at org.apache.hadoop.hive.ql.exec.Operator.initEvaluatorsAndReturnStruct(Operator.java:1059)
	at org.apache.hadoop.hive.ql.exec.SelectOperator.initializeOp(SelectOperator.java:75)
	at org.apache.hadoop.hive.ql.exec.Operator.initialize(Operator.java:366)
	at org.apache.hadoop.hive.ql.exec.Operator.initialize(Operator.java:556)
	at org.apache.hadoop.hive.ql.exec.Operator.initializeChildren(Operator.java:508)
	at org.apache.hadoop.hive.ql.exec.Operator.initialize(Operator.java:376)
	at org.apache.hadoop.hive.ql.exec.Operator.initialize(Operator.java:556)
	at org.apache.hadoop.hive.ql.exec.Operator.initializeChildren(Operator.java:508)
	at org.apache.hadoop.hive.ql.exec.Operator.initialize(Operator.java:376)
	at org.apache.hadoop.hive.ql.exec.MapOperator.initializeMapOperator(MapOperator.java:501)
	at org.apache.hadoop.hive.ql.exec.mr.ExecMapper.configure(ExecMapper.java:104)
	... 22 more
```
