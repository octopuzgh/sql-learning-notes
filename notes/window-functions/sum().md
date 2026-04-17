## 一、需求理解

计算每日利润的累计和（截止到当日的总利润），按日期排序输出。

**原始数据**：`daily_profits` 表
| profit_date | profit |
|-------------|--------|
| 2024-01-01 | 100 |
| 2024-01-02 | 200 |
| 2024-01-03 | 50 |
| 2024-01-04 | 150 |

**期望输出**：
| profit_date | profit | cumulative_profit |
|-------------|--------|-------------------|
| 2024-01-01 | 100 | 100 |
| 2024-01-02 | 200 | 300 |
| 2024-01-03 | 50 | 350 |
| 2024-01-04 | 150 | 500 |


## 二、SQL 写法

```sql
SELECT
    *,
    SUM(profit) OVER (
        ORDER BY profit_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_profit
FROM daily_profits
ORDER BY profit_date;
```


## 三、参数解析

```sql
SUM(profit) OVER (
    ORDER BY profit_date                    -- 按日期排序
    ROWS BETWEEN UNBOUNDED PRECEDING        -- 从第一行开始
          AND CURRENT ROW                   -- 到当前行结束
)
```

| 参数 | 含义 |
|------|------|
| `SUM(profit)` | 要对 profit 列求和 |
| `ORDER BY profit_date` | 按日期排序，决定累计顺序 |
| `UNBOUNDED PRECEDING` | 从分区（这里全表）的第一行开始 |
| `CURRENT ROW` | 到当前行结束 |

**简写形式**（效果相同）：
```sql
SUM(profit) OVER (ORDER BY profit_date ROWS UNBOUNDED PRECEDING)
```


## 四、执行逻辑

```
第1行（2024-01-01, 100）：
  窗口范围 = 第1行到第1行 → SUM = 100

第2行（2024-01-02, 200）：
  窗口范围 = 第1行到第2行 → SUM = 100 + 200 = 300

第3行（2024-01-03, 50）：
  窗口范围 = 第1行到第3行 → SUM = 100 + 200 + 50 = 350

第4行（2024-01-04, 150）：
  窗口范围 = 第1行到第4行 → SUM = 100 + 200 + 50 + 150 = 500
```


## 五、范围参数对照表

| 写法 | 含义 |
|------|------|
| `UNBOUNDED PRECEDING` | 从第一行开始 |
| `N PRECEDING` | 当前行前 N 行 |
| `CURRENT ROW` | 当前行 |
| `N FOLLOWING` | 当前行后 N 行 |
| `UNBOUNDED FOLLOWING` | 到最后一行 |

### 常用组合

| 需求 | Frame 写法 |
|------|-----------|
| 累计到当前（累计求和） | `ROWS UNBOUNDED PRECEDING` |
| 前3行移动平均 | `ROWS BETWEEN 3 PRECEDING AND CURRENT ROW` |
| 前后各2行（中心平均） | `ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING` |
| 全表聚合 | `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |


## 六、注意事项

| 注意点 | 说明 |
|--------|------|
| **需要 ORDER BY** | 使用 Frame 时必须有 `ORDER BY` |
| **默认行为** | 有 `ORDER BY` 无 Frame 时，默认 `ROWS UNBOUNDED PRECEDING` |
| **内外 ORDER BY 一致** | 建议窗口函数内外的 `ORDER BY` 保持一致，否则累计意义会乱 |


## 七、扩展场景

### 场景1：移动平均（前3天）
```sql
SELECT
    *,
    AVG(profit) OVER (
        ORDER BY profit_date 
        ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
    ) AS ma3
FROM daily_profits;
```

### 场景2：分组内累计
```sql
-- 每个部门内按日期累计
SELECT
    *,
    SUM(profit) OVER (
        PARTITION BY department 
        ORDER BY profit_date 
        ROWS UNBOUNDED PRECEDING
    ) AS dept_cumulative
FROM daily_profits;
```

### 场景3：占比计算
```sql
SELECT
    *,
    profit * 100.0 / SUM(profit) OVER () AS percentage
FROM daily_profits;
```


## 八、总结

| 问题 | 答案 |
|------|------|
| 累计求和标准写法 | `SUM(col) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING)` |
| 简写形式 | `SUM(col) OVER (ORDER BY date)` |
| 关键参数 | `ORDER BY` 决定累计顺序 |
| 应用场景 | 累计和、移动平均、同比环比 |

**一句话**：`ROWS UNBOUNDED PRECEDING` 是累计求和的标准写法，从第一行加到当前行。简写 `ROWS UNBOUNDED PRECEDING` 可以省略 `BETWEEN...AND...`。
```