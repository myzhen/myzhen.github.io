---
layout: post
title: pglogical部署文档
categories: extension
description: pglogical extension
keywords: pg-extension, pgxs
---

### 安装

|  **节点**  |     **IP**     |
| :--------: | :------------: |
|  Provider  | 192.168.30.150 |
| Subscriber | 192.168.30.160 |

数据库在initdb时通过-m参数指定初始化一个兼容Postgres或兼容Oracle的集群。pglogical针对这两种模式的数据库集群在配置相同，下面以Oraccle模式为例显示如何使用pglogical扩展。

1. 数据库部署

   在192.168.30.150和192.168.30.160两个节点部署数据库集群，并初始化为Oracle兼容模式，分别表示发布者节点和订阅者节点。

2. 安装插件

   需要在两个节点上都执行CREATE EXTENSION pglogical来加载插件。因此，两个节点均需执行如下操作：

   首先，将已编译好的pglogical包中的文件放到Postgres中相应目录结构中，每个文件对应的位置与其他插件相同：

   |        **位置目录**         |             **文件**              |
   | :-------------------------: | :-------------------------------: |
   |       lib/postgresql/       | pglogical.so、pglogical_output.so |
   | share/postgresql/extension/ |     control文件、sql脚本文件      |
   |            bin/             |    pglogical_create_subscriber    |
   
   然后，修改Postgres服务器的配置文件postgresql.conf以支持逻辑解码，*注意，下面的是Oracle模式例子，在Postgres模式下shared_preload_libraries参数没有libparser_oracle库。*

- 配置postgresql.conf

  ```shell
  shared_preload_libraries = 'libparser_oracle, pglogical'
  listen_addresses = '*'
  wal_level = logical
  max_worker_processes = 10
  max_replication_slots = 10
  max_wal_senders = 10
  ```

  pg_hba.conf 配置以允许复制连接。

- 配置pg_hba.conf

  ```shell
  host    all             all             192.168.30.0/0            trust
  host    replication     all             192.168.30.0/0            trust
  ```

- 创建扩展

  ```sql
  create extension pglogical;
  ```

  分别在发布和订阅节点执行上述配置操作。

### Proivder 节点

1. 创建发布者节点

   ```sql
   SELECT pglogical.create_node(node_name := 'test-provider',dsn := 'host=192.168.30.150 port=5432 user=sys dbname=postgres');
   ```

2. 创建复制集

   ```sql
   SELECT pglogical.create_replication_set (
          set_name := 'test_replica_set',
          replicate_insert := true,
          replicate_update := true,
          replicate_delete := true,
          replicate_truncate := false);
   ```

3. 创建测试表

   ```sql
   create table my_test(col1 int primary key, col2 text);
   ```

4. 将表添加到复制集

   ```sql
   SELECT pglogical.replication_set_add_all_tables (
          set_name := 'test_replica_set',
          schema_names := ARRAY['public'],
          synchronize_data := true);
   ```

### Subscriber 节点

1. 创建订阅者节点

   ```sql
   SELECT pglogical.create_node(node_name := 'test-subscriber',dsn := 'host=192.168.30.160 port=5432 user=sys dbname=postgres');
   ```

2. 在订阅者节点手动创建与布者节点完全相同的表结构

   ```sql
   create table my_test(col1 int primary key, col2 text);
   ```

3. 创建订阅

   ```sql
   SELECT pglogical.create_subscription (
            subscription_name := 'test_subscription',
            provider_dsn := 'host=192.168.30.150 port=5432 user=sys dbname=postgres',
            replication_sets := ARRAY['test_replica_set']);
   ```
   
   更方便的，可以将上面步骤2和3合并为一个步骤，即在执行create_subscription创建订阅时通过指定参数synchronize_structure := true来自动同步发布者节点的表结构，亦为DLL同步。
   
   ```sql
   SELECT pglogical.create_subscription (
            subscription_name := 'test_subscription',
            provider_dsn := 'host=192.168.30.150 port=5432 user=sys dbname=postgres',
            replication_sets := ARRAY['test_replica_set'], synchronize_structure := true);
   ```

### 测试

```sql
insert into my_test select generate_series(1,100),'test';
update my_test set col2 = 'modified';
delete from my_test where col1 > 50;
```

在proivder节点执行上述DML命令，然后在Subscriber节点上查看逻辑复制的结果。

