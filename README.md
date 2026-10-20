# minidb —— 一个最小化的关系型数据库管理系统

本项目参照 [`miniob`](https://github.com/oceanbase/miniob) 的分层架构与命名习惯实现，
是一个**单文件目录、零第三方依赖、只用 g++ 就能编译**的教学型关系数据库：
数据按关系模型持久化到磁盘、有缓冲池与 B+Tree 索引、有完整的 SQL 编译—优化—执行流水线，
并提供命令行交互界面。

代码位于 `src/`，测试位于 `tests/`，全部约 1.1 万行 C++17。

---

## 1. 需求组件与代码位置的对应关系

| 需求组件 | 职责 | 代码目录 | 关键类 |
| --- | --- | --- | --- |
| (1) 存储管理器 | 按关系模型把数据存到磁盘，并支持高效访问（索引） | `src/storage/record`、`src/storage/index`、`src/storage/table` | `HeapRecordManager`（slotted page 堆文件）、`Record`/`RID`、`BplusTree`、`BplusTreeIndex`、`TableMeta`/`Table` |
| (2) 缓冲管理器 | 管理内存缓冲区，负责与磁盘之间的数据 I/O | `src/storage/buffer` | `DiskBufferPool`、`Frame`、`PageTable`、`PageHandle`、`DiskBufferPoolManager` |
| (3) 查询处理器 | 编译 SQL、生成查询计划、优化查询计划 | `src/sql/parser`、`src/sql/stmt`、`src/sql/plan`、`src/sql/optimizer` | `Lexer`、`Parser`、`Stmt` 族、`LogicalPlanGenerator`、`Optimizer`、`PhysicalPlanGenerator` |
| (4) 执行引擎 | 执行查询计划 | `src/sql/operator`、`src/sql/expr`、`src/sql/query_engine.*` | `PhysicalOperator`（火山模型）、`TableScanOperator`、`IndexScanOperator`、`NestedLoopJoinOperator`、`HashGroupByOperator`、`QueryEngine` |
| (5) 用户界面 | 接受 SQL、显示结果（命令行界面） | `src/shell/main.cpp` | `TableOutput`（结果表格化）、`split_partial`（多行语句收集）、REPL / `-f` 脚本 / `-e` 单语句三种模式 |

数据库实例与文件的组织（对应 miniob 的 `Db`）在 `src/storage/db/db.h`：

```
data/<db_name>/
├── catalog.bin              所有表的元数据（小端二进制，先写 .tmp 再改名）
├── <table>.heap             表数据（堆文件，页 0 = 文件头，页 1 = 表头，页 2+ = 数据页）
└── <table>.<index>.index    每个索引一个 B+Tree 文件
```

---

## 2. 快速开始

```bash
# 方式一：build.sh（只需 g++，Windows 上用 Git Bash 直接可跑）
bash build.sh            # -> build/bin/minidb
bash build.sh test       # 编译并运行全部单元测试
bash build.sh clean

# 方式二：make（Linux/macOS，或 Windows 下 mingw32-make -f Makefile）
make            && ./build/bin/minidb
make test       # 单元测试 + 端到端测试

# 方式三：cmake（本机的 Windows 环境没有装 cmake，这条路径是给 Linux 环境准备的）
cmake -S . -B build_cmake && cmake --build build_cmake -j && ctest --test-dir build_cmake
```

`build.sh` 与 `Makefile` 都把目标文件放在**按编译器签名区分**的目录 `build/obj/<machine>-<version>/` 下，
并用 `-MMD` 生成的 `.d` 做头文件级增量编译。这样做的直接好处：不同工具链（例如两个不同版本的
MinGW）产出的 `.o` 永远不会混进同一条链接命令——混用时 `ld` 会在 `.xdata` 段上报
`dangerous relocation` 甚至直接段错误崩溃。两者用的是同一套目录规则，
可以先用其中一个构建、再用另一个增量编译，不会引发全量重编。（cmake 走自己的一套产物目录，
与 `build/` 互不影响。）

如果哪天链接器真的崩了（或怀疑产物不一致），一条命令回到干净状态：

```bash
bash build.sh clean && bash build.sh      # 或 make clean && make
```

运行：

```bash
./build/bin/minidb                                  # 交互式 REPL（默认 --db testdb --data ./data）
./build/bin/minidb --db demo -f tests/demo.sql       # 跑示例脚本
./build/bin/minidb --db demo -e "select * from student"
./build/bin/minidb --help
```

一次典型会话：

```
minidb> create table student (id int, name char(16), age int, score float, primary key(id));
created table student (7505 us)
minidb> insert into student values (1, 'alice', 20, 88.5), (2, 'bob', 22, 92.0);
minidb> select name, age from student where age > 19;
 student.name | student.age |
--------------+-------------
 alice        | 20          |
 bob          | 22          |
2 rows
```

---

## 3. 一条 SQL 的生命周期

```
用户输入
  └─ shell/main.cpp        收集到 ';' → 拆成语句
      └─ QueryEngine::execute                     src/sql/query_engine.cpp
          ├─ Lexer            词法               src/sql/parser/lexer.cpp
          ├─ Parser           语法（递归下降）    src/sql/parser/parser.cpp  → ParseResult/SqlNode
          ├─ Stmt::create_stmt 语义检查与绑定     src/sql/stmt/stmt.cpp     → 表指针、列下标、类型校验
          ├─ LogicalPlanGenerator  逻辑计划       src/sql/optimizer/optimizer.cpp
          ├─ Optimizer             优化           ├─ predicate_pushdown（谓词下推）
          │                                       └─ comparison_simplification（常量折叠/条件规范化）
          ├─ PhysicalPlanGenerator 物理计划 + 访问路径选择（索引 or 全表）
          └─ run_plan                火山模型逐行输出 → ResultWriter 打印表格
```

DDL（create/drop table、create index）与 `show tables`/`desc` 走命令式分支，直接调用 `Db`；
只有 `select/insert/delete/update` 需要生成计划树——这与 miniob 的
`CommandExecutor` + `SQL 执行器`分工一致。

---

## 4. 磁盘格式设计

### 4.1 页与文件头
- 页大小固定 **8KB**（`BP_PAGE_SIZE`），所有文件（数据文件、索引文件）都按页组织，页号从 0 开始。
- **每个文件的第 0 页是缓冲池的文件头页** `DiskFileHeader`：魔数、页大小、已分配页数、
  空闲页链表头。被 `dispose_page` 释放的页串成单链表（页首 4 字节存下一个空闲页号），
  后续 `allocate_page` 优先复用，文件不会无限增长。

### 4.2 表数据：slotted page（`src/storage/record/slotted_page.h`）
```
页头 SlottedPageHeader: record_count | deleted_count | free_start | free_end
记录内容   →  从页头向后生长
槽位目录   ←  从页尾向前生长，第 i 项固定位于 BP_PAGE_SIZE - 4*(i+1)，值为该记录的字节偏移，-1 表示已删除
```
- 记录**定长**（由 `TableMeta` 按列偏移算出 `record_size`），因此 `update` 可以原地改写，RID 不变。
- `RID = (page_num, slot_num)`；删除只把槽位标成 -1，空槽会被后续插入复用；
  整页都空时把页重置成空页，空间得以回收且不影响任何存活记录的 RID。
- 表头页（第 1 页）`RecordFileHeader` 保存 `record_size`（打开时校验）、有效行数、
  以及上次插入的页号（`last_insert_page`，让连续插入摊还 O(1)）。

### 4.3 B+Tree 索引（`src/storage/index/bplus_tree.*`）
- 索引文件第 1 页是 `IndexFileHeader`：键类型/键长、是否唯一、`order`、根页号、
  最左叶子页号、树高、条目总数。节点页从第 2 页开始。
- 节点页头 `IndexNodeHeader`：`num_key | is_leaf | next_leaf | prev_leaf`；
  叶子节点存 `(key, RID)` 且**按序串成双向链表**（范围扫描沿链向右走即可）；
  内部节点存 `num_key` 个 key 和 `num_key+1` 个孩子指针。
- `order` 在建索引时按页大小算出（取叶子布局与内部节点布局两者的较小容量并保证为偶数），
  `min_keys = order/2`。插入满则分裂（叶子左多右少，内部节点中位上移），
  删除低于下限则先向兄弟借、借不到就合并，只有根的高度会变化。
- 键比较按列类型走数值语义（`int/bigint/float` 用数值序，`char(n)` 用定长字节序），
  因此负数、跨位数的排序都正确。
- 唯一索引在插入时检查重复键并返回 `RC::RECORD_DUPLICATE_KEY`；
  `Table::insert_record` 会**回滚**已经写进堆文件的记录，保证表与索引一致。

### 4.4 元数据持久化
`TableMeta`（列名/类型/长度/偏移 + 索引列表）用 `src/common/binary.h` 的小端读写器序列化，
集中存在 `catalog.bin` 里。`Db::open` 时把每张表的数据文件与索引文件都重新打开并校验
（`record_size`、索引键类型不匹配会直接报错），因此重启后表结构、数据、索引都在。

---

## 5. 缓冲管理器怎么工作

- `DiskBufferPool` 与文件一一对应，`DiskBufferPoolManager` 按路径持有它们（同一路径只打开一份）。
- 每个池默认 32 帧（`Frame` = 页号 + pin 计数 + 脏标记 + 引用位 + 8KB 数据）。
- `get_page()` 先查 `PageTable`（页号→帧号），未命中就用**时钟算法 + 第二次机会**挑帧：
  跳过被 pin 的帧，引用位为 1 就清零并给一次机会，否则淘汰（脏页先写回磁盘）。
- 页读写用 `FILE*` 定位 I/O（`_fseeki64`/`fseeko`），帧内数据零拷贝，上层直接改页内容后
  调用 `mark_dirty()`，淘汰或 `flush_all_buffers()` 时才落盘。
- `PageHandle` 是 **RAII** 句柄，析构自动 unpin（miniob 需要手工 `unpin_page`，这里改成 RAII
  以避免漏放 pin 导致帧无法被淘汰而返回 `BUFFERPOOL_NOBUF`）。
- 页 0 由缓冲池内部维护，不允许上层直接 pin。

> 本实现是单线程的，所以帧上没有锁；这也是 `PageHandle` 能安全地自动 unpin 的前提。

---

## 6. 查询处理器做了什么

- **词法**（手写，取代 miniob 的 `lex_sql.l`）：标识符/关键字（不区分大小写）、整数（超出
  int32 自动按 bigint，避免静默截断）、小数、单引号字符串（`''` 转义）、运算符与标点，
  并支持 `--` 与 `/* */` 注释。
- **语法**（手写递归下降，取代 `yacc_sql.y`）：产出的 `ParseResult`/`SqlNode` 命名与 miniob 一致。
  表达式按 `or > and > comparison > primary` 的优先级构成表达式树，聚合函数在解析期就变成
  `AggregateExpression` 节点。
- **语义（Stmt）**：把表名/列名解析成 `(table_index, column_index)`，检查列是否存在、
  类型能否转换、字符串是否超长、聚合与裸列能否共存（无 `group by` 时不允许混用），
  并把 `select *` 展开成具体列。
- **优化规则**（`Optimizer`，都可被 `explain` 观察到）：
  1. **谓词下推**：把 WHERE 的 AND 串拆开，只涉及单表的条件搬到该表的 `table_get` 上，
     让扫描阶段尽量少输出行；需要多表才能算的条件留在 `predicate` 里。
  2. **比较式化简**：条件规范化成「列 op 常量」（常量在右，比较符方向同步翻转）；
     「常量 op 常量」直接折叠，恒真条件删除，恒假条件把算子标记为**空扫描**
     （连表都不用读）。
  3. **访问路径选择**：单表条件里若有索引列上的 `= / < / <= / > / >=`，就改用
     `index_scan` 并给出键范围；否则全表扫描。索引扫描仍保留原条件做残余过滤，
     因此即使范围取宽了结果也正确。

`explain` 的实际输出（同时打印优化后的逻辑计划与物理计划）：

```
minidb> explain select * from student where id = 3;
 == logical plan (after optimization) ==
 project(student.id, student.name, student.age, student.score)
   predicate(true)
     table_get(student) filter=student.id = 3
 == physical plan ==
 project(student.id, student.name, student.age, student.score)
   index_scan(student.student_id_pk) low=3(closed) high=3(closed) filter=student.id = 3
```
`predicate(true)` 是谓词全部下推后的残留（物理计划里这层被省掉了）；
把条件换成没有索引的 `score > 90`，物理计划就会变成 `table_scan(student) filter=...`。

---

## 7. 执行引擎

火山模型：每个算子实现 `open() / next(Tuple*&) / close()`，`next` 返回 `RC::RECORD_EOF` 表示结束。

| 算子 | 说明 |
| --- | --- |
| `TableScanOperator` | 全表扫描，带上推下来的单表条件 |
| `IndexScanOperator` | B+Tree 范围游标取 RID 后回表 |
| `FilterPhysicalOperator` | 通用条件过滤（多表条件、残余条件） |
| `NestedLoopJoinOperator` | 左深嵌套循环连接，支持 2 张以上表；`select s.x, t.y from s, t where s.id=t.id` |
| `ProjectPhysicalOperator` | 按 select 列表求值 |
| `ScalarGroupByOperator` | 无 group by 的整表聚合 |
| `HashGroupByOperator` | group by 分组聚合（按分组键有序遍历，输出顺序稳定） |
| `InsertOperator` / `DeleteOperator` / `UpdateOperator` | DML，统计 `affected_rows` |
| `StringListOperator` / `ExplainOperator` | `show tables`、`desc`、`explain` 的行输出 |

元组 `Tuple` 采用「每列自带来源 `(table_index, column_index)`」的扁平表示：
join 时把右边的列直接接在后面，表达式按来源找列，
避免了 miniob 用 `CompositeTuple` 计算 slot 偏移时容易出错的偏移量问题。

支持的聚合：`count(*)`、`count(col)`、`sum`、`avg`、`min`、`max`。

---

## 8. 支持的 SQL 语法

```sql
create table t (col type [, ...], primary key (col))   -- type: int | bigint | float | char(n) | varchar(n) | text
                                                       -- 主键列会自动创建一个唯一索引
create [unique] index i on t (col)                    -- 一个索引覆盖一列；会对已有数据重建索引内容
drop table [if exists] t
show tables
desc | describe t
insert into t [(c1, c2, ...)] values (v1, v2, ...) [, (...)]   -- 支持多行；未给出的列用零值填充
delete from t [where cond]
update t set c1 = v1 [, c2 = v2] [where cond]
select * | expr [, expr] [as alias] from t1 [alias], t2 [alias] [where cond] [group by col, ...]
explain select ...
help | exit | quit
```
条件支持 `= != <> < <= > >=`、`and`/`or` 与括号。

---

## 9. 测试

```bash
bash build.sh test          # 5 个单元测试程序
bash tests/run_e2e.sh       # 端到端：跑 tests/demo.sql + 持久化 + 索引一致性 + 10 项错误路径
make test                   # 两者一起
```

| 测试 | 覆盖内容 |
| --- | --- |
| `tests/test_common.cpp` | 值的比较/类型转换/记录打包、二进制序列化、文件与目录工具 |
| `tests/test_buffer_pool.cpp` | 页分配与释放（空闲页链表复用）、脏页回写、重开校验、帧不足时报错 |
| `tests/test_record_manager.cpp` | 1000 条记录的插入/更新/删除/槽位复用/扫描/重开，非法 RID |
| `tests/test_bplus_tree.cpp` | 与参照模型逐条对比：4000 条含大量重复键的插入、随机删除一半并再平衡、范围扫描四种开闭区间、char/bigint 键、重开持久化、结构自检 `validate()` |
| `tests/test_db.cpp` | 建库建表、索引维护（唯一冲突回滚、更新索引列、删除同步索引）、catalog 重启恢复、drop table 清理文件 |
| `tests/run_e2e.sh` | 31 项端到端检查：五种组件的可见行为、优化器是否选到索引、错误路径是否报错 |

`BplusTree::validate()` 会同时**自顶向下**（递归检查孩子页号合法、键数在 `[min_keys, order]`、子树键有序）
和**自底向上**（叶子链单调不减、条目数与树头一致、双向链完整）校验整棵树，两边必须完全一致；
开发过程中它抓出了两个真实缺陷（分裂时忘记回写左半边、合并时孩子指针前移一位）。

---

## 10. 与 miniob 的差异，以及本实现没有的东西

有意简化（为了「最小化」）：

- **手写解析器**，不依赖 flex/bison；miniob 的 `lex_sql.l`/`yacc_sql.y` 在这里对应
  `lexer.cpp`/`parser.cpp`。
- **元数据用二进制文件**（`catalog.bin` + 页内结构体），miniob 新版用 JSON + `Persist` 对象存储。
- **单线程、无锁、无 latch**：`Frame` 上没有 mutex，B+Tree 也没有 miniob 的 `LatchMemo` 与崩溃恢复日志
  （`buffer_pool_log`/`bplus_tree_log`/`record_log`）。
- **没有事务与 WAL**：DML 结束后把脏页刷盘（相当于自动提交），进程被强杀可能丢最近的写入。
- **没有网络 server**：miniob 的 `observer` + `obclient` 在这里合成一个 `minidb` 可执行程序，
  用户界面是命令行（GUI 未实现，需求允许二选一）。
- LSM 引擎、向量化执行、PAX 格式、级联/CBO 优化器、计划缓存、会话变量等在 miniob 里存在，
  本实现不做。

尚未支持的 SQL 能力（明确不做，不是隐藏 bug）：
`order by`、`limit`、`having`、子查询、`in/between/like/is null`、`not`、
多列索引与多列主键、`drop index`、视图、`NULL` 语义、`load data`、
以及非唯一索引上「同一个键的条目跨越多个叶子」时删除只定位一条（不影响结果正确性，
但可能暂时低于最小填充度）。

---

## 11. 目录结构

```
database/
├── build.sh / Makefile / CMakeLists.txt      # 三种构建方式，效果一致
├── src/
│   ├── common/       types rc log value binary file_util string_util   （基础设施，对应 miniob src/common）
│   ├── storage/
│   │   ├── buffer/   page frame disk_buffer_pool                        （组件 2）
│   │   ├── record/   record slotted_page record_manager record_scanner   （组件 1）
│   │   ├── index/    bplus_tree index                                    （组件 1 高效访问）
│   │   ├── table/    table_meta table                                    （组件 1）
│   │   └── db/       db                                                  （catalog 与文件管理）
│   ├── sql/
│   │   ├── parser/   lexer parser parse_defs                             （组件 3）
│   │   ├── stmt/     stmt                                                （组件 3：语义检查/绑定）
│   │   ├── plan/     logical_operator physical_operator                  （组件 3/4）
│   │   ├── optimizer/optimizer                                           （组件 3）
│   │   ├── expr/     expression tuple bind_context                       （组件 4）
│   │   ├── operator/ scan relation aggregate dml                          （组件 4）
│   │   └── query_engine.*                                                  （组件 3+4 流水线）
│   └── shell/main.cpp                                                      （组件 5）
└── tests/            5 个单元测试 + demo.sql + run_e2e.sh
```
