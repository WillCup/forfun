title: 一步步搞定ETL依赖调度平台（二）-zakaban整合实施
date: 2016-06-16 11:22:25
tags: [ETL, 调度平台]
---

再列以下前面提到的步骤：
1. 通过azkaban api创建一个project: etl_20160615;
2. 调用java程序根据tbl_blood和etl内容生成azkaban的项目文件20160615.zip;
3. 通过azkaban api上传此文件到etl_20160615的project上
4. 通过azkaban api遍历etl_20160615项目下的所有flow，进行调度。

其实（一）里面是第二步的实现。今儿调研了以下azkaban的ajax api，并没有想象中的那么简单，所以就再单独说一下怎样完成其他步骤——重要的难点在于**session-id的获取**。

### 1. 方案与问题
|方案|问题|解决方案|
|---|---|---|
|使用azkaban api|需要session-id进行server端验证 |1. 使用phantomjs等模拟实现浏览器，从cookie中获取session-id——费事</br>2. 想办法使一个session-id永不过期(这个实施起来最简单，但是稳定程度有待测试)</br>3.azkaban web server启动了rest.li里的userManagerResource，login是返回session-id的，但是需要研习以下rest.li</br>找了半天，解决了:  curl -k -X POST --data "username=azkaban&password=azkaban&action=login" http://10.1.5.66:8001 |
|直接操作数据|需要对上传zip文件后生成的mysql表足够了解，包括存储的每个字段的内容与规范|将现有的项目中的内容弄出来验证规范后，开始实施|
|抽取操作中用到的azkaban的主要类|准备实例化这些类的成员变量|这些成员变量或许比直接插入mysql内容差不多难搞|

---

### 2. 实施

#### 2.1 获取session-id
鉴于时间因素，获取session-id的方案：
>curl -k -X POST --data "username=azkaban&password=azkaban&action=login" http://10.1.5.66:8001
返回值：
```json
{
  "status" : "success",
  "session.id" : "4e3da52b-fa1f-478d-836c-42122ced2341"
}
```
解析session-id的脚本
```bash
curl -k -X POST --data "username=azkaban&password=azkaban&action=login" http://10.1.5.66:8001 > etl_tmp
session_id=`cat etl_tmp | grep session | awk -F\" '!/[{}]/{print $(NF-1)}'`
echo "we got session id : $session_id"

```

#### 2.2  创建project
这里其实先安装了一个shell的json解析工具JSON.sh：https://github.com/dominictarr/JSON.sh
```bash
curl -k -X POST --data "session.id=${session_id}&name=${project_name}&description=${project_desc}" http://${host}/manager?action=create > ${etl_tmp}
status=`cat ${etl_tmp} | JSON.sh -b| grep status | awk '{print($2)}'`
if [ $status != '"success"' ]; then
        echo "error happends when creting project ${project_name}. status is ${status}"
fi

echo "project ${project_name} has been created successfully."
```

#### 2.3 上传zip文件
```bash
curl -k -i -H "Content-Type: multipart/mixed" -X POST --form "session.id=${session_id}" --form 'ajax=upload' --form "file=@${project_zip_file}" --form "project=${project_name}" http://${host}/manager?ajax=upload > ${etl_tmp}
status=`cat ${etl_tmp} | awk '{if(index($0,"error") > 0) print("error")}'`
if [ $status == "error" ]; then
        echo "error happends when uploading project ${project_name}. status is ${status}"
        exit 1
fi
echo "uploading project ${project_name} successfully"
```
ps: 没有继续使用JSON.sh的原因是解析报错...看来还是不健全

#### 2.4 执行flow
```bash
curl -k --get --data "session.id=${session_id}&ajax=fetchprojectflows&project=${project_name}" http://${host}/manager > ${etl_tmp}
for flow in `cat ${etl_tmp} |  JSON.sh -b | grep flowId |  awk '{gsub("\"","",$2);print($2)}'`
do
        echo "flow is $flow, ready to execute"
        curl -k --get --data "session.id=${session_id}" --data 'ajax=executeFlow' --data "project=${project_name}" --data "flow=${flow}" http://${host}/executor >${etl_tmp}${flow}
        echo "$flow execution done"
done
echo "all flows for project ${project_name} has been executed"
```


### 3.shell全文
```bash
#!/bin/bash
if [ !$# -eq 2]; then
        echo "useage : etl.sh datetime"
        exit
fi

date=$1
etl_tmp=/tmp/etl_tmp
project_name=etl_$datetime
host=10.1.5.66:8001
project_desc=daily_schedule
project_zip_file=/tmp/$datetime.zip

# 获取session id
curl -k -X POST --data "username=azkaban&password=azkaban&action=login" http://10.1.5.66:8001 > $etl_tmp
session_id=`cat ${etl_tmp} | grep session | awk -F\" '!/[{}]/{print $(NF-1)}'`
echo "we got session id : $session_id"

# 创建project
echo "project name is $project_name"
curl -k -X POST --data "session.id=${session_id}&name=${project_name}&description=${project_desc}" http://${host}/manager?action=create > ${etl_tmp}
status=`cat ${etl_tmp} | JSON.sh -b| grep status | awk '{print($2)}'` 
if [ $status != '"success"' ]; then
        echo "error happends when creting project ${project_name}. status is ${status}"
fi
echo "project ${project_name} has been created successfully."

# 上传zip文件到指定project
curl -k -i -H "Content-Type: multipart/mixed" -X POST --form "session.id=${session_id}" --form 'ajax=upload' --form "file=@${project_zip_file}" --form "project=${project_name}" http://${host}/manager?ajax=upload > ${etl_tmp}
status=`cat ${etl_tmp} | awk '{if(index($0,"error") > 0) print("error")}'`
if [ $status == "error" ]; then
        echo "error happends when uploading project ${project_name}. status is ${status}"
        exit 1
fi
echo "uploading project ${project_name} successfully"


# 遍历执行project下的flow
curl -k --get --data "session.id=${session_id}&ajax=fetchprojectflows&project=${project_name}" http://${host}/manager > ${etl_tmp}
for flow in `cat ${etl_tmp} |  JSON.sh -b | grep flowId |  awk '{gsub("\"","",$2);print($2)}'`
do
        echo "flow is $flow, ready to execute"
        curl -k --get --data "session.id=${session_id}" --data 'ajax=executeFlow' --data "project=${project_name}" --data "flow=${flow}" http://${host}/executor >${etl_tmp}${flow}echo "$flow execution done"
done
echo "all flows for project ${project_name} has been executed"
```
