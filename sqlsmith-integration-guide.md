# SQLsmith 集成统一查询引擎 Fuzz 测试方案

> 目标：用 SQLsmith 随机生成 SQL，通过 Python Harness 调用统一查询引擎 REST 接口，与基准引擎结果比对，自动发现功能缺陷。  
> 适用引擎：JDBC / Druid / StarRocks 等多源统一查询引擎  
> 文档版本：v1.0 | 2026-03-31

---

## 一、整体架构

```
SQLsmith（随机生成 SQL）
     │
     │  stdout 输出 SQL 流
     ▼
Python Harness（核心粘合层）
     ├─ 调用统一查询引擎 REST API
     ├─ 同时调用基准引擎（直连 MySQL/StarRocks）
     ├─ 结果比对（hash / 行数 / 逐行）
     └─ 记录 diff / crash 到报告文件
```

**核心优势：**
- SQLsmith 基于真实 schema 生成**语义合法**的 SQL，比纯随机字符串更有效
- 双引擎结果比对，自动发现结果差异，无需人工逐条验证
- 差异用例自动落地 `failures.jsonl`，方便追踪和复现

---

## 二、Step 1：编译安装 SQLsmith

SQLsmith 需要从源码编译（C++），推荐在 Linux / WSL 环境下操作。

### 2.1 安装依赖

```bash
sudo apt-get install -y g++ libpq-dev libsqlite3-dev cmake git
```

### 2.2 克隆并编译

```bash
git clone https://github.com/anse1/sqlsmith.git
cd sqlsmith
make
```

### 2.3 验证安装

```bash
./sqlsmith --help
```

### 2.4 常用启动参数说明

| 参数 | 说明 |
|------|------|
| `--target=sqlite3:FILE` | 连接 SQLite 文件，按其 schema 生成 SQL |
| `--target=postgresql://...` | 连接 PostgreSQL，按真实 schema 生成 |
| `--max-queries=N` | 最多生成 N 条 SQL |
| `--dump-all-queries` | 只输出 SQL 到 stdout，不实际执行（**最适合本方案**） |

---

## 三、Step 2：创建 Schema 镜像文件

SQLsmith 会读取目标库的 schema，生成真实列名和表名的 SQL。  
最简单的做法是用 **SQLite 文件镜像你的表结构**：

```bash
sqlite3 test_schema.db << 'EOF'
-- 复制你的核心业务表结构
CREATE TABLE orders (
    id         INTEGER,
    user_id    INTEGER,
    amount     DECIMAL(10,2),
    status     VARCHAR(20),
    created_at DATE
);

CREATE TABLE users (
    id    INTEGER,
    name  VARCHAR(100),
    city  VARCHAR(50),
    age   INTEGER
);

CREATE TABLE products (
    id       INTEGER,
    name     VARCHAR(200),
    category VARCHAR(50),
    price    DECIMAL(10,2),
    stock    INTEGER
);

-- 继续添加你的其他表...
EOF
```

> **建议**：将核心查询涉及的表都加进来，SQLsmith 生成的 SQL 质量与 schema 丰富度正相关。表越多、列越多，生成的 JOIN / 子查询越丰富。

---

## 四、Step 3：Python Fuzz Harness

### 4.1 依赖安装

```bash
pip install requests pymysql
```

### 4.2 完整代码（`fuzz_runner.py`）

```python
#!/usr/bin/env python3
"""
SQLsmith + 统一查询引擎 REST 接口 Fuzz 测试框架
"""

import subprocess
import requests
import json
import hashlib
import logging
import time
import sqlite3
from dataclasses import dataclass, field
from typing import Optional
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime
import pymysql  # pip install pymysql

# ──────────────────────────────────────────
# 配置区（按实际情况修改）
# ──────────────────────────────────────────
SQLSMITH_BIN = "./sqlsmith/sqlsmith"       # sqlsmith 可执行文件路径
SCHEMA_DB    = "./test_schema.db"          # SQLite schema 镜像文件

# 统一查询引擎 REST 接口
UNIFIED_ENGINE_URL = "http://localhost:8080/api/query"
UNIFIED_ENGINE_HEADERS = {
    "Content-Type": "application/json",
    "Authorization": "Bearer YOUR_TOKEN"   # 替换为实际 token
}
DATASOURCE_ID = "starrocks_prod"           # 传给引擎的数据源标识

# 基准引擎（直连，用于结果比对）
BASELINE_DB = {
    "host":     "localhost",
    "port":     3306,
    "user":     "root",
    "password": "password",
    "database": "testdb"
}

# 运行参数
MAX_QUERIES     = 5000   # 生成 SQL 总数
TIMEOUT_SECONDS = 10     # 单条 SQL 超时（秒）
MAX_WORKERS     = 4      # 并发线程数
RESULT_LIMIT    = 1000   # 最多比对前 N 行结果

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler(f"fuzz_{datetime.now().strftime('%Y%m%d_%H%M%S')}.log"),
        logging.StreamHandler()
    ]
)
log = logging.getLogger(__name__)


# ──────────────────────────────────────────
# 数据结构
# ──────────────────────────────────────────
@dataclass
class QueryResult:
    sql: str
    success: bool
    rows: list = field(default_factory=list)
    error: Optional[str] = None
    elapsed_ms: float = 0.0
    row_count: int = 0

@dataclass
class CompareResult:
    sql: str
    matched: bool
    reason: str        # 'ok' | 'row_count_diff' | 'data_diff' | 'engine_error' | 'baseline_error'
    unified_rows: int = 0
    baseline_rows: int = 0


# ──────────────────────────────────────────
# 调用统一查询引擎 REST API
# ──────────────────────────────────────────
def call_unified_engine(sql: str, datasource: str) -> QueryResult:
    payload = {
        "sql": sql,
        "datasourceId": datasource,
        "limit": RESULT_LIMIT
    }
    start = time.time()
    try:
        resp = requests.post(
            UNIFIED_ENGINE_URL,
            headers=UNIFIED_ENGINE_HEADERS,
            json=payload,
            timeout=TIMEOUT_SECONDS
        )
        elapsed = (time.time() - start) * 1000

        if resp.status_code == 200:
            data = resp.json()
            # ⚠️ 根据实际接口响应格式调整此处
            rows = data.get("data", {}).get("rows", [])
            return QueryResult(
                sql=sql, success=True,
                rows=rows, elapsed_ms=elapsed,
                row_count=len(rows)
            )
        else:
            return QueryResult(
                sql=sql, success=False,
                error=f"HTTP {resp.status_code}: {resp.text[:200]}",
                elapsed_ms=elapsed
            )
    except requests.Timeout:
        return QueryResult(sql=sql, success=False, error="TIMEOUT")
    except Exception as e:
        return QueryResult(sql=sql, success=False, error=str(e))


# ──────────────────────────────────────────
# 调用基准引擎（直连 MySQL）
# ──────────────────────────────────────────
def call_baseline(sql: str) -> QueryResult:
    start = time.time()
    try:
        conn = pymysql.connect(**BASELINE_DB, connect_timeout=5)
        with conn.cursor() as cursor:
            cursor.execute(sql)
            rows = cursor.fetchmany(RESULT_LIMIT)
            rows = [list(r) for r in rows]
        conn.close()
        elapsed = (time.time() - start) * 1000
        return QueryResult(
            sql=sql, success=True,
            rows=rows, elapsed_ms=elapsed,
            row_count=len(rows)
        )
    except Exception as e:
        return QueryResult(sql=sql, success=False, error=str(e))


# ──────────────────────────────────────────
# 结果比对
# ──────────────────────────────────────────
def rows_hash(rows: list) -> str:
    """对结果集排序后计算 hash，消除行顺序差异"""
    normalized = sorted([
        json.dumps(r, sort_keys=True, default=str) for r in rows
    ])
    return hashlib.md5("\n".join(normalized).encode()).hexdigest()


def compare(sql: str, unified: QueryResult, baseline: QueryResult) -> CompareResult:
    # 基准引擎报错（通常是 sqlsmith 生成了方言不兼容的 SQL）
    if not baseline.success:
        return CompareResult(sql=sql, matched=True, reason="baseline_error")

    # 统一引擎报错但基准引擎成功 → BUG
    if not unified.success:
        return CompareResult(
            sql=sql, matched=False,
            reason="engine_error",
            baseline_rows=baseline.row_count
        )

    # 行数比对
    if unified.row_count != baseline.row_count:
        return CompareResult(
            sql=sql, matched=False, reason="row_count_diff",
            unified_rows=unified.row_count,
            baseline_rows=baseline.row_count
        )

    # 数据内容 hash 比对
    if rows_hash(unified.rows) != rows_hash(baseline.rows):
        return CompareResult(
            sql=sql, matched=False, reason="data_diff",
            unified_rows=unified.row_count,
            baseline_rows=baseline.row_count
        )

    return CompareResult(sql=sql, matched=True, reason="ok",
                         unified_rows=unified.row_count,
                         baseline_rows=baseline.row_count)


# ──────────────────────────────────────────
# 过滤写操作 SQL（只测查询）
# ──────────────────────────────────────────
SKIP_KEYWORDS = {"insert", "update", "delete", "drop", "create", "alter", "truncate"}

def should_skip(sql: str) -> bool:
    first_word = sql.strip().split()[0].lower() if sql.strip() else ""
    return first_word in SKIP_KEYWORDS


# ──────────────────────────────────────────
# SQLsmith 进程：逐行 yield SQL
# ──────────────────────────────────────────
def stream_sqlsmith_queries(max_queries: int):
    cmd = [
        SQLSMITH_BIN,
        f"--target=sqlite3:{SCHEMA_DB}",
        f"--max-queries={max_queries}",
        "--dump-all-queries"
    ]
    log.info(f"启动 SQLsmith: {' '.join(cmd)}")

    proc = subprocess.Popen(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.DEVNULL,
        text=True,
        bufsize=1
    )

    buf = []
    for line in proc.stdout:
        line = line.rstrip()
        if line.endswith(";"):
            buf.append(line)
            sql = " ".join(buf).strip()
            buf = []
            if sql and not should_skip(sql):
                yield sql
        elif line:
            buf.append(line)

    proc.wait()


# ──────────────────────────────────────────
# 主流程：双引擎比对模式
# ──────────────────────────────────────────
def run_fuzz(max_queries: int = MAX_QUERIES):
    stats = {"total": 0, "passed": 0, "failed": 0, "skipped": 0}
    failures = []

    def process_one(sql):
        unified  = call_unified_engine(sql, DATASOURCE_ID)
        baseline = call_baseline(sql)
        return compare(sql, unified, baseline)

    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as pool:
        futures = {pool.submit(process_one, sql): sql
                   for sql in stream_sqlsmith_queries(max_queries)}
        stats["total"] = len(futures)

        for fut in as_completed(futures):
            result: CompareResult = fut.result()

            if result.reason == "baseline_error":
                stats["skipped"] += 1
                continue

            if result.matched:
                stats["passed"] += 1
                if stats["passed"] % 100 == 0:
                    log.info(f"✅ 已通过 {stats['passed']} 条")
            else:
                stats["failed"] += 1
                failures.append(result)
                log.warning(
                    f"❌ FAIL [{result.reason}] "
                    f"unified={result.unified_rows}行 "
                    f"baseline={result.baseline_rows}行\n"
                    f"   SQL: {result.sql[:120]}..."
                )

    # 打印报告
    print("\n" + "="*60)
    print("📊 Fuzz 测试报告")
    print("="*60)
    print(f"  总生成 SQL        : {stats['total']}")
    print(f"  通过              : {stats['passed']}")
    print(f"  失败              : {stats['failed']}")
    print(f"  跳过(基准引擎错误): {stats['skipped']}")
    total_valid = stats['passed'] + stats['failed']
    if total_valid > 0:
        print(f"  通过率            : {stats['passed']/total_valid*100:.1f}%")

    # 保存失败用例
    if failures:
        output_file = f"failures_{datetime.now().strftime('%Y%m%d_%H%M%S')}.jsonl"
        with open(output_file, "w", encoding="utf-8") as f:
            for r in failures:
                f.write(json.dumps({
                    "sql":           r.sql,
                    "reason":        r.reason,
                    "unified_rows":  r.unified_rows,
                    "baseline_rows": r.baseline_rows
                }, ensure_ascii=False) + "\n")
        print(f"\n  ⚠️  失败用例已保存到 {output_file}")

    return stats


# ──────────────────────────────────────────
# 简化模式：只验证引擎不崩溃（无需基准引擎）
# ──────────────────────────────────────────
def run_smoke_test(max_queries: int = 1000):
    """只验证统一引擎不返回 500 / 不崩溃，不做结果比对"""
    crash_sqls = []

    for i, sql in enumerate(stream_sqlsmith_queries(max_queries)):
        result = call_unified_engine(sql, DATASOURCE_ID)

        if not result.success and result.error != "TIMEOUT":
            crash_sqls.append({"sql": sql, "error": result.error})
            log.error(f"CRASH #{len(crash_sqls)}: {result.error[:100]}")

        if i % 200 == 0:
            log.info(f"进度: {i}/{max_queries}, crash={len(crash_sqls)}")

    log.info(f"Smoke test 完成，发现 {len(crash_sqls)} 个 crash")

    if crash_sqls:
        with open("smoke_crashes.jsonl", "w", encoding="utf-8") as f:
            for item in crash_sqls:
                f.write(json.dumps(item, ensure_ascii=False) + "\n")
        log.info("crash 用例已保存到 smoke_crashes.jsonl")

    return crash_sqls


# ──────────────────────────────────────────
# 入口
# ──────────────────────────────────────────
if __name__ == "__main__":
    import sys
    mode = sys.argv[1] if len(sys.argv) > 1 else "fuzz"

    if mode == "smoke":
        run_smoke_test()
    else:
        run_fuzz()
```

---

## 五、Step 4：适配你的 REST 接口格式

代码中 `call_unified_engine` 函数里的响应解析需要根据你的实际接口调整。

### 当前假设的接口格式

**Request：**
```json
POST /api/query
{
    "sql": "SELECT * FROM orders LIMIT 10",
    "datasourceId": "starrocks_prod",
    "limit": 1000
}
```

**Response：**
```json
{
    "code": 0,
    "data": {
        "rows": [
            {"id": 1, "user_id": 100, "amount": 99.9},
            {"id": 2, "user_id": 101, "amount": 150.0}
        ],
        "total": 2
    }
}
```

**对应代码（已内置）：**
```python
rows = data.get("data", {}).get("rows", [])
```

### 常见接口格式适配示例

```python
# 格式 A：data 直接是数组
rows = data.get("data", [])

# 格式 B：result.list
rows = data.get("result", {}).get("list", [])

# 格式 C：rows 在顶层
rows = data.get("rows", [])

# 格式 D：列式存储（columns + data 分离）
columns = data["columns"]   # ["id", "name", ...]
raw     = data["data"]      # [[1, "a"], [2, "b"]]
rows    = [dict(zip(columns, r)) for r in raw]
```

---

## 六、运行方式

### 6.1 双引擎比对模式（推荐）

```bash
python fuzz_runner.py fuzz
```

### 6.2 仅 Smoke Test（无需基准引擎）

```bash
python fuzz_runner.py smoke
```

### 6.3 输出文件说明

| 文件 | 说明 |
|------|------|
| `fuzz_YYYYMMDD_HHMMSS.log` | 完整运行日志 |
| `failures_YYYYMMDD_HHMMSS.jsonl` | 失败 SQL 用例（每行一个 JSON） |
| `smoke_crashes.jsonl` | Smoke 模式发现的 crash SQL |

---

## 七、故障排查与调优

### 7.1 SQLsmith 生成的 SQL 大量被基准引擎拒绝

**原因：** SQLite 方言与 MySQL 差异（如 `ILIKE`、`SIMILAR TO` 等）  
**解法：** 改用 PostgreSQL 作为 target，或对 SQLsmith 输出做方言转换

```bash
# 使用 PG 作为 target（schema 更接近 MySQL）
./sqlsmith --target=postgresql://user:pass@localhost/testdb \
           --max-queries=5000 \
           --dump-all-queries
```

### 7.2 想只生成某类 SQL（如只测 JOIN）

SQLsmith 暂不支持过滤，可在 Python 层加关键字过滤：

```python
def is_join_query(sql: str) -> bool:
    return "join" in sql.lower()

# 在 stream_sqlsmith_queries 里添加：
if not is_join_query(sql):
    continue
```

### 7.3 结果比对出现浮点精度误差

```python
# 在 compare 函数里对浮点列做容差处理
import math

def normalize_row(row: dict) -> dict:
    return {
        k: round(v, 6) if isinstance(v, float) else v
        for k, v in row.items()
    }

# rows_hash 里调用 normalize_row
normalized = sorted([
    json.dumps(normalize_row(r), sort_keys=True, default=str) for r in rows
])
```

### 7.4 提高生成 SQL 的多样性

在 schema 文件中增加更多表、更多数据类型：

```sql
-- 加入 NULL 列、枚举、日期、布尔等类型，sqlsmith 生成更丰富的类型转换 SQL
CREATE TABLE events (
    id          INTEGER,
    type        VARCHAR(20),
    payload     TEXT,
    is_active   BOOLEAN,
    score       DOUBLE,
    tags        VARCHAR(500),
    event_time  TIMESTAMP,
    deleted_at  TIMESTAMP    -- NULL 列
);
```

---

## 八、CI 集成（GitHub Actions 示例）

```yaml
name: SQL Fuzz Test

on:
  push:
    branches: [main, dev]
  schedule:
    - cron: '0 2 * * *'   # 每天凌晨 2 点跑一次

jobs:
  fuzz:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: 编译 SQLsmith
        run: |
          sudo apt-get install -y g++ libpq-dev libsqlite3-dev
          git clone https://github.com/anse1/sqlsmith.git
          cd sqlsmith && make

      - name: 安装 Python 依赖
        run: pip install requests pymysql

      - name: 运行 Fuzz（Smoke 模式）
        run: python fuzz_runner.py smoke
        env:
          UNIFIED_ENGINE_URL: ${{ secrets.UNIFIED_ENGINE_URL }}
          ENGINE_TOKEN: ${{ secrets.ENGINE_TOKEN }}

      - name: 上传失败用例
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: fuzz-failures
          path: "*.jsonl"
```

---

## 九、快速上手清单

```
□ 1. 在 Linux/WSL 编译 sqlsmith，确认 --dump-all-queries 可输出 SQL
□ 2. 创建 test_schema.db，镜像核心业务表结构
□ 3. 复制 fuzz_runner.py，修改配置区（URL、token、数据源 ID）
□ 4. 根据实际接口格式调整 call_unified_engine 里的 rows 解析
□ 5. 先跑 smoke 模式（python fuzz_runner.py smoke），验证流程通畅
□ 6. 配置基准引擎连接，跑 fuzz 模式进行结果比对
□ 7. 分析 failures.jsonl，按 reason 分类处理问题
□ 8. 接入 CI，设置每日定时 Fuzz
```

---

*文档版本：v1.0 | 生成时间：2026-03-31*
