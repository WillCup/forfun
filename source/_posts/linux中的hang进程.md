
title: linux中的hang进程
date: 2017-05-25 11:10:38
tags: [youdaonote]
---

5月24号晚上发现62上有两个hang的进程，占满了内存与cpu


第一个是M2H任务
```
root     10082     1 97 May21 ?        3-20:08:33 /server/java/jdk1.8.0_60/bin/java -Xmx4096m -Dhdp.version=2.4.2.0-258 -Djava.net.preferIPv4Stack=true -Dhdp.version=2.4.2.0-258 -Dhadoop.log.dir=/var/log/hadoop/root -Dhadoop.log.file=hadoop.log -Dhadoop.home.dir=/usr/hdp/2.4.2.0-258/hadoop -Dhadoop.id.str=root -Dhadoop.root.logger=INFO,console -Djava.library.path=:/usr/hdp/2.4.2.0-258/hadoop/lib/native/Linux-amd64-64:/usr/hdp/2.4.2.0-258/hadoop/lib/native -Dhadoop.policy.file=hadoop-policy.xml -Djava.net.preferIPv4Stack=true -Xmx4096m -Dhadoop.security.logger=INFO,NullAppender org.apache.sqoop.Sqoop import -Dmapreduce.job.queuename=xstorm --connect jdbc:mysql://10.0.100.73:3306/xiaodai?useCursorFetch=true&dontTrackOpenResources=true&defaultFetchSize=2000 --driver com.mysql.jdbc.Driver --username weixddata_read --password G7iu1BMo9LWs2e --hive-database ods_tinyv --columns ID,USER_ID,WB_USERNAME,MOBILE,BIZ_ID,MSG_TYPE,SMS_TYPE,DATA_SOURCE,SMS_ID,TASK_ID,SEND_STATUS,RES_CODE,REPORT_STATUS,SEND_TIME,REPORT_TIME,SEND_EXCEPTION --where DATE_FORMAT(SEND_TIME, '%Y-%m-%d')='2017-05-20' --table SMS_INFO --hive-import --hive-overwrite --target-dir ods_tinyv_SMS_INFO --outdir /server/app/sqoop/vo --bindir /server/app/sqoop/vo --verbose -m 1 --delete-target-dir --hive-import --hive-table o_wb_xiaodai_sms_info_i --hive-partition-key dt --hive-partition-value 2017-05-20 --null-string \\N --null-non-string \\N
```

还有一个HIVE任务
```
root      3483     1 96 May23 ?        1-11:12:07 /server/java/jdk1.8.0_60/bin/java -Xmx4096m -Dhdp.version=2.4.2.0-258 -Djava.net.preferIPv4Stack=true -Dhdp.version=2.4.2.0-258 -Djava.net.preferIPv4Stack=true -XX:NewRatio=12 -XX:MaxHeapFreeRatio=40 -XX:MinHeapFreeRatio=15 -XX:+UseNUMA -XX:+UseParallelGC -XX:-UseGCOverheadLimit -Dhdp.version=2.4.2.0-258 -Dhadoop.log.dir=/var/log/hadoop/root -Dhadoop.log.file=hadoop.log -Dhadoop.home.dir=/usr/hdp/2.4.2.0-258/hadoop -Dhadoop.id.str=root -Dhadoop.root.logger=INFO,console -Djava.library.path=:/usr/hdp/2.4.2.0-258/hadoop/lib/native/Linux-amd64-64:/usr/hdp/2.4.2.0-258/hadoop/lib/native -Dhadoop.policy.file=hadoop-policy.xml -Djava.net.preferIPv4Stack=true -Xmx4096m -Xmx1024m -Dhadoop.security.logger=INFO,NullAppender -Dhdp.version=2.4.2.0-258 -Dhadoop.log.dir=/var/log/hadoop/root -Dhadoop.log.file=hadoop.log -Dhadoop.home.dir=/usr/hdp/2.4.2.0-258/hadoop -Dhadoop.id.str=root -Dhadoop.root.logger=INFO,console -Djava.library.path=:/usr/hdp/2.4.2.0-258/hadoop/lib/native/Linux-amd64-64:/usr/hdp/2.4.2.0-258/hadoop/lib/native:/usr/hdp/2.4.2.0-258/hadoop/lib/native/Linux-amd64-64:/usr/hdp/2.4.2.0-258/hadoop/lib/native -Dhadoop.policy.file=hadoop-policy.xml -Djava.net.preferIPv4Stack=true -Xmx4096m -Xmx4096m -Xmx1024m -Dhadoop.security.logger=INFO,NullAppender org.apache.hadoop.util.RunJar /usr/hdp/2.4.2.0-258/hive/lib/hive-exec-1.2.1000.2.4.2.0-258.jar org.apache.hadoop.hive.ql.exec.mr.ExecDriver -localtask -plan file:/tmp/root/228f95e6-26ee-4466-bd3d-a91195c86bf4/hive_2017-05-23_10-38-11_000_7465740175512782543-1/-local-10004/plan.xml -jobconffile file:/tmp/root/228f95e6-26ee-4466-bd3d-a91195c86bf4/hive_2017-05-23_10-38-11_000_7465740175512782543-1/-local-10005/jobconf.xml

```

手贱，先kill掉了，不然还能看到上面的plan文件，和job配置文件什么的。


