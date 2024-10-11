---
title: "乐观锁与悲观锁"
description: ""
summary: ""
date: 2024-10-11T15:00:00+08:00
lastmod: 2024-10-11T15:00:00+08:00
weight: 300
seo:
  title: "乐观锁与悲观锁"
  description: ""
  canonical: ""
  noindex: false
---

## 简介

乐观锁和悲观锁都是处理并发访问时常用的锁机制，主要用于数据库或多线程编程中，来防止数据不一致问题。
它们的区别在于处理并发冲突的策略不同，因此适用于不同的场景。

## 乐观锁

乐观锁（Optimistic Locking）虽然名字叫锁，但实际上它并不加锁。
而是通过版本号机制或时间戳机制，在提交时检查资源是否被其他事务修改。
若数据没有被修改，提交操作正常进行，否则，抛出异常或返回错误。

### 适用场景

* 读多写少的场景：例如社交网络、用户资料管理等，数据的读取远多于更新。
* 非关键事务：如电商网站的商品详情页，用户查看商品信息时不需要锁定商品数据。
* 高并发环境：如缓存系统、消息队列等，更新冲突的可能性低。

### 实现示例

更新用户资料，如果更新行数为 0，说明版本号不匹配，操作失败。

```sql {frame="none"}
UPDATE users SET name = 'new_name', version = version + 1 
WHERE id = 1 AND version = 1;
```

## 悲观锁

悲观锁（Pessimistic Locking）假设访问资源时会发生冲突。
因此在读取或修改资源之前，直接锁定资源，其他线程或事务在锁释放之前无法对该资源进行操作。
锁可以分为读锁（共享锁）和写锁（排他锁）。

### 适用场景

* 写多读少的场景：如银行转账、库存管理等，数据频繁更新，必须确保数据的一致性。
* 关键事务：涉及资金、重要操作的系统中，必须确保在操作时数据不被其他事务修改。
* 竞争激烈的环境：如秒杀、抢购活动，必须严格控制对数据的并发访问。

### 实现示例

使用 MySQL 事务更新库存。

```sql {frame="none"}
START TRANSACTION;
SELECT quantity FROM inventory WHERE product_id = 1 FOR UPDATE;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 1;
COMMIT;
```

## 优缺点对比

| 特性       | 乐观锁                       | 悲观锁                       |
|------------|------------------------------|------------------------------|
| 适用场景   | 读多写少，冲突概率低         | 写多读少，冲突概率高         |
| 锁机制     | 使用版本号或时间戳           | 使用数据库行级或表级锁       |
| 性能       | 通常性能较好，减少锁竞争     | 性能较低，可能导致资源浪费   |
| 数据一致性 | 通过重试实现                 | 通过加锁直接确保一致性       |
| 并发控制   | 重试机制                     | 锁机制                       |

## 场景分析

假设我们有一个产品管理系统，允许多个管理员编辑产品信息。
为了防止两个管理员同时修改同一个产品的信息，我们可以使用乐观锁或悲观锁。

### 乐观锁

产品表中添加配置版本号字段，提交修改时检查版本号是否一致。
如果一致则更新，不一致则提示用户数据已被修改，请刷新页面重新操作。

* 优点：实现简单，无锁操作。
* 缺点：冲突时用户需要重新配置。

### 悲观锁

用户点击编辑的时候锁定数据，直到提交修改，再释放锁。
另一个用户点击编辑时，发现数据被锁定，只能等待上个用户提交。

* 优点：冲突时提前感知，不做无用功。
* 缺点：需要多写代码和防止锁长时间未释放。

产品表结构如下。

```sql {frame="none"}
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10, 2),
    description TEXT,
    locked_by INT NULL,
    locked_at DATETIME NULL  -- 锁定的时间
);
```

程序中事务的执行逻辑如下。

```sql {frame="none"}
-- 用户请求编辑产品ID为123的产品信息
BEGIN;  -- 开始事务

-- 查询产品信息并加锁，锁定期间其他用户无法修改该产品
SELECT * FROM products WHERE id = 123 FOR UPDATE;

-- 如果 locked_by 不为空，则表示产品已被其他用户锁定

-- 如果 locked_by 为空，则记录锁定信息，表示当前产品被用户锁定
UPDATE products SET locked_by = 1 WHERE id = 123;

COMMIT;  -- 提交事务
```

成功获取锁之后，可提交修改产品信息。

```sql {frame="none"}
BEGIN;  -- 开始事务

-- 更新产品信息
UPDATE products SET 
    name = 'New Product Name',
    price = 99.99,
    description = 'Updated product description',
    locked_by = NULL  -- 解除锁定
WHERE id = 123;

COMMIT;  -- 提交事务
```

定时处理长时间未释放的锁。

```sql {frame="none"}
-- 超过30分钟的锁自动解锁
UPDATE products 
SET locked_by = NULL, locked_at = NULL
WHERE locked_at < NOW() - INTERVAL 30 MINUTE;
```

后端获取锁时的伪代码如下。

```go {frame="none"}
func editProduct(userId int, productId int) error {
    // 开始事务
    tx, err := db.Begin()
    if err != nil {
        return err
    }

    defer tx.Rollback()

    // 尝试加锁
    _, err = tx.Exec("SELECT * FROM products WHERE id = ? FOR UPDATE", productId)
    if err != nil {
        return err  // 如果锁定失败或超时，返回错误
    }

    // 检查 locked_by 字段，判断是否已被锁定
    var lockedBy int
    err = tx.QueryRow("SELECT locked_by FROM products WHERE id = ?", productId).Scan(&lockedBy)
    if err != nil {
        return err
    }

    if lockedBy != 0 && lockedBy != userId {
        return fmt.Errorf("该产品正在被用户 %d 编辑", lockedBy)
    }

    // 锁定成功，允许用户编辑
    _, err = tx.Exec("UPDATE products SET locked_by = ? WHERE id = ?", userId, productId)
    if err != nil {
        return err
    }

    // 提交事务
    return tx.Commit()
}
```
