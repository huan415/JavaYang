# ALTER报唯一键冲突

## 背景

早上部署生产环境，一来上班同事就找我说sql执行报错，虽然不是我的sql（ALTER TABLE ** ADD COLUMN），但是涉及到的表update非常频繁（update是因为数据同步，而数据同步由我负责），所以就拉我一起看了。

## 原因

alter的表更新非常频繁。由于在部署生产，比较紧急，快速解决方案（不是很好的方案）

1. 停服务，这样就不会频繁update了
2. 锁表，LOCK TABLES ***    UNLOCK TABLES;

以上两种方式都会造成服务不可用（慎用）

## 

参考：

1. https://www.cnblogs.com/wangb2/p/14252609.html
2. https://www.cnblogs.com/igoodful/p/14075649.html