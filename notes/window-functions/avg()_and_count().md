# SQL学习笔记：窗口函数 vs GROUP BY 性能对比

## 题目理解
按性别和大学分组，统计每个组的学生人数、平均活跃天数、平均答题数。

## 两种实现方式对比

### 方式一：窗口函数 + DISTINCT(自己思考)
```sql
SELECT DISTINCT
    gender,
    university,
    COUNT(*) OVER (PARTITION BY university, gender) AS user_num,
    AVG(active_days_within_30) OVER (PARTITION BY university, gender) AS avg_active_day,
    AVG(question_cnt) OVER (PARTITION BY university, gender) AS avg_question_cnt
FROM user_profile
ORDER BY gender, university;
```

### 方式二：GROUP BY（传统聚合）
```sql
SELECT
    gender,
    university,
    COUNT(*) AS user_num,
    AVG(active_days_within_30) AS avg_active_day,
    AVG(question_cnt) AS avg_question_cnt
FROM user_profile
GROUP BY gender, university
ORDER BY gender, university;
```

## 性能测试结果

| 指标 | 方式一（窗口函数） | 方式二（GROUP BY） |
|------|-------------------|-------------------|
| 运行时间 | 36ms | 37ms |
| 占用内存 | 6520KB | 6516KB |
| 代码复杂度 | 中等 | 简单 |

**结论：两者性能几乎完全一致，差异在误差范围内。**

## 为什么性能差不多？

### 数据量因素
```
假设 user_profile 表：
- 总行数：~1000-5000 行（小数据量）
- 分组数：性别(2) × 大学(~10) = ~20组
- 每组平均：50-250行
```

### 执行计划分析

**方式一（窗口函数）：**
```
1. 全表扫描（N行）
2. 窗口函数计算分区聚合（N行，每行都计算）
3. DISTINCT 去重（约20行）
4. 排序输出

时间复杂度：O(N) + O(N) + O(20log20)
```

**方式二（GROUP BY）：**
```
1. 全表扫描（N行）
2. 哈希聚合（构建约20个分组）
3. 排序输出

时间复杂度：O(N) + O(20log20)
```

### 为什么窗口函数没有更慢？

| 阶段 | 方式一 | 方式二 | 差异 |
|------|--------|--------|------|
| 全表扫描 | ✅ N行 | ✅ N行 | 相同 |
| 聚合计算 | 窗口函数（N次） | GROUP BY（N次累加）| 基本相同 |
| 去重开销 | DISTINCT（20行） | 无 | 微小差异 |
| 临时表 | 可能有 | 可能有 | 相同 |

**核心原因：数据量小，额外开销可以忽略。**

## 大数据量下的理论差异

当数据量达到百万级别时，差异会显现：

| 数据规模 | 方式一（窗口函数） | 方式二（GROUP BY） | 差异倍数 |
|----------|-------------------|-------------------|----------|
| 10万行 | ~50ms | ~40ms | 1.25x |
| 100万行 | ~500ms | ~300ms | 1.67x |
| 1000万行 | ~6000ms | ~2500ms | 2.4x |

### 为什么大数据量下 GROUP BY 更快？

```sql
-- GROUP BY 的优化：
-- 1. 可以使用哈希聚合（内存中快速分组）
-- 2. 可以利用排序优化（排序后分组）
-- 3. 流式处理，不产生中间结果

-- 窗口函数的问题：
-- 1. 必须为每一行保留聚合值（N行输出）
-- 2. 然后通过 DISTINCT 去重（额外开销）
-- 3. 窗口函数通常需要临时表
```

## 执行计划对比

### 查看执行计划
```sql
-- MySQL
EXPLAIN SELECT ... GROUP BY ...;
EXPLAIN SELECT DISTINCT ... OVER(...);

-- PostgreSQL
EXPLAIN ANALYZE SELECT ...;
```

### 预期执行计划差异

**GROUP BY：**
```
-> Sort
   -> HashAggregate (分组聚合)
      -> Seq Scan (全表扫描)
```

**窗口函数：**
```
-> Unique (DISTINCT去重)
   -> Sort
      -> WindowAgg (窗口函数计算)
         -> Seq Scan (全表扫描)
```

## 各自的最佳使用场景

### GROUP BY 更合适的场景
```sql
-- 1. 只需要聚合结果，不需要明细
SELECT dept, AVG(salary) FROM emp GROUP BY dept;

-- 2. 多个聚合函数
SELECT 
    dept,
    COUNT(*),
    SUM(salary),
    AVG(salary),
    MAX(salary),
    MIN(salary)
FROM emp GROUP BY dept;

-- 3. 大数据量报表
-- GROUP BY 内存效率更高
```

### 窗口函数更合适的场景
```sql
-- 1. 需要同时查看明细和聚合
SELECT 
    name,
    dept,
    salary,
    AVG(salary) OVER (PARTITION BY dept) AS dept_avg
FROM emp;

-- 2. 排名和行号
SELECT 
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS rank,
    name,
    salary
FROM emp;

-- 3. 移动计算（累计、环比）
SELECT 
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) AS cumulative
FROM sales;
```

## 优化建议

### 1. 小数据量（<10万行）
```sql
-- 两种方式都可以，选择更易读的
-- 推荐：GROUP BY（语义更清晰）
SELECT gender, university, COUNT(*), AVG(...)
FROM user_profile
GROUP BY gender, university;
```

### 2. 大数据量（>100万行）
```sql
-- 优先使用 GROUP BY
-- 如果必须用窗口函数，考虑加索引
CREATE INDEX idx_gender_university ON user_profile(gender, university);
```

### 3. 需要同时输出明细和统计
```sql
-- 必须用窗口函数，无法用 GROUP BY 替代
SELECT 
    *,
    COUNT(*) OVER (PARTITION BY gender, university) AS group_count
FROM user_profile;
```

## 关键知识点

### 1. 窗口函数的执行顺序
```sql
FROM → WHERE → GROUP BY → HAVING → 窗口函数 → SELECT → ORDER BY → LIMIT
```

### 2. PARTITION BY 语法
```sql
-- ✅ 正确：逗号分隔
PARTITION BY column1, column2

-- ❌ 错误：不能用 AND
PARTITION BY column1 AND column2
```

### 3. DISTINCT 与窗口函数
```sql
-- 窗口函数不减少行数，每行都返回聚合值
-- 需要配合 DISTINCT 才能得到唯一分组
SELECT DISTINCT ... OVER(...)  -- 先扩展再去重，效率低
```

## 总结

| 对比维度 | GROUP BY | 窗口函数 + DISTINCT |
|----------|----------|---------------------|
| 语义清晰度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 小数据量性能 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 大数据量性能 | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| 内存占用 | 低 | 中高 |
| 代码简洁 | 简洁 | 略显冗余 |

**最佳实践：**
- **纯聚合需求** → 用 `GROUP BY`
- **明细+聚合需求** → 用窗口函数（不加 DISTINCT）
- **不要用窗口函数模拟 GROUP BY**（除非特殊原因）

**本次测试结论：**
在测试数据量下，两者性能几乎无差异。但 `GROUP BY` 语义更清晰、扩展性更好，是更优的选择。
