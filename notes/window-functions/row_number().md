
# SQL学习笔记

## 窗口函数优化案例：查询复旦大学GPA第一名

### 题目理解
查询复旦大学中GPA最高的学生的GPA分数。

### 三种实现方式对比

#### 方式一：全局窗口函数 + 后过滤（最初想法）
```sql
WITH school_gpa AS (
    SELECT
        gpa,
        university,
        ROW_NUMBER() OVER (
            PARTITION BY university 
            ORDER BY gpa DESC
        ) AS rank_num
    FROM user_profile
)
SELECT gpa
FROM school_gpa
WHERE university = '复旦大学' AND rank_num = 1;
```
**执行逻辑：**
1. 对所有大学的所有学生按GPA排名
2. 过滤出复旦大学且排名第1的记录

#### 方式二：先过滤 + 窗口函数（理论更优）
```sql
WITH school_gpa AS (
    SELECT
        gpa,
        university,
        ROW_NUMBER() OVER (
            ORDER BY gpa DESC
        ) AS rank_num
    FROM user_profile
    WHERE university = '复旦大学'  -- 先过滤
)
SELECT gpa
FROM school_gpa
WHERE rank_num = 1;
```
**执行逻辑：**
1. 先过滤出复旦大学的学生
2. 只对过滤后的数据进行排序和编号
3. 取排名第1的记录

#### 方式三：简单聚合（开发最简单）
```sql
SELECT gpa
FROM user_profile
WHERE university = '复旦大学'
ORDER BY gpa DESC
LIMIT 1;
```
**执行逻辑：**
1. 过滤出复旦大学的学生
2. 按GPA降序排序
3. 返回第一条记录

### 性能对比测试

| 指标 | 方式一 | 方式二 | 方式三 |
|------|--------|--------|--------|
| 运行时间 | 32ms | 33ms | 34ms |
| 占用内存 | 6404KB | 6440KB | 6796KB |
| 代码复杂度 | 中等 | 中等 | 简单 |
| 理论性能 | 差 | 好 | 最好 |

### 为什么测试结果差异不大？

**原因分析：**
1. **数据量太小**：测试环境数据量不足以体现性能差异
2. **优化器智能**：数据库优化器可能将方式一、二优化为相同执行计划
3. **测量误差**：32-34ms的差异在正常波动范围内

**理论性能分析（大数据量场景）：**

| 数据规模 | 方式一 | 方式二 | 方式三 |
|----------|--------|--------|--------|
| 10万条（复旦1万） | ~50ms | ~20ms | ~15ms |
| 100万条（复旦10万）| ~500ms | ~80ms | ~60ms |
| 1000万条（复旦100万）| ~5000ms | ~300ms | ~200ms |

### 关键知识点

#### 1. 窗口函数执行顺序
```sql
-- 执行顺序：WHERE → 窗口函数 → SELECT
SELECT 
    ROW_NUMBER() OVER (ORDER BY gpa DESC) AS rank_num
FROM user_profile
WHERE university = '复旦大学'  -- 先执行
```
- **WHERE 先于窗口函数执行**
- 窗口函数只作用于过滤后的数据

#### 2. PARTITION BY 的使用场景
```sql
-- 场景1：需要多个大学的独立排名
ROW_NUMBER() OVER (PARTITION BY university ORDER BY gpa DESC)

-- 场景2：只需要单个大学的排名
ROW_NUMBER() OVER (ORDER BY gpa DESC)  -- 不需要 PARTITION BY
```

#### 3. 优化原则：先过滤，后处理
```sql
-- ❌ 低效：先处理所有数据
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (...) AS rn
    FROM large_table  -- 全表处理
) WHERE condition

-- ✅ 高效：先过滤数据
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (...) AS rn
    FROM large_table
    WHERE condition  -- 先过滤
)
```

### 最佳实践建议

#### 选择指南

| 场景 | 推荐方式 | 理由 |
|------|----------|------|
| 简单查询单个值 | 方式三（LIMIT） | 代码最简洁，性能最优 |
| 需要返回多条记录 | 方式二（先过滤） | 性能好，可扩展 |
| 需要多维度排名 | 方式一（PARTITION BY）| 功能强大，适合复杂需求 |

#### 开发建议
1. **小数据量**（<10万）：三种方式均可，选择最易读的
2. **大数据量**（>100万）：优先考虑方式三或方式二
3. **复杂业务逻辑**：使用CTE（方式一/二）提高可读性
4. **索引优化**：在 `(university, gpa DESC)` 上建立复合索引

#### 推荐索引
```sql
-- 提升查询性能
CREATE INDEX idx_uni_gpa ON user_profile(university, gpa DESC);

-- 覆盖索引（包含查询所需的所有列）
CREATE INDEX idx_uni_gpa_cover ON user_profile(university, gpa DESC, emp_no);
```

### 总结
- **理论最优**：方式二（先过滤后排序）
- **开发最优**：方式三（LIMIT简单直接）
- **性能差异**：大数据量下明显，小数据量可忽略
- **核心原则**：先过滤数据，减少窗口函数处理的数据量
```