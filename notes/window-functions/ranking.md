
# SQL 学习笔记：每月播放次数 Top N 的周杰伦歌曲

## 一、需求理解

统计 2022 年 18-22 岁用户播放周杰伦歌曲的数据，按月分组，计算每首歌的播放次数，并生成每月排名。

**数据表**：
- `play_log`：播放流水（fdate, user_id, song_id）
- `user_info`：用户信息（user_id, age）
- `song_info`：歌曲信息（song_id, song_name, singer_name）

**输出字段**：
- `month`：月份
- `ranking`：当月排名
- `song_name`：歌曲名称
- `play_pv`：播放次数


## 二、完整 SQL

```sql
WITH month_count AS (
    SELECT
        MONTH(p.fdate) AS m,
        s.song_name,
        COUNT(*) AS c
    FROM play_log p
    JOIN user_info u ON p.user_id = u.user_id
    JOIN song_info s ON p.song_id = s.song_id
    WHERE YEAR(p.fdate) = 2022
        AND u.age BETWEEN 18 AND 22
        AND s.singer_name = '周杰伦'
    GROUP BY
        MONTH(p.fdate),
        s.song_id,
        s.song_name
)
SELECT
    m AS month,
    ROW_NUMBER() OVER (PARTITION BY m ORDER BY c DESC, song_name) AS ranking,
    song_name,
    c AS play_pv
FROM month_count
ORDER BY
    m,
    ranking;
```


## 三、执行步骤详解

### Step 1：CTE 部分（month_count）
```sql
-- 筛选 + 关联 + 分组统计
SELECT
    MONTH(p.fdate) AS m,      -- 提取月份
    s.song_name,               -- 歌曲名称
    COUNT(*) AS c              -- 播放次数
FROM play_log p
JOIN user_info u ON p.user_id = u.user_id      -- 关联用户表
JOIN song_info s ON p.song_id = s.song_id      -- 关联歌曲表
WHERE YEAR(p.fdate) = 2022                     -- 2022年
    AND u.age BETWEEN 18 AND 22                -- 年龄范围
    AND s.singer_name = '周杰伦'               -- 周杰伦歌曲
GROUP BY
    MONTH(p.fdate),          -- 按月分组
    s.song_id,               -- 按歌曲分组（保证唯一）
    s.song_name              -- 非聚合列必须分组
```

### Step 2：主查询（排名）
```sql
SELECT
    m AS month,
    ROW_NUMBER() OVER (
        PARTITION BY m           -- 按月分区，每月独立排名
        ORDER BY c DESC,         -- 按播放次数降序
                  song_name      -- 次数相同时按歌名排序（保证唯一）
    ) AS ranking,
    song_name,
    c AS play_pv
FROM month_count
ORDER BY
    m,           -- 按月排序
    ranking;     -- 按排名排序
```


## 四、关键知识点

| 知识点 | 说明 |
|--------|------|
| **CTE（公用表表达式）** | `WITH ... AS`，将复杂查询分步处理，提高可读性 |
| **INNER JOIN** | 三表关联：流水表 → 用户表 → 歌曲表 |
| **YEAR() / MONTH()** | 日期函数，提取年份和月份 |
| **BETWEEN AND** | 年龄范围筛选，包含边界值 |
| **GROUP BY** | SELECT 中非聚合列必须全部出现在 GROUP BY 中 |
| **ROW_NUMBER()** | 窗口函数，给每组内每行分配唯一编号 |
| **PARTITION BY** | 分组排名，每月独立从 1 开始 |
| **ORDER BY 多列** | 播放次数相同时按歌名排序，保证排名稳定 |


## 五、窗口函数详解

```sql
ROW_NUMBER() OVER (
    PARTITION BY m 
    ORDER BY c DESC, song_name
) AS ranking
```

| 参数 | 作用 | 示例 |
|------|------|------|
| `PARTITION BY m` | 按月分组 | 1月独立排名，2月独立排名 |
| `ORDER BY c DESC` | 播放次数多的排前面 | 次数 10 → 排名 1，次数 5 → 排名 2 |
| `ORDER BY song_name` | 次数相同时按歌名字典序 | "明明就" 排在 "说好的幸福呢" 前面 |


## 六、输出示例

| month | ranking | song_name | play_pv |
|-------|---------|-----------|---------|
| 1 | 1 | 明明就 | 4 |
| 1 | 2 | 说好的幸福呢 | 4 |
| 1 | 3 | 大笨钟 | 2 |
| 2 | 1 | 明明就 | 2 |
| 2 | 2 | 说好的幸福呢 | 1 |
| 2 | 3 | 大笨钟 | 1 |


## 七、取每月 Top 3 扩展

如果需要只取每月前 3 名：

```sql
WITH month_count AS (
    -- 同上
),
ranked AS (
    SELECT
        m AS month,
        ROW_NUMBER() OVER (PARTITION BY m ORDER BY c DESC, song_name) AS ranking,
        song_name,
        c AS play_pv
    FROM month_count
)
SELECT *
FROM ranked
WHERE ranking <= 3
ORDER BY month, ranking;
```


## 八、注意事项

| 注意点 | 说明 |
|--------|------|
| **GROUP BY 完整性** | `song_name` 必须在 GROUP BY 中，否则报错 |
| **排名稳定性** | 加 `song_name` 作为第二排序，避免随机排序 |
| **CTE 命名** | 使用有意义的名称，如 `month_count` |
| **别名规范** | 使用 `AS` 明确别名，提高可读性 |


## 九、总结

| 问题 | 答案 |
|------|------|
| 三表如何关联？ | `play_log` → `user_info` → `song_info` 逐层 JOIN |
| 如何按月分组？ | `GROUP BY MONTH(fdate)` |
| 如何每月独立排名？ | `ROW_NUMBER() OVER (PARTITION BY month ORDER BY count DESC)` |
| 如何取 Top N？ | 外层 `WHERE ranking <= N` |

**一句话**：CTE 先统计每月每首歌的播放次数，再用 `ROW_NUMBER() OVER (PARTITION BY month ORDER BY count DESC)` 生成每月排名，最后可加 `WHERE ranking <= N` 取 Top N。
