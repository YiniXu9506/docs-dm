---
title: 乐观模式下分库分表合并同步
summary: 介绍 DM 提供的乐观模式下分库分表的合并同步功能。
---

# 乐观模式下分库分表合并同步

本文介绍了 DM 提供的乐观模式下分库分表的合并同步功能，此功能可用于将上游 MySQL/MariaDB 实例中结构相同/不同的表同步到下游 TiDB 的同一个表中。

> **注意：**
>
> 在没有深入了解乐观模式的原理和使用限制的情况下不建议使用该模式，否则可能造成同步中断甚至数据不一致的严重后果。

## 背景

DM 支持在线上执行分库分表的 DDL 语句（通称 Sharding DDL），默认使用“悲观模式”，即当上游一个分表执行某一 DDL 后，这个分表的同步会暂停，等待其他所有分表都执行了同样的 DDL 才在下游执行该 DDL 并继续数据同步。这种“悲观协调”模式的优点是可以保证同步到下游的数据不会出错，缺点是会暂停数据同步而不利于对上游进行灰度变更。有些用户可能会花较长时间在单一分表执行 DDL，验证一定时间后才会更改其他分表的结构。在悲观模式同步的设定下，这些 DDL 会阻塞同步，binlog 事件会大量积压。

因此，需要提供一种新的“乐观协调”模式，在一个分表上执行的 DDL，自动修改成兼容其他分表的语句后，立即同步到下游，不会阻挡任何分表执行的 DML 的同步。

## 原理

在“乐观协调”模式下，DM-worker 接收到来自上游的 DDL 后，会把更新后的表结构转送给 DM-master。DM-worker 会追踪各分表当前的表结构，DM-master 合并成可兼容来自每个分表 DML 的合成结构，然后把与此对应的 DDL 同步到下游；对于 DML 会直接同步到下游。

![optimistic-ddl-flow](/media/optimistic-ddl-flow.png)

### 例子

例如上游 MySQL 有三个分表（`tbl00`, `tbl01` 以及 `tbl02`），使用 DM 同步到下游 TiDB 的 `tbl` 表中，如下图所示：

![optimistic-ddl-example-1](/media/optimistic-ddl-example-1.png)

在上游增加一列 `Level`：

```SQL
ALTER TABLE `tbl00` ADD COLUMN `Level` INT;
```

![optimistic-ddl-example-2](/media/optimistic-ddl-example-2.png)

此时下游 TiDB 要准备接受来自 `tbl00` 有 `Level` 的 DML、以及来自 `tbl01` 和 `tbl02` 没有 `Level` 的 DML。

![optimistic-ddl-example-3](/media/optimistic-ddl-example-3.png)

这时候如下的 DML 无需修改就可以同步到下游：

```SQL
UPDATE `tbl00` SET `Level` = 9 WHERE `ID` = 1;
INSERT INTO `tbl02` (`ID`, `Name`) VALUES (27, 'Tony');
```

![optimistic-ddl-example-4](/media/optimistic-ddl-example-4.png)

在 `tbl01` 同样增加一列 `Level`：

```SQL
ALTER TABLE `tbl01` ADD COLUMN `Level` INT;
```

![optimistic-ddl-example-5](/media/optimistic-ddl-example-5.png)

此时下游已经有相同的 Level 列了，所以 DM-master 比较表结构之后不做任何操作。

在 `tbl01` 刪除一列 `Name`：

```SQL
ALTER TABLE `tbl01` DROP COLUMN `Name`;
```

![optimistic-ddl-example-6](/media/optimistic-ddl-example-6.png)

此时下游仍需要接收来自 `tbl00` 和 `tbl02` 含 `Name` 的 DML 语句，因此不会立刻删除该列。

同样，各种 DML 仍可直接同步到下游：

```SQL
INSERT INTO `tbl01` (`ID`, `Level`) VALUES (15, 7);
UPDATE `tbl00` SET `Level` = 5 WHERE `ID` = 5;
```

![optimistic-ddl-example-7](/media/optimistic-ddl-example-7.png)

在 `tbl02` 增加一列 `Level`：

```SQL
ALTER TABLE `tbl02` ADD COLUMN `Level` INT;
```

![optimistic-ddl-example-8](/media/optimistic-ddl-example-8.png)

此时所有分表都已有 `Level` 列。

在 `tbl00` 和 `tbl02` 各刪除一列 `Name`：

```SQL
ALTER TABLE `tbl00` DROP COLUMN `Name`;
ALTER TABLE `tbl02` DROP COLUMN `Name`;
```

![optimistic-ddl-example-9](/media/optimistic-ddl-example-9.png)

到此步 `Name` 列也从所有分表消失了，所以可以安全从下游移除：

```SQL
ALTER TABLE `tbl` DROP COLUMN `Name`;
```

![optimistic-ddl-example-10](/media/optimistic-ddl-example-10.png)

## 风险 

使用乐观模式同步时，由于 DDL 会即时同步到下游，若使用不当，可能导致上下游数据不一致。

### 例子

例如以下三个分表合并同步到 TiDB：

![optimistic-ddl-fail-example-1](/media/optimistic-ddl-fail-example-1.png)

在 `tbl01` 新增一列 `Age`，默认值定为 `0`：

```SQL
ALTER TABLE `tbl01` ADD COLUMN `Age` INT DEFAULT 0;
```

![optimistic-ddl-fail-example-2](/media/optimistic-ddl-fail-example-2.png)

 在 `tbl00` 新增一列 `Age`，但默认值定为 `-1`：

```SQL 
ALTER TABLE `tbl00` ADD COLUMN `Age` INT DEFAULT -1;
```

![optimistic-ddl-fail-example-3](/media/optimistic-ddl-fail-example-3.png)

此时所有来自 `tbl00` 的 `Age` 都不一致了。这是由于 `DEFAULT 0` 和 `DEFAULT -1` 互不兼容。虽然 DM 遇到这种情况会报错，但上下游不一致的问题就需要手动去解决。

### 使数据不一致的操作

- 各分表的表结构不兼容，例：
    - 两个分表各自添加相同名称的列，但其类型不同。
    - 两个分表各自添加相同名称的列，但其默认值不同。
    - 两个分表各自添加相同名称的生成列，但其生成表达式不同。
    - 两个分表各自添加相同名称的索引，但其键组合不同。
    - 其他同名异构的情况。
- 在分表上执行对数据具有破坏性的 DDL，然后尝试回滚，例：
    - 刪除一列 X，之后又把 X 加回來。

### 使用限制

从上面的例子中可以看出，使用“乐观协调”模式有一定的风险，需要严格遵照以下方针：

- 执行每个批次的 DDL 前和后，要确保每个分表的结构达成一致。
- 进行灰度 DDL 时，只集中在一个分表上测试。
- 灰度完成后，在其他分表上尽量以最简单直接的 DDL 迁移到最终的 schema，而不要重放灰度测试中对或错的每一步。
    - 例如：在分表执行过 `ADD COLUMN A INT; DROP COLUMN A; ADD COLUMN A FLOAT;`，在其他分表直接执行 `ADD COLUMN A FLOAT` 即可，不需要三条 DDL 都执行一遍。
- 执行 DDL 时要注意观察 DM 同步状态。当同步报错时，需要判断这个批次的 DDL 是否会造成数据不一致。

此外，不论是使用“乐观协调”或“悲观协调”，DM 仍是有以下限制：

- 增量同步任务需要确保开始同步的 binlog position 对应的各分表的表结构必须一致。
- 进入 sharding group 的新表必须与其他成员的表结构一致（正在执行一个 DDL 批次时禁止 `CREATE/RENAME TABLE`）。
- 不支持 `DROP TABLE` / `DROP DATABASE`。
- TiDB 不支持的 DDL 语句在 DM 也不支持。
- 新增列的默认值不能包含 `current_timestamp`、`rand()`、`uuid()` 等，否则会造成上下游数据不一致。

## 乐观协调模式的配置

在任务的配置文件中指定 `shard-mode` 为 `optimistic` 则使用“乐观协调”模式，示例配置文件可以参考 [DM 任务完整配置文件介绍](task-configuration-file-full.md)。
