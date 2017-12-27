
title: runningdatav2版本迁移计划
date: 2017-06-13 16:45:17
tags: [youdaonote]
---

#### 注意点
 - 编辑、更新
    - 各类ETL自动创建与之对应的ExecObj对象【重载save方法实现】
    - 个别自动分析依赖的ETL同时新建或更新相关ExecBlood【H2H, H2M】
    - 个别需要输入产出的ETL类型需要添加outputs字段，并在save的时候自创创建对应的NullETL对象【JAR】
    - 根据自身特性创建自身的rel_name 【save的时候分析并添加上】
 - 调度相关
    - (日月周)日常调度【完成依赖解析与任务文件生成，测试通过】
    - 定时调度【执行部分使用既有方案】
    - 临时调度【使用既有方案】
    - 新建调度时，针对的对象应该是修改为ExecObj


#### 影响范围
 - 整体过程中xstorm新建、编辑不可用
 - 任务调度中间可能出现失败

#### 实施步骤

1. 向各事业部发送不可用通知
2. 数据库备份
3. 确保新的save方法都已经注释掉,否则会影响清洗过程。
4. 将数据库对应model字段全部添加上
> ./manage.py migrate metamap 0052 --settings=metamap.config

可能需要手动添加exec_obj_id字段...【历史原因】
```sql
--
-- Add field exec_obj to anaetl
--
ALTER TABLE `metamap_anaetl` ADD COLUMN `exec_obj_id` integer NULL;
ALTER TABLE `metamap_anaetl` ALTER COLUMN `exec_obj_id` DROP DEFAULT;
--
-- Add field exec_obj to etl
--
ALTER TABLE `metamap_etl` ADD COLUMN `exec_obj_id` integer NULL;
ALTER TABLE `metamap_etl` ALTER COLUMN `exec_obj_id` DROP DEFAULT;
--
-- Add field exec_obj to jarapp
--
ALTER TABLE `metamap_jarapp` ADD COLUMN `exec_obj_id` integer NULL;
ALTER TABLE `metamap_jarapp` ALTER COLUMN `exec_obj_id` DROP DEFAULT;
--
-- Add field exec_obj to sqoophive2mysql
--
ALTER TABLE `metamap_sqoophive2mysql` ADD COLUMN `exec_obj_id` integer NULL;
ALTER TABLE `metamap_sqoophive2mysql` ALTER COLUMN `exec_obj_id` DROP DEFAULT;
--
-- Add field exec_obj to sqoopmysql2hive
--
ALTER TABLE `metamap_sqoopmysql2hive` ADD COLUMN `exec_obj_id` integer NULL;
ALTER TABLE `metamap_sqoopmysql2hive` ALTER COLUMN `exec_obj_id` DROP DEFAULT;
CREATE INDEX `metamap_anaetl_30a76f4d` ON `metamap_anaetl` (`exec_obj_id`);
ALTER TABLE `metamap_anaetl` ADD CONSTRAINT `metamap_anaetl_exec_obj_id_1bac03db_fk_metamap_execobj_id` FOREIGN KEY (`exec_obj_id`) REFERENCES `metamap_execobj` (`id`);
CREATE INDEX `metamap_etl_30a76f4d` ON `metamap_etl` (`exec_obj_id`);
ALTER TABLE `metamap_etl` ADD CONSTRAINT `metamap_etl_exec_obj_id_8c6a086a_fk_metamap_execobj_id` FOREIGN KEY (`exec_obj_id`) REFERENCES `metamap_execobj` (`id`);
CREATE INDEX `metamap_jarapp_30a76f4d` ON `metamap_jarapp` (`exec_obj_id`);
ALTER TABLE `metamap_jarapp` ADD CONSTRAINT `metamap_jarapp_exec_obj_id_6340f92e_fk_metamap_execobj_id` FOREIGN KEY (`exec_obj_id`) REFERENCES `metamap_execobj` (`id`);
CREATE INDEX `metamap_sqoophive2mysql_30a76f4d` ON `metamap_sqoophive2mysql` (`exec_obj_id`);
ALTER TABLE `metamap_sqoophive2mysql` ADD CONSTRAINT `metamap_sqoophive2mys_exec_obj_id_ee2cb5c2_fk_metamap_execobj_id` FOREIGN KEY (`exec_obj_id`) REFERENCES `metamap_execobj` (`id`);
CREATE INDEX `metamap_sqoopmysql2hive_30a76f4d` ON `metamap_sqoopmysql2hive` (`exec_obj_id`);
ALTER TABLE `metamap_sqoopmysql2hive` ADD CONSTRAINT `metamap_sqoopmysql2hi_exec_obj_id_9d61c4ff_fk_metamap_execobj_id` FOREIGN KEY (`exec_obj_id`) REFERENCES `metamap_execobj` (`id`);

```

5. 按照以下顺序开始清洗
- clean_etl 

    生成对应ETL的ExecObj
- clean_rel 

    清洗H2M,M2H中不规范的name，规范化hive表名到rel_name字段中
- clean_m2h 

    清洗M2H任务，生成对应的ExecObj。另外找到所有TBLBlood中父表是此表rel_name的记录，新建与之对应的ExecBlood记录
- before_clean_blood

    过滤一下TblBlood中的parent，对于不存在于H2H和M2好的父表，临时创建为之创建NULLETL，并创建对应ExecObj，然后再创建ExecBlood。 
    
    **==注意==**：由于NULLETL的特殊性，需要在此创建ExecObj的deptask的日月周调度【在save方法中已实现】
- clean_blood
    
    清洗TblBlood到对应ExecBlood，因为前面已经处理了所有情况，不该出现任务child或者parent不存在的问题。

    自检sql

    ```sql
    SELECT name from metamap_execobj GROUP BY name HAVING count(1) > 1;

    SELECT * from metamap_execobj where name = 'w_data_platform@drop_d_regula_stock';
    
    SELECT * from metamap_execobj where name = 'app_jlc@jlc_life_accumulative';
    ```
- clean_h2m
    
    清洗H2M到ExecObj对象，获取依赖ETL的H2H ExecObj对象，添加对应ExecBlood。
    如果是JAR产生的表，暂不支持，需要额外处理【JAR添加自身的output】。

    **==注意==**：
    H2M中的个别现象，多个周期的数据存放在同一个表中：
    SELECT * from metamap_sqoophive2mysql where id in (71,72,73,74,75,76)
    需要额外手动维护execblood
- clean_jar
    
    清洗JAR到ExecObj对象
- clean_email
    
    清洗ANAETL到ExecObj对象
- clean_task
    
    分别执行clean， type=4, type=1等(1,2,3,4,6)
    
    ```sql
    --- 这些都是ana_etl本身的valid是0
    SELECT * from metamap_willdependencytask where id in (864, 870, 759, 643, 164, 143, 85)
    ```
- clean_ptask
    
    更新定时任务的task参数，由原来的will_deptask的id，更新为type为100的最新清洗完的task id.
- clean_null

    为所有的NULLETL对象创建日月周的task调度

5. 测试各种任务调度【日常调度、定时调度、即时调度】
6. clean_exec_id

    清洗原有的ETL对象的exec_obj_id字段
    
6. 放开ETL对象的save方法注释
7. 恢复线上使用，开启beta测试，随时解决未发现的新生bug
8. 修改日调度脚本中的url到新版本
9. 添加调度的页面覆盖回来[sche/edit_new.html， /views/sche_etl_v2.py]


稳定运行一周，将dep task中旧的task全部删除



TODO
1. H2M的依赖手动添加以下。SELECT * from metamap_sqoophive2mysql where id in (71,72,73,74,75,76)
2. 测试各种ETL的save方法

新增逻辑：
1. 
