# cmttree 接线契约（随快照落地，不进题面正文）

这份文件定义 cmttree 内核的**接线**：字段名、配置词表、异常码、结果结构、报告行格式。
它不给出实现；树怎么建、环怎么破、序怎么定、页面怎么画都由实现方自己完成。

## 1. 记录字段（注入的评论记录集合，一条一个字典）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | 字符串 | 是 | 唯一标识；同一标识出现多条时，取**规范 JSON 字节序最小**的那条，其余记 `DUPLICATE_ID` |
| `parent` | 字符串或空 | 否 | 父引用；空/缺省表示是顶层 |
| `author` | 字符串 | 否 | 作者标识，缺省空串 |
| `at` | 字符串 | 否 | 时刻，形态固定 `YYYY-MM-DDTHH:MM:SSZ`；其它形态或缺失 → 记 `BAD_TIME`，该记录的 `at` 置空 |
| `votes` | 整数 | 否 | 票数，缺省 0 |
| `deleted` | 布尔 | 否 | 删除标记，缺省假 |
| `pinned` | 布尔 | 否 | 置顶标记，缺省假 |

## 2. 配置词表（注入的树构建配置）

| 键 | 取值 | 语义 |
|---|---|---|
| `roots` | `no-parent` / `missing-parent` / `deleted-parent` 的任意子集 | 指明哪几种记录**升为根**：父引用为空 / 父标识不在记录集里 / 父记录存在但 `deleted` 为真 |
| `dangling` | `attach-root` / `drop` | 父标识取不到（不在记录集里，或父记录已被丢弃）且 `missing-parent` 不在 `roots` 里时的处置：升为根 / 丢弃并记账 |
| `selfParent` | `root` / `drop` | 父引用等于自身时：升为根 / 丢弃 |
| `cycle` | `promote-min` / `detach-all` | 环的破法：切断环内**标识最小**那条的父引用 / 环内每条的父引用都切断 |
| `maxDepth` | 非负整数 | 深度上限；根的深度为 0 |
| `overDepth` | `hoist` / `cut` | 超深节点：上提到深度上限那一层 / 连同子树剪掉 |
| `order` | 由 `pinned-first`、`at-asc`、`at-desc`、`votes-desc`、`votes-asc`、`id-asc` 组成的列表 | 同级排序键，逐个 token 依次比较 |
| `traversal` | `preorder` / `levelorder` | 遍历序：先自己后孩子、按根序递归 / 按层推进 |

固定规则（不随配置变）：

- 排序的**末位**永远是标识的码位升序，即使 `order` 里没写 `id-asc`。
- 无效时刻（`BAD_TIME` 的那条）排在**所有有效时刻之后**，`at-asc` 与 `at-desc` 都一样；两条都无效时继续比后面的 token。
- 丢弃是**级联**的：一条记录被丢弃后，它的子记录失去父引用，按 `dangling` 那条阶梯继续处置，直到不再变化。
- `hoist` 逐个上提：每次取「当前深度最小、同深度标识升序」的超限节点，把它挂到它在当前树里**深度为 maxDepth-1 的祖先**下（`maxDepth` 为 0 时升为根），使其深度恰为 `maxDepth`，后代随之上移；重复到没有超限节点。
- `cut` 把超限节点连同其子树剔除，**每个**被剔除的节点各记一条 `DEPTH_LIMIT`。

## 3. 异常码（六类，逐条给定位）

| 码 | 触发 | `detail` 形态 |
|---|---|---|
| `ORPHAN_PARENT` | 父标识不在记录集里，或父记录已被丢弃 | `parent=<标识>` / `parent=<标识>;parent-dropped=true` |
| `SELF_PARENT` | 父引用等于自身 | `policy=<selfParent>` |
| `CYCLE` | 沿父引用走回本次路径上的节点 | `members=<环内标识升序逗号分隔>;cut=<被切断的标识或 all>` |
| `DEPTH_LIMIT` | 节点深度超过 `maxDepth` | `policy=cut;depth=<剪掉前深度>` / `policy=hoist;depth=<上提前深度>` |
| `DUPLICATE_ID` | 同一标识出现多条 | `occurrences=<条数>` |
| `BAD_TIME` | 时刻形态不合或缺失 | `at=missing` / `at=invalid` |

异常条目形如 `{"code":..., "id":..., "ids":[...], "detail":...}`：`id` 是主定位（环取被切断的那条），`ids` 是这条异常牵动的**全部**记录标识（多数码只有它自己，`CYCLE` 是环内全部）。异常清单按 `(code, id, detail)` 升序。

## 4. 结果结构（内核导出）

| 键 | 内容 |
|---|---|
| `config` | 规范化后的配置（缺省值补齐） |
| `counts` | `records` / `ids` / `nodes` / `roots` / `edges` / `max_depth` / `positions` / `anomalies` / `dropped` |
| `roots` | 根标识按展示序 |
| `order` | 确定性遍历序的标识列表 |
| `positions` | 标识 → 在 `order` 里的下标（从 0 起） |
| `nodes` | 每个保留节点一条：`id` / `author` / `at` / `votes` / `deleted` / `pinned` / `parent` / `depth` / `children`（已排序）/ `position` / `root_reason` / `anomaly_codes`；按 `id` 升序 |
| `anomalies` | 见 §3 |
| `dropped` | 被丢弃记录的标识升序（丢弃不许无声发生） |
| `invariants` | 逐族 `{"name":..., "checks":<检查数>, "passed":<布尔>}` |
| `digest` | 对**不含 digest** 的规范序列化做 sha256，取前 16 位小写十六进制 |

`root_reason` 取 `null` 或 `dangling` / `missing-parent` / `deleted-parent` / `self-parent` / `cycle-cut` / `hoist`。
规范序列化固定为 `json.dumps(payload, sort_keys=True, separators=(",", ":"), ensure_ascii=True)`。

## 5. 十二族不变量（族名固定，逐族给检查数与成立数）

| 族名 | 要求 |
|---|---|
| `order_is_legal_preorder`（层序时同名换成 `order_is_legal_levelorder`） | 遍历序是树的合法遍历：**前序**下要求父先于子、每棵子树在序里连续；**层序**下要求深度沿序不减，且同深度的相邻两节点满足「父在序里的位置不后退；同父时在子列表里的下标递增」 |
| `each_node_once` | 每个保留节点在遍历序里恰好出现一次 |
| `depth_matches_parent` | 根深度为 0，其余节点深度等于父深度加一 |
| `depth_within_limit` | 任何节点深度都不超过 `maxDepth` |
| `children_sorted` | 每个节点的子列表与根列表都与 `order` 一致（相邻两两比较） |
| `parent_child_symmetric` | 父的子列表里有它，子的父引用指向父 |
| `positions_match_order` | 下标表与遍历序逐位自洽 |
| `no_silent_loss` | 每个出现的标识要么在树里、要么在 `dropped` 里 |
| `cycle_keeps_members` | 环被打破后环内节点一个不少，全部出现在树里 |
| `anomaly_locators` | 每条异常的 `id` 与 `ids` 都是真实出现过的记录标识 |
| `shuffle_invariant` | 把输入记录逆序、右移三位后重跑，导出逐字节相同 |
| `replay_byte_identical` | 同一注入输入重放一次，导出逐字节相同 |

## 6. 命令行与页面

- `python3 -m cmttree report --in <文件>`
  逐行打印（顺序固定）：`records=`、`ids=`、`nodes=`、`roots=`、`edges=`、`max_depth=`、`positions=`、`dropped=`、
  `anomalies=`（六码按 BAD_TIME、CYCLE、DEPTH_LIMIT、DUPLICATE_ID、ORPHAN_PARENT、SELF_PARENT 顺序，形如 `码:条数` 逗号连接）、
  `invariants=<成立族数>/<族数>`、逐族一行 `invariant=<族名>:<检查数>:pass|fail`、
  `shuffle_invariant=`、`cycle_keeps_members=`、`no_silent_loss=`、`replay_byte_identical=`、`digest=`。
- `python3 -m cmttree page --in <文件> --out <页面>`
  写出自包含页面后逐行打印：`page_nodes=`、`page_positions=`、`page_anomaly_codes=`（出现过的异常码种数）、
  `page_external_requests=`（页面里对外引用计数，须为 0）、`page_counts_match_kernel=`（页面内联的计数与内核一致）。
- 页面零外部请求的判据：所有 `src=` / `href=` 取值不含 `://`；全文不出现 `fetch(`、`XMLHttpRequest`、
  `new WebSocket`、`EventSource`、`sendBeacon`、`@import`、`url(http`、`import(`。
- 页面主视图：可折叠评论树（节点给作者、时刻、票数与深度缩进）；被处理过的异常节点换一种样式标注，
  可点开看类别码；右侧给遍历序的序号列表，点序号在树里高亮对应节点；页面上每个数字都来自内核结论。

## 7. 纯性硬要求

内核模块只用 Python 标准库，不读文件系统、不读环境变量、不读系统时钟、不抽随机数、不联网；
输入只有记录集合与配置两个参数；导出里不许出现时钟时刻、随机标识、迭代插入序一类不稳定内容。
