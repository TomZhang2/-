# 统一查询引擎 SQL 功能覆盖测试方案

> 适用范围：JDBC / Druid / StarRocks 等多源统一查询引擎  
> 目标：通过工具 + 方法论，系统性覆盖所有 SQL 语法与算子，确保功能完整性与跨引擎一致性。

---

## 一、测试目标与核心挑战

| 维度 | 挑战 |
|------|------|
| SQL 语法广度 | DDL / DML / DQL / DCL / TCL 五大类，子句组合爆炸 |
| 算子完整性 | 聚合、窗口、JOIN、子查询、集合运算、表达式等 |
| 跨引擎一致性 | 不同引擎对同一 SQL 的语义差异、数据类型映射 |
| 方言兼容 | MySQL 方言 / ANSI SQL / StarRocks 扩展语法 |
| 边界场景 | NULL 处理、空表、超大数据集、特殊字符 |

---

## 二、测试分层体系

```
┌─────────────────────────────────────────┐
│  L4  端到端回归测试（跨引擎结果比对）         │
├─────────────────────────────────────────┤
│  L3  集成测试（引擎路由 + 执行计划验证）      │
├─────────────────────────────────────────┤
│  L2  算子级测试（每个算子独立验证）           │
├─────────────────────────────────────────┤
│  L1  语法解析单测（AST 正确性）              │
└─────────────────────────────────────────┘
```

---

## 三、SQL 语法 & 算子覆盖矩阵

### 3.1 DQL（查询语句）— 最核心

#### SELECT 子句

| 测试点 | 示例 | 优先级 |
|--------|------|--------|
| 基础列选择 | `SELECT a, b FROM t` | P0 |
| 列别名 | `SELECT a AS col1` | P0 |
| `SELECT *` | `SELECT * FROM t` | P0 |
| 表达式 | `SELECT a + b, a * 2` | P0 |
| 字面量 | `SELECT 1, 'str', NULL` | P1 |
| DISTINCT | `SELECT DISTINCT a` | P0 |
| 子查询列 | `SELECT (SELECT max(x) FROM t2)` | P1 |

#### FROM 子句 & JOIN

| 测试点 | 示例 | 优先级 |
|--------|------|--------|
| 单表 | `FROM t` | P0 |
| 表别名 | `FROM t1 AS a` | P0 |
| INNER JOIN | `t1 JOIN t2 ON t1.id = t2.id` | P0 |
| LEFT JOIN | `t1 LEFT JOIN t2 ON ...` | P0 |
| RIGHT JOIN | `t1 RIGHT JOIN t2 ON ...` | P0 |
| FULL OUTER JOIN | `t1 FULL JOIN t2 ON ...` | P1 |
| CROSS JOIN | `t1 CROSS JOIN t2` | P1 |
| SELF JOIN | `t1 a JOIN t1 b ON a.pid = b.id` | P1 |
| 多表 JOIN | 3 表及以上链式 JOIN | P1 |
| 子查询作为表 | `FROM (SELECT ...) sub` | P0 |
| LATERAL JOIN | `LATERAL VIEW explode(...)` | P2 |

#### WHERE 子句

| 测试点 | 示例 | 优先级 |
|--------|------|--------|
| 比较运算符 | `=, !=, <, >, <=, >=` | P0 |
| 逻辑运算符 | `AND, OR, NOT` | P0 |
| IN / NOT IN | `WHERE a IN (1, 2, 3)` | P0 |
| BETWEEN | `WHERE a BETWEEN 1 AND 10` | P0 |
| LIKE / NOT LIKE | `WHERE name LIKE 'a%'` | P0 |
| IS NULL / IS NOT NULL | `WHERE a IS NULL` | P0 |
| EXISTS / NOT EXISTS | `WHERE EXISTS (SELECT ...)` | P1 |
| 子查询谓词 | `WHERE a > (SELECT avg(x) FROM t)` | P1 |

#### GROUP BY & 聚合算子

| 聚合函数 | 测试点 | 优先级 |
|----------|--------|--------|
| COUNT | `COUNT(*), COUNT(a), COUNT(DISTINCT a)` | P0 |
| SUM / AVG | 含 NULL、含 DISTINCT | P0 |
| MAX / MIN | 字符串、数字、日期类型 | P0 |
| GROUP BY | 单列、多列、表达式、别名 | P0 |
| HAVING | `HAVING COUNT(*) > 1` | P0 |
| ROLLUP | `GROUP BY ROLLUP(a, b)` | P1 |
| CUBE | `GROUP BY CUBE(a, b)` | P1 |
| GROUPING SETS | `GROUPING SETS ((a), (b), ())` | P1 |

#### ORDER BY & LIMIT

| 测试点 | 示例 | 优先级 |
|--------|------|--------|
| ASC / DESC | `ORDER BY a DESC` | P0 |
| 多列排序 | `ORDER BY a ASC, b DESC` | P0 |
| NULL 排序 | `ORDER BY a NULLS FIRST/LAST` | P1 |
| LIMIT | `LIMIT 10` | P0 |
| OFFSET | `LIMIT 10 OFFSET 5` | P0 |

#### 窗口函数（Window Functions）

| 函数类型 | 测试点 | 优先级 |
|----------|--------|--------|
| 排名函数 | `ROW_NUMBER(), RANK(), DENSE_RANK()` | P0 |
| 分布函数 | `PERCENT_RANK(), CUME_DIST()` | P1 |
| 偏移函数 | `LAG(a, 1), LEAD(a, 1)` | P0 |
| 聚合窗口 | `SUM(a) OVER (PARTITION BY b)` | P0 |
| FIRST_VALUE / LAST_VALUE | 含 IGNORE NULLS | P1 |
| 帧规范 | `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | P1 |
| NTILE | `NTILE(4) OVER (ORDER BY score)` | P1 |

#### 子查询

| 类型 | 示例 | 优先级 |
|------|------|--------|
| 标量子查询 | `SELECT (SELECT max(x) FROM t)` | P0 |
| 相关子查询 | `WHERE a = (SELECT ... WHERE t2.id = t1.id)` | P1 |
| IN 子查询 | `WHERE a IN (SELECT x FROM t2)` | P0 |
| EXISTS 子查询 | `WHERE EXISTS (SELECT 1 FROM t2 WHERE ...)` | P1 |
| FROM 子查询 | `FROM (SELECT ...) alias` | P0 |
| CTE (WITH) | `WITH cte AS (...) SELECT * FROM cte` | P0 |
| 递归 CTE | `WITH RECURSIVE cte AS (...)` | P2 |

#### 集合运算

| 算子 | 测试点 | 优先级 |
|------|--------|--------|
| UNION ALL | 列数、类型一致性 | P0 |
| UNION | 去重逻辑 | P0 |
| INTERSECT | | P1 |
| EXCEPT / MINUS | | P1 |

### 3.2 内置函数覆盖

#### 数值函数

`ABS, CEIL, FLOOR, ROUND, TRUNCATE, MOD, POWER, SQRT, LOG, EXP, SIGN, RAND`

#### 字符串函数

`CONCAT, CONCAT_WS, SUBSTR/SUBSTRING, LENGTH, TRIM, LTRIM, RTRIM, UPPER, LOWER, REPLACE, REGEXP_REPLACE, SPLIT, INSTR, LPAD, RPAD, REPEAT, REVERSE, ASCII, CHAR`

#### 日期时间函数

`NOW, CURDATE, DATE, YEAR, MONTH, DAY, HOUR, MINUTE, SECOND, DATE_ADD, DATE_SUB, DATEDIFF, DATE_FORMAT, TO_DATE, UNIX_TIMESTAMP, FROM_UNIXTIME, TIMESTAMPDIFF`

#### 条件函数

`IF, IFNULL, NULLIF, COALESCE, CASE WHEN ... THEN ... ELSE ... END, IIF`

#### 类型转换

`CAST(a AS INT), CONVERT(a, CHAR), 隐式类型转换`

#### 聚合扩展

`GROUP_CONCAT (MySQL), LISTAGG (ANSI), array_agg (PG风格), APPROX_COUNT_DISTINCT (StarRocks)`

### 3.3 DDL / DML（视实现需要）

| 类型 | 语句 |
|------|------|
| DDL | CREATE TABLE, DROP TABLE, ALTER TABLE, TRUNCATE |
| DML | INSERT INTO, UPDATE, DELETE, MERGE INTO (Upsert) |
| DCL | GRANT, REVOKE |
| TCL | BEGIN, COMMIT, ROLLBACK, SAVEPOINT |

---

## 四、推荐工具与方法

### 4.1 自动化 SQL 覆盖工具

#### ① **SQLsmith** — SQL 随机模糊生成
- 用途：随机生成大量语法合法的 SQL，触发未知 bug
- 原理：基于语法规则的随机组合生成
- 接入方式：对目标引擎执行生成的 SQL，对比结果或检查 crash

```bash
# SQLsmith GitHub: https://github.com/anse1/sqlsmith
sqlsmith --target="postgresql://..." --max-queries=10000
```

#### ② **Apache Calcite TestSuite** — 标准 SQL 算子测试
- 用途：覆盖 SQL 标准算子，Calcite 的解析器测试集可直接复用
- 适合：你的引擎若基于 Calcite，可复用其 `SqlValidatorTest`、`SqlToRelConverterTest`

#### ③ **dbt (Data Build Tool) + dbt-audit-helper** — 跨引擎结果比对
- 用途：在多个引擎上运行相同 SQL，对比结果集（行数、hash、采样数据）
- 场景：验证 JDBC / Druid / StarRocks 返回结果一致性

```yaml
# dbt test 示例
- name: result_row_count_match
  tests:
    - dbt_utils.equality:
        compare_model: ref('query_on_mysql')
```

#### ④ **SqlLogicTest (SLT)** — 业界标准 SQL 测试框架
- 来源：SQLite 团队开发，TiDB/DuckDB/CockroachDB 广泛使用
- 格式：`.test` 文件，包含 SQL + 期望结果
- 集成方式：直接集成 go-sqllogictest 或 Python 版本

```
# SLT 用例格式
statement ok
CREATE TABLE t1(a INT, b INT)

query II rowsort
SELECT a, b FROM t1 ORDER BY a
----
1 2
3 4
```

> **强烈推荐**：TiDB 维护了 30,000+ 条 SLT 用例，可直接复用  
> 仓库：https://github.com/pingcap/tidb/tree/master/tests/logictest

#### ⑤ **Fuzz Testing with AFL / libFuzzer**
- 用途：对 SQL 解析器进行二进制 fuzz，发现 crash 和解析异常
- 适合：有自定义 SQL Parser 的场景

#### ⑥ **QuerySurgeon / DataFusion** 
- DataFusion 提供了完整的 SQL 算子测试集
- 仓库：https://github.com/apache/datafusion

### 4.2 测试数据策略

#### 基准数据集

| 数据集 | 特点 | 用途 |
|--------|------|------|
| **TPC-H** | 8张业务表，22条标准查询 | 关联查询、聚合 |
| **TPC-DS** | 复杂数仓场景，99条查询 | 窗口函数、CTE |
| **SSB (Star Schema Benchmark)** | 星型模型 | JOIN 性能与语义 |
| **自定义边界数据** | NULL 列、空表、单行表 | 边界验证 |

```sql
-- TPC-H Q1 示例（验证聚合+条件+排序）
SELECT
    l_returnflag,
    l_linestatus,
    SUM(l_quantity) AS sum_qty,
    AVG(l_extendedprice) AS avg_price,
    COUNT(*) AS count_order
FROM lineitem
WHERE l_shipdate <= DATE '1998-12-01' - INTERVAL '90' DAY
GROUP BY l_returnflag, l_linestatus
ORDER BY l_returnflag, l_linestatus;
```

### 4.3 跨引擎结果比对方案

```
┌────────────────┐     同一SQL      ┌──────────────────┐
│  统一查询引擎   │ ──────────────► │  JDBC (MySQL)    │
│  (被测系统)    │                  └──────────────────┘
│               │ ──────────────► ┌──────────────────┐
│               │                  │  Druid           │
│               │ ──────────────► └──────────────────┘
└────────────────┘                ┌──────────────────┐
        │                          │  StarRocks       │
        │                          └──────────────────┘
        │
        ▼
   结果比对服务
   ┌─────────────────────────┐
   │ 1. 行数比对              │
   │ 2. 结果集 hash 比对      │
   │ 3. 排序后逐行比对        │
   │ 4. 数值精度误差容忍      │
   └─────────────────────────┘
```

---

## 五、测试用例组织结构

```
tests/
├── syntax/               # L1: 语法解析测试
│   ├── select/
│   ├── join/
│   ├── subquery/
│   └── window/
├── operators/            # L2: 算子功能测试
│   ├── aggregation/
│   ├── window_functions/
│   ├── set_operations/
│   └── expressions/
├── integration/          # L3: 引擎集成测试
│   ├── jdbc/
│   ├── druid/
│   └── starrocks/
├── cross_engine/         # L4: 跨引擎对比测试
│   ├── tpch/
│   ├── tpcds/
│   └── custom/
├── edge_cases/           # 边界与异常
│   ├── null_handling/
│   ├── empty_table/
│   ├── data_types/
│   └── errors/
└── slt/                  # SqlLogicTest 用例
    └── *.test
```

---

## 六、覆盖率度量指标

### 6.1 SQL 语法覆盖率
- 工具：基于 ANTLR 的 SQL 语法树，统计语法规则触达率
- 目标：**核心语法规则 ≥ 95%**

### 6.2 算子覆盖率
- 定义算子清单（约 80 个核心算子）
- 每个算子至少一个正向测试 + 一个边界测试
- 目标：**P0 算子 100%，P1 算子 ≥ 90%**

### 6.3 跨引擎一致性
- 相同 SQL 在所有支持引擎上结果一致率
- 目标：**结果一致率 ≥ 99.5%**（允许浮点精度差异）

### 6.4 回归覆盖
- 所有 Bug Fix 配套回归用例
- 发版前执行完整回归，通过率 **100%**

---

## 七、分阶段执行计划

### Phase 1：基础覆盖（第 1-2 周）
- [ ] 搭建 SqlLogicTest 框架，接入 TiDB 现成 SLT 用例集
- [ ] 构建 TPC-H 测试数据集（Scale Factor=1）
- [ ] 覆盖 SELECT/WHERE/JOIN/GROUP BY/ORDER BY P0 用例
- [ ] 搭建跨引擎结果比对服务（基于行数 + hash）

### Phase 2：算子深化（第 3-4 周）
- [ ] 窗口函数全算子覆盖
- [ ] 子查询（相关/非相关）全场景
- [ ] 集合运算（UNION/INTERSECT/EXCEPT）
- [ ] CTE 与递归查询
- [ ] 内置函数（数值/字符串/日期）全覆盖

### Phase 3：边界与方言（第 5-6 周）
- [ ] NULL 值处理专项测试
- [ ] 数据类型边界（INT 溢出、DECIMAL 精度、DATE 范围）
- [ ] 引擎方言差异测试（StarRocks 扩展语法、Druid SQL 限制）
- [ ] 错误处理与异常 SQL 测试

### Phase 4：自动化与 Fuzz（持续）
- [ ] 接入 SQLsmith 进行随机 SQL 生成测试
- [ ] CI 流水线集成，每次 PR 触发完整回归
- [ ] TPC-DS 复杂查询覆盖

---

## 八、典型边界测试 Checklist

```
□ NULL 与任意值的比较（= NULL vs IS NULL）
□ 空表 GROUP BY / HAVING / ORDER BY
□ 单行表的窗口函数
□ DISTINCT + ORDER BY 列不在 SELECT 中
□ 嵌套子查询深度 ≥ 3 层
□ JOIN ON 条件为永假（0 行结果）
□ 字符串与数字隐式类型转换
□ LIMIT 0 / OFFSET 超出行数
□ 聚合函数嵌套（COUNT(DISTINCT a) 中的 DISTINCT）
□ 窗口函数 PARTITION BY 空列
□ UNION ALL 两侧列类型不完全一致（宽化）
□ 超长字符串、特殊字符（换行、制表符、Unicode）
□ 时区相关日期计算
□ 除以零、取模零
□ CASE WHEN 全分支为 NULL
```

---

## 九、工具选型汇总

| 工具/框架 | 用途 | 推荐度 |
|-----------|------|--------|
| **SqlLogicTest** | 标准 SQL 功能测试框架 | ⭐⭐⭐⭐⭐ |
| **TPC-H / TPC-DS** | 标准测试数据集 | ⭐⭐⭐⭐⭐ |
| **SQLsmith** | 随机 SQL Fuzz | ⭐⭐⭐⭐ |
| **dbt + audit-helper** | 跨引擎结果比对 | ⭐⭐⭐⭐ |
| **Apache Calcite Tests** | 算子单元测试参考 | ⭐⭐⭐⭐ |
| **DataFusion Test Suite** | SQL 算子覆盖参考 | ⭐⭐⭐ |
| **Testcontainers** | 本地多引擎容器化 | ⭐⭐⭐⭐ |
| **ANTLR + coverage** | 语法规则覆盖率统计 | ⭐⭐⭐ |

---

*文档版本：v1.0 | 生成时间：2026-03-31*
