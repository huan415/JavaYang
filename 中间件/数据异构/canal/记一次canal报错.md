# 记一次canal报错（column size is not match for table）

## 背景

昨天晚上早早回去逗娃，正开心，项目经理突然来了电话，说测试环境Canal同步出问题了，由于隔天要部署上线，比较紧急。我看了群里面大家聊的报错截图，马上让运维改配置canal.instance.tsdb.enable=false（canal.properties和example/instance.properties两个文件都改），临时解决，今天分析问题，并记录一下。
如下图：很明显字段不一样，所以先去掉校验。

![](D:\project\huan415\ssh\JavaYang\中间件\数据异构\canal\assets\canal_err_1.png)

代码：

```java
 if (!existRDSNoPrimaryKey) {
                // online ddl增加字段操作步骤：
                // 1. 新增一张临时表，将需要做ddl表的数据全量导入
                // 2. 在老表上建立I/U/D的trigger，增量的将数据插入到临时表
                // 3. 锁住应用请求，将临时表rename为老表的名字，完成增加字段的操作
                // 尝试做一次reload，可能因为ddl没有正确解析，或者使用了类似online ddl的操作
                // 因为online ddl没有对应表名的alter语法，所以不会有clear cache的操作
                tableMeta = getTableMeta(event.getTable().getDbName(), event.getTable().getTableName(), false, position);// 强制重新获取一次
                if (tableMeta == null) {
                    tableError = true;
                    if (!filterTableError) {
                        throw new CanalParseException("not found [" + event.getTable().getDbName() + "."
                                                      + event.getTable().getTableName() + "] in db , pls check!");
                    }
                }

                // 在做一次判断
                if (tableMeta != null && columnInfo.length > tableMeta.getFields().size()) {
                    tableError = true;
                    if (!filterTableError) {
                        throw new CanalParseException("column size is not match for table:" + tableMeta.getFullName()
                                                      + "," + columnInfo.length + " vs " + tableMeta.getFields().size());
                    }
                }
                // } else {
                // logger.warn("[" + event.getTable().getDbName() + "." +
                // event.getTable().getTableName()
                // + "] is no primary key , skip alibaba_rds_row_id column");
            }
```

## 原因

分析原因是表结构变化，做了如下操作都不能复现：

1. 原来的表新增和删除字段
2. 表删掉再新增

复现过程：

1. 新建一张表
2. 再执行另一个sql语句（CREATE TABLE IF NOT EXISTS  ），表结构不一样-----------这一次的表字段数要比较少，这一次虽然执行无效，但是实际canal已经将表结构记录在内存里了
3. 执行修改语句。这时候canal发现这次变更的数据，结构跟内存里不一样，就报错，而且阻塞其他线程。

结论：
      原来有表了。又执行一次CREATE TABLE IF NOT EXISTS，而且字段数还比较少（字段增加没事）。这样的操作很奇怪，会不会我们的代码问题，存在这样的操作。
     跟业务开发确认，由于业务需求，执行两次创建表的sql语句（CREATE TABLE IF NOT EXISTS  ），而且两次的表结构不一样（有点奇葩，这个业务上的事儿，我在这里不多做解释了）。

```mysql
CREATE TABLE IF NOT EXISTS  `huan415_55557` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `f1` varchar(50) DEFAULT null,
	`f2` varchar(50) DEFAULT null,
	`f3` varchar(50) DEFAULT null,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8 ROW_FORMAT=COMPACT;
```

