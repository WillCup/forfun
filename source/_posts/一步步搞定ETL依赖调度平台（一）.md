title: 一步步搞定ETL依赖调度平台（一）
date: 2016-06-15 18:48:43
tags: [ETL, 调度平台]
---

### 1. 依赖解析
- sql AST解析。相关工具antlr,这是一个定义与解析特定语言的工具。
- hive denpendency： explain dependency select * from a.
- 其他sql解析工具，功能类似hive dependency

鉴于我们的数据仓库使用hive搭建，使用hive dependency的兼容性更有保证。注意两点：1.如果a表不存在与hive中，就会报错找不到table。也就是说ETL开发的时候，它的依赖必须已经存在与hive库里；2. 由于依赖解析功能在ETL开发平台上，所以必须通过hiveserver2进行访问，效率可能不高，但是ETL平台对此要求也不会太高。

需要额外说一下antlr的好处：一是可以深入剖析sql，在这一层做字段级别的权限控制；二是可以做一些sql的分析，反馈给ETL开发者一些优化建议；


### 2. 存库
```sql

-- ----------------------------
-- Table structure for etl
-- ----------------------------
DROP TABLE IF EXISTS `etl`;
CREATE TABLE `etl` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `query` varchar(2000) DEFAULT NULL COMMENT 'sql语句',
  `meta` varchar(20) DEFAULT NULL COMMENT '数据库',
  `tbl_name` varchar(30) NOT NULL COMMENT '表名',
  `author` varchar(30) DEFAULT NULL COMMENT '创建人',
  `pre_sql` varchar(2000) DEFAULT NULL COMMENT '先执行的sql',
  `priority` int(5) NOT NULL DEFAULT '5',
  `on_schedule` tinyint(4) DEFAULT NULL COMMENT '0 不调度 1 调度中',
  `valid` tinyint(4) DEFAULT NULL,
  `ctime` bigint(20) DEFAULT NULL,
  `utime` bigint(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=19 DEFAULT CHARSET=utf8;


-- ----------------------------
-- Table structure for tbl_blood
-- ----------------------------
DROP TABLE IF EXISTS `tbl_blood`;
CREATE TABLE `tbl_blood` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `tbl_name` varchar(20) NOT NULL,
  `parent_tbl` varchar(20) NOT NULL,
  `related_etl_id` int(11) NOT NULL,
  `valid` tinyint(4) NOT NULL DEFAULT '0',
  `ctime` bigint(20) DEFAULT '0',
  `utime` bigint(20) DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=21 DEFAULT CHARSET=utf8;
```

每当新建一个ETL的时候都会分析它的依赖，如果分析失败，就不会插入ETL。

另外补充一个字段和表关系的，从hive meta库查询sql
```sql
SELECT
	db.DB_ID AS db_id,
	db.`NAME` AS db_name,
	a.TBL_ID AS tbl_id,
	a.TBL_NAME AS tbl_name,
	a.TBL_TYPE AS tbl_type,
	d.TYPE_NAME AS col_type_name,
	d.`COMMENT` AS col_comment,
	d.COLUMN_NAME AS column_name
FROM
	TBLS a
LEFT JOIN SDS b ON a.SD_ID = b.SD_ID
LEFT JOIN COLUMNS_V2 d ON b.CD_ID = d.CD_ID
LEFT JOIN DBS db ON a.DB_ID = db.DB_ID
```
---
### 3.每日调度
有了以上两步之后剩下的就是每日调度问题。分以下几个步骤：
1. 通过azkaban api创建一个project: etl_20160615;
2. 调用java程序根据tbl_blood和etl内容生成azkaban的项目文件20160615.zip;
3. 通过azkaban api上传此文件到etl_20160615的project上
4. 通过azkaban api遍历etl_20160615项目下的所有flow，进行调度。


生成azkaban文件的逻辑：
1. 找出没有被任何其他ETL以来的叶子节点
2. 遍历所有叶子节点，找到这个叶子节点的所有有效依赖，放入dependencies里，如果尚未生成该ETL的job文件
3. 向上递归执行，直到没有dependency的节点，也就是最上一层节点就停止。

```java
 public String generateAzkabanDAG() throws MetaException, ArchiveException {
        try {
            String serFolder = DateUtil.getTodayDateTime();
            List<TblBlood> leafBlood = tblBloodDao.selectAllLeaf();
            Set<String> doneBlood = new HashSet<String>();
            String serFolderLocation = AZKABAN_BASE_LOCATION + serFolder;
            if (leafBlood.size() > 0) {
                loadLeafBloods(leafBlood, doneBlood, serFolderLocation);
            }
            
            ZipUtils.addFilesToZip(new File(serFolderLocation), 
                    new File(serFolderLocation + ".zip"));
        } catch (IOException e) {
            log.error("error happends when generateAzkabanDAG");
            throw new MetaException("error happends when generateAzkabanDAG");
        }
        return null;
    }

    

    private void loadLeafBloods(List<TblBlood> leafBlood, Set<String> doneBlood, String serFolder) throws IOException {
        for (TblBlood leaf : leafBlood) {
            List<TblBlood> parentNode = tblBloodDao.selectParentByTblName(leaf.getTblName());
            if (!doneBlood.contains(leaf.getTblName())) {
                generateJobFile(leaf, parentNode, serFolder);
                doneBlood.add(leaf.getTblName());
            }
            loadLeafBloods(parentNode, doneBlood, serFolder);
        }
    }


    private void generateJobFile(TblBlood currentBlood,  List<TblBlood> parentNode, String serFolder) throws IOException {

        String jobName = currentBlood.getTblName();
        String jobType = "command";
        String command = "echo 'I am command for ' " + jobName;
        String content;
        Set<String> dependencies = new HashSet<String>();
        for (TblBlood blood : parentNode) {
            dependencies.add(blood.getTblName());
        }
        String jobDependencies = joiner.join(dependencies);
        content = "# " + jobName + "\n"
                + "type=" + jobType +"\n"
                        + "command=" + command + "\n";
        if (StringUtils.isNotBlank(jobDependencies)) {
            content += "dependencies=" + jobDependencies +"\n";
        }
        File file = new File(serFolder + "/" + jobName.replace("@", "-") + ".job");
        FileUtils.write(file, content, "utf8", false);
    }
```


### 4.参考
http://azkaban.github.io/azkaban/docs/latest/#ajax-api

### 5. TODO

- flow和job目前都是不支持优先级的，需要探查下azkaban中同一个flow里job执行次序是怎样的。
- 目前字段解析只能通过部分从hive meta抽，获取到字段与当前表的关系，不能逐个找到字段与字段之间的血统关系。如果使用antlr的话可以分析到每一个as之前是什么，理论上是可以实现字段的血统关系的。
- 对于每月、每周、每小时的个别ETL的支持策略
- 支持ETL变量传入和临时调试
