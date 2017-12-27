
title: R
date: 2017-01-09 10:35:22
tags: [youdaonote]
---

```
library(SparkR)
library(tseries)
library(forecast)
library(magrittr)

sc<-sparkR.init()
hivecontext<-sparkRHive.init(sc)
sqlcontext<-sparkRSQL.init(sc)

dateline<-seq.Date(from = as.Date("2015-05-01"),to=as.Date("2017-01-04"),by="day")%>%data.frame(.)
colnames(dateline)<-"log_day"
dateline<-createDataFrame(sqlContext=sqlcontext,data=dateline)
registerTempTable(dateline,"premium_date")
  

#  Query the premium log of the users who are still remained for 30d and not labeled as brush.
z0<-"select	a.userid,
		          from_unixtime(unix_timestamp(c.log_day,'yyyyMMdd'),'yyyy-MM-dd') as log_day,
              c.premium_final
from dim_jlc.dim_user_info a
left join app_jlc.brush_user b on a.userid=b.userid and b.log_day='2017-01-04'
left join fact_jlc.premium_record c on a.userid=c.userid
where b.userid is null and datediff('2017-01-04',to_date(a.invest1st_time))>30"

z1<-"select	a.userid,
		to_date(a.invest1st_time) as invest1st_day
from dim_jlc.dim_user_info a
left join app_jlc.brush_user b on a.userid=b.userid
where b.userid is null and datediff('2017-01-04',to_date(a.invest1st_time))>30"

result<-sql(hivecontext,z0)
userinfo<-sql(hivecontext,z1)
colnames(result)<-c("userid","log_day","premium")
cache(result)
cache(userinfo)

user_num<-dim(userinfo)[1]
for(i in 1:user_num){
  userid<-
  start_day<-select(userinfo,"invest1st_day")
  dateline<-seq()
}



#rawdata<-fread("raw0104.csv")
#rawdata_df<-tbl_df(rawdata)
#colnames(rawdata_df)<-c("userid","log_day","premium")

userinfo_df<-collect(userinfo)
userinfo_df<-tbl_df(userinfo_df)

```
