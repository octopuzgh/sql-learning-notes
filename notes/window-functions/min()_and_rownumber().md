
# SQL学习笔记：查询每个大学最低分学生的三种方法对比

## 题目理解

查询每个大学中 GPA 最低的学生信息（device_id, university, gpa），按大学升序排序。

**示例数据**：
| device_id | university | gpa |
|-----------|------------|-----|
| 2138 | 北京大学 | 3.4 |
| 6543 | 北京大学 | 3.2 |
| 3214 | 复旦大学 | 4.0 |
| 4321 | 复旦大学 | 3.6 |
| 5432 | 山东大学 | 3.8 |
| 2131 | 山东大学 | 3.3 |
| 2315 | 浙江大学 | 3.6 |

**预期输出**：
| device_id | university | gpa |
|-----------|------------|-----|
| 6543 | 北京大学 | 3.2 |
| 4321 | 复旦大学 | 3.6 |
| 2131 | 山东大学 | 3.3 |
| 2315 | 浙江大学 | 3.6 |


## 二、三种实现方式

### 方式一：MIN 窗口函数
```sql
SELECT
    device_id,
    university,
    gpa
FROM (
    SELECT
        device_id,
        university,
        gpa,
        MIN(gpa) OVER (PARTITION BY university) AS min_gpa
    FROM user_profile
) t
WHERE gpa = min_gpa
ORDER BY university;
```

### 方式二：ROW_NUMBER 窗口函数
```sql
SELECT
    device_id,
    university,
    gpa
FROM (
    SELECT
        device_id,
        university,
        gpa,
        ROW_NUMBER() OVER (PARTITION BY university ORDER BY gpa) AS rn
    FROM user_profile
) t
WHERE rn = 1
ORDER BY university;
```

### 方式三：子查询 + IN
```sql
SELECT
    device_id,
    university,
    gpa
FROM user_profile
WHERE (university, gpa) IN (
    SELECT university, MIN(gpa)
    FROM user_profile
    GROUP BY university
)
ORDER BY university;
```


## 三、性能测试结果

| 方式 | 运行时间 | 占用内存 | 说明 |
|------|---------|---------|------|
| MIN 窗口函数 | 34ms | 6520KB | ✅ 最快 |
| ROW_NUMBER 窗口函数 | 37ms | 6520KB | 稍慢 |
| 子查询 + IN | 34ms | 6528KB | ✅ 最快 |


## 四、三种方式对比分析

| 对比维度 | MIN 窗口函数 | ROW_NUMBER | 子查询 + IN |
|---------|-------------|------------|-------------|
| **代码复杂度** | 中等 | 中等 | 简单 |
| **逻辑清晰度** | 直观（分数等于最低分） | 通用（排名取第一） | 最直观 |
| **并列最低分处理** | 全部取出 | 只取一条 | 全部取出 |
| **可移植性** | 好 | 好 | 最好 |
| **本题适用性** | ✅ 符合 | ✅ 符合 | ✅ 符合 |


## 五、并列最低分场景对比

假设北京大学有两条 GPA 3.2 的记录：

| 方式 | 结果 |
|------|------|
| MIN 窗口函数 | 返回 **2 条**（所有最低分） |
| ROW_NUMBER | 返回 **1 条**（只取一条） |
| 子查询 + IN | 返回 **2 条**（所有最低分） |

**本题示例输出每个学校只有一条，说明即使有并列也只取一条**，因此：

| 方式 | 是否符合题意 |
|------|-------------|
| MIN 窗口函数 | ⚠️ 如果有并列会多取 |
| ROW_NUMBER | ✅ 严格符合 |
| 子查询 + IN | ⚠️ 如果有并列会多取 |


## 六、三种方式执行逻辑

### 方式一：MIN 窗口函数
```
1. 计算每个大学的最低 GPA（窗口函数）
2. 筛选 GPA = 最低分的记录
3. 按大学排序
```

### 方式二：ROW_NUMBER
```
1. 按大学分组，组内按 GPA 升序编号（最低分编号为 1）
2. 筛选编号为 1 的记录
3. 按大学排序
```

### 方式三：子查询 + IN
```
1. 子查询：计算每个大学的最低 GPA
2. 主查询：筛选 (university, gpa) 在子查询结果中的记录
3. 按大学排序
```


## 七、优缺点总结

### MIN 窗口函数
| 优点 | 缺点 |
|------|------|
| 逻辑直观（分数等于最低分） | 并列时会全部取出 |
| 性能好 | 需要 DISTINCT 配合（否则重复） |

### ROW_NUMBER 窗口函数
| 优点 | 缺点 |
|------|------|
| 精确控制取几条 | 代码略复杂 |
| 可灵活调整（改 ORDER BY 取最高分） | 需要子查询 |
| 并列时只取一条 | |

### 子查询 + IN
| 优点 | 缺点 |
|------|------|
| 逻辑最直观 | 并列时会全部取出 |
| 兼容性最好 | 子查询可能重复扫描 |


## 八、本题推荐写法

根据题目示例输出（每个学校只输出一条），**推荐使用 ROW_NUMBER**：

```sql
SELECT
    device_id,
    university,
    gpa
FROM (
    SELECT
        device_id,
        university,
        gpa,
        ROW_NUMBER() OVER (PARTITION BY university ORDER BY gpa) AS rn
    FROM user_profile
) t
WHERE rn = 1
ORDER BY university;
```

**原因**：
1. 严格符合题意（并列只取一条）
2. 性能良好（37ms）
3. 可扩展性强（改 ORDER BY 可查最高分）


## 九、总结

| 问题 | 答案 |
|------|------|
| 三种方式都能实现吗？ | ✅ 都能 |
| 哪个性能最好？ | MIN 窗口函数 和 子查询（34ms） |
| 哪个严格符合题意？ | ROW_NUMBER（并列只取一条） |
| 推荐用哪个？ | **ROW_NUMBER**（精确控制） |

**一句话**：性能差异不大（34-37ms），优先选择逻辑正确且可读性好的方案。本题要求每个学校一条，用 `ROW_NUMBER` 最稳妥。
