
title: canal同步的TimeStamp问题
date: 2017-12-11 13:30:37
tags: [youdaonote]
---

部门接了新的实时方面的需求，选定使用canal进行mysql binlog方面的处理。

上线一段时间后都比较稳定，但是有一天，实时数据开发的同事找到我说有个别字段是有问题的。大概表结构如下：
```sql
CREATE TABLE `bb` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `lastUpdateTime` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `firstTimeBindTime` datetime DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM AUTO_INCREMENT=1526640 DEFAULT CHARSET=utf8 
```
有问题的字段为`lastUpdateTime`，主库为`2017-12-11 11:22:40`的时候，canal client中消费到的却是`2017-12-10 22:22:40`，而且所有这个字段都是相差同样的13个小时。但是有一个地方比较值得注意，就是`firstTimeBindTime`的值都是正常的。


#### 排查流程
- 查看canal server的系统时区，进行修正确认
```
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime 
date # 发现时间已经正确了
```
- 重启canal server

发现还是有问题，去mysql主机上翻看binlog，找到对应的记录。
```
mysqlbinlog -vv /server/mysql_data/mysql-bin.000141 
```

发现这个字段对应的时间戳是正确的`1512962560`。那么问题看来还是在canal server这块了。



#### 源码走读

顺便看下源码吧。对于binlog进行解析的主类：`MysqlEventParser.start()`

```java

...
// event处理与解析
CanalEntry.Entry entry = parseAndProfilingIfNecessary(event);
...

```

进入`parseAndProfilingIfNecessary`方法

```java
...
// 使用parser解析
CanalEntry.Entry event = binlogParser.parse(bod);
...
```

找到对应的parser类为LogEventConvert，可以看到`LogEventConvert`是继承了`BinlogParser`的。
```java
protected BinlogParser buildParser() {
    LogEventConvert convert = new LogEventConvert();
    ......
    return convert;
}

```

再去`LogEventConvert`的parse方法
```java
...
// 对于修改数据的event的处理
case LogEvent.WRITE_ROWS_EVENT:
    return parseRowsEvent((WriteRowsLogEvent) logEvent);
...
```

继续找
```java
while (buffer.nextOneRow(columns)) {
    // 处理row记录
    RowData.Builder rowDataBuilder = RowData.newBuilder();
    if (EventType.INSERT == eventType) {
        // insert的记录放在before字段中
        tableError |= parseOneRow(rowDataBuilder, event, buffer, columns, true, tableMeta);
    } else if (EventType.DELETE == eventType) {
        // delete的记录放在before字段中
        tableError |= parseOneRow(rowDataBuilder, event, buffer, columns, false, tableMeta);
    } else {
        // update需要处理before/after
        tableError |= parseOneRow(rowDataBuilder, event, buffer, columns, false, tableMeta);
        if (!buffer.nextOneRow(changeColumns)) {
            rowChangeBuider.addRowDatas(rowDataBuilder.build());
            break;
        }

        tableError |= parseOneRow(rowDataBuilder, event, buffer, changeColumns, true, tableMeta);
    }

    rowChangeBuider.addRowDatas(rowDataBuilder.build());
}

```

对于行数据的每个字段的处理，是先获取到value，然后再针对地做一些额外处理。
```java
....
final Serializable value = buffer.getValue();
...
```

通过`RowsLogBuffer`进行value的获取，我们找到给value赋值的地方`fetchValue`
```java
final Serializable fetchValue(int type, final int meta, boolean isBinary) {
    .....
    switch (type) {
	 case LogEvent.MYSQL_TYPE_TIMESTAMP: {
        final long i32 = buffer.getUint32();
        if (i32 == 0) {
            value = "0000-00-00 00:00:00";
        } else {
            String v = new Timestamp(i32 * 1000).toString();
            value = v.substring(0, v.length() - 2);
        }
        javaType = Types.TIMESTAMP;
        length = 4;
        break;
    }
    .....
}

```
把这部分弄出去，使用上面binlog文件中的timestamp单独测试.

```java

import java.sql.Timestamp;

/**
 * Created by root on 17-12-11.
 */
public class Will {
    public static void main(String[] p ){
        String value = "";
        final long i32 = 1512962560l;
        System.out.println(TimeZone.getDefault());
        System.out.println(i32);
        if (i32 == 0) {
            value = "0000-00-00 00:00:00";
        } else {
            String v = new Timestamp(i32 * 1000).toString();
            value = v.substring(0, v.length() - 2);
        }
        System.out.println(value);
    }
}

```

在测试机是正常的, 但是在线上环境是不正常的，最后在java里输出了一下`Timezone.getDefault()`，发现线上是"America/New_York"，正好就是差13个小时的。可是在系统里运行date命令，返回的是上海时区。上网搜了一下，有人建议使用下面命令确定时区
```
[root@etl01 ~]# ls -l /etc/localtime
lrwxrwxrwx. 1 root root 38 Oct 21 00:25 /etc/localtime -> ../usr/share/zoneinfo/America/New_York
```
这里看到虽然我前面使用cp已经覆盖了/etc/localtime文件，而且并没有报错。但是查看的时候，还是显示为到纽约的链接。


删掉这个文件，重新cp。
```
[root@etl01 ~]# rm -vf /etc/localtime
removed \u2018/etc/localtime\u2019
[root@etl01 ~]# cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime 
[root@etl01 ~]# ls -l /etc/localtime
-rw-r--r--. 1 root root 405 Dec 11 13:14 /etc/localtime
[root@etl01 ~]# java Will
sun.util.calendar.ZoneInfo[id="America/New_York",offset=-18000000,dstSavings=3600000,useDaylight=true,transitions=235,lastRule=java.util.SimpleTimeZone[id=America/New_York,offset=-18000000,dstSavings=3600000,useDaylight=true,startYear=0,startMode=3,startMonth=2,startDay=8,startDayOfWeek=1,startTime=7200000,startTimeMode=0,endMode=3,endMonth=10,endDay=1,endDayOfWeek=1,endTime=7200000,endTimeMode=0]]
America/New_York

```

还是不对，无奈，再次删掉。按照系统的样子，创建软链，成功。
```
[root@etl01 ~]# rm /etc/localtime 
rm: remove regular file \u2018/etc/localtime\u2019? y
[root@etl01 ~]# ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
[root@etl01 ~]# ls -l /etc/localtime
lrwxrwxrwx. 1 root root 33 Dec 11 13:15 /etc/localtime -> /usr/share/zoneinfo/Asia/Shanghai
[root@etl01 ~]# java Will
sun.util.calendar.ZoneInfo[id="Asia/Shanghai",offset=28800000,dstSavings=0,useDaylight=false,transitions=19,lastRule=null]
Asia/Shanghai

```


CentOS Linux release 7.0.1406 (Core)系统..........比较无语

