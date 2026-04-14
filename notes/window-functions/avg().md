```markdown
# SQL学习笔记：窗口函数 vs GROUP BY 性能对比

## 题目理解
查询大学中平均答题数小于5或平均回答数小于20的大学，输出大学名称及对应的平均值。

## 两种实现方式对比

### 方式一：窗口函数 + 子查询
```sql
SELECT *
FROM (
    SELECT DISTINCT
        university,
        AVG(question_cnt) OVER (PARTITION BY university) AS avg_question_cnt,
        AVG(answer_cnt) OVER (PARTITION BY university) AS avg_answer_cnt
    FROM user_profile
) AS t
WHERE avg_question_cnt < 5 OR avg_answer_cnt < 20;
```

### 方式二：GROUP BY + HAVING
```sql
SELECT
    university,
    AVG(question_cnt) AS avg_question_cnt,
    AVG(answer_cnt) AS avg_answer_cnt
FROM user_profile
GROUP BY university
HAVING AVG(question_cnt) < 5 OR AVG(answer_cnt) < 20;
```

## 性能测试结果

| 指标 | 方式一（窗口函数） | 方式二（GROUP BY） |
|------|-------------------|-------------------|
| 运行时间 | 33ms | 33ms |
| 占用内存 | 6544KB | 6520KB |
| 代码复杂度 | 中等 | 简单 |

**结论：两者性能完全一致，GROUP BY 代码更简洁。**

## 重要发现：HAVING 中使用别名的性能陷阱

### ❌ 错误写法（性能较差）
```sql
SELECT
    university,
    AVG(question_cnt) AS q,
    AVG(answer_cnt) AS a
FROM user_profile
GROUP BY university
HAVING q < 5 OR a < 20;  -- 使用别名过滤
-- 运行时间：47ms
```

### ✅ 正确写法（性能更好）
```sql
SELECT
    university,
    AVG(question_cnt) AS q,
    AVG(answer_cnt) AS a
FROM user_profile
GROUP BY university
HAVING AVG(question_cnt) < 5 OR AVG(answer_cnt) < 20;  -- 直接使用聚合函数
-- 运行时间：33ms
```

### 为什么 HAVING 中使用别名会变慢？

**SQL 执行顺序：**
```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
```

- `HAVING` 在 `SELECT` **之前**执行
- 别名（`q`、`a`）在 `SELECT` 阶段才定义
- 使用别名时，数据库需要额外处理映射关系，导致性能下降

**正确做法：**
- `HAVING` 中使用聚合函数（`AVG(...)`、`COUNT(...)` 等）
- `ORDER BY` 中可以使用别名（因为它在 SELECT 之后执行）

## 三种写法完整对比

| 写法 | 运行时间 | 是否推荐 | 说明 |
|------|----------|----------|------|
| 窗口函数 | 33ms | ⭐⭐⭐ | 功能强大，适合复杂场景 |
| GROUP BY（正确） | 33ms | ⭐⭐⭐⭐⭐ | 简洁高效，首选 |
| GROUP BY（别名） | 47ms | ❌ | 性能差，不推荐 |

## 关键知识点

### 1. 窗口函数 vs GROUP BY 选择原则
```sql
-- 只需要聚合结果 → GROUP BY
SELECT university, AVG(question_cnt)
FROM user_profile
GROUP BY university;

-- 需要明细 + 聚合 → 窗口函数
SELECT 
    name,
    university,
    question_cnt,
    AVG(question_cnt) OVER (PARTITION BY university) AS uni_avg
FROM user_profile;
```

### 2. HAVING 正确用法
```sql
-- ✅ 正确：使用聚合函数
HAVING AVG(question_cnt) < 5

-- ✅ 正确：使用 GROUP BY 中的列
HAVING university = '北京大学'  -- 但建议放 WHERE

-- ❌ 错误：使用别名
HAVING avg_q < 5

-- ❌ 错误：使用非聚合、非分组列
HAVING name = '张三'  -- name 不在 GROUP BY 中
```

### 3. 别名使用时机
```sql
-- WHERE 中：不能用别名
WHERE AVG(question_cnt) < 5  -- ❌ 聚合函数不能在 WHERE

-- HAVING 中：不能用别名（性能差）
HAVING AVG(question_cnt) < 5  -- ✅ 正确

-- ORDER BY 中：可以用别名
ORDER BY avg_question_cnt  -- ✅ 正确（SELECT 之后执行）

-- SELECT 中：可以用别名
SELECT AVG(question_cnt) AS avg_q  -- ✅ 正确
```

## 最佳实践总结

```sql
-- 推荐写法（清晰且高效）
SELECT
    university,
    AVG(question_cnt) AS avg_question_cnt,
    AVG(answer_cnt) AS avg_answer_cnt
FROM user_profile
GROUP BY university
HAVING AVG(question_cnt) < 5 OR AVG(answer_cnt) < 20
ORDER BY avg_question_cnt DESC;  -- ORDER BY 中可用别名
```

**核心要点：**
1. 纯聚合需求 → 用 `GROUP BY`，简洁高效
2. `HAVING` 中直接写聚合函数，不要用别名
3. 需要明细+聚合 → 用窗口函数
4. 大数据量下，`GROUP BY` 内存友好，性能更稳定
```