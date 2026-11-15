# ct-llm-sse 需求规约

> 状态：需求层梳理完成，**未立项**。
> 仓库：`ct-pkg/llm-sse`（本地 `/Users/samxx/trae-ws/ct-llm-sse`），`main` 零提交；本轮只落文档与清单，没有源码、phase 卡、roadmap。
> 依据：本包任务书（一句话职责）、同批包 `ct-http` / `ct-sse` / `ct-fsguard` 的 `docs/requirements.md`、`ct-json` 实现与清单、`ct-lang` 的包指南与 roadmap。原文一律不改。
> 输入缺口：本仓没有 `docs/proposal.md` / `docs/analysis.md`，也没有既有 README。需求来源是任务书 + 依赖包已冻结的契约；凡本文自行裁定的口径标「本文裁定」并给理由，可推翻，但推翻须同步改判据。
> 边界：只做需求判定；不含设计选型、实现代码、phase 卡、roadmap、阶段切分。

## 0 阅读约定

- 需求编号 `R-<组>-<序号>`：A（契约与传输边界）、B（请求构造）、C（流式解码）、D（tool_call 归并）、E（错误/取消/结束/非流式兜底）、F（工程与交付）。
- 阻塞等级四值：
  - **已支持**：现有 CT 与依赖能力够用；
  - **软阻塞**：能力缺失但有不改语言的绕行，绕行写在条目里；
  - **硬阻塞**：现有 CT 无法实现且无绕行，必须改语言或改需求；
  - **外部阻塞**：本包无法单方解决，取决于别的包或本机工具链；判据照写，但当前无法转绿。
- 判据只接受**可执行**（给定输入 → 期望输出，可断言）与**可核对**（grep / 符号表 / 字节比对）。**「能编译通过」不作为任何条目的判据。**
- 术语：**增量事件** = 一次回调；**槽** = 一个 tool_call 的归并单元；**分片** = 上游一次给的片段；**封口** = 槽从 `OPEN` 转 `CLOSED`。

## 1 一句话需求

把 HTTP 响应体里的 SSE 事件流，按 OpenAI Chat Completions / Anthropic Messages 两种方言解码成四类增量事件（文本、tool_call、结束、错误），按 `index` 归并 tool_call 分片并在流结束时产出可解析的完整参数；本包只做协议语义，不做传输。

## 2 使用边界

| 角色 | 说明 |
| --- | --- |
| 上游 | ct-http 已打开的响应：状态码、content-type、逐块字节、传输结束 / 失败 / 取消 |
| 中游 | ct-sse 把字节分帧成 SSE 事件；ct-json 读写 JSON |
| 下游 | 调用方注册的增量回调；请求体字节由本包构造后交给 ct-http |
| 不属于本包 | HTTP 传输 / 重连 / 超时、SSE 分帧、JSON 词法、模型路由、prompt 模板、本地推理、工具执行 |

## 3 契约面冻结

### 3.1 与传输层的关系

**本文裁定：本包不持有 HTTP 请求，接受一个「已打开的响应流」。**

| # | 冻结口径 | 理由 |
| --- | --- | --- |
| C1 | 输入三件事：① 状态码 + content-type；② 逐块字节；③ 结束 / 失败 / 取消通知 | ct-http 的同步模型只有 `post_stream(cb, ud)`（ct-http R-12），回调就是字节块；本包再持请求会把取消、超时、重试拆到两个包 |
| C2 | 本包不发请求、不读 socket、不解析 chunked、不管鉴权 / URL / 方法 | 包边界；`grep -rn 'socket\|curl_\|Authorization\|https://' src` 必须 0 命中 |
| C3 | 字节入口接受任意切分：一行、一个 UTF-8 字符、一个事件都可被切开 | 与 ct-sse R-A-01 同口径；夹具用字面量分片即可离线 |
| C4 | 输出是增量事件回调；回调返回非 0 立即中断，`feed` 返回 `-1` 并置错误 | 与 ct-sse §4 #4 的 `feed` 口径一致，避免两套中断语义 |
| C5 | 只处理 `text/event-stream` 与整包 `application/json`；其它类型是错误 | 非流式兜底见 R-E-06 |

### 3.2 流式事件模型

| 增量事件 | OpenAI 来源 | Anthropic 来源 | 必有字段 |
| --- | --- | --- | --- |
| TEXT | `choices[0].delta.content` | `content_block_delta.text_delta.text` | `text`, `len` |
| TOOL_CALL | `choices[0].delta.tool_calls[k]` | `content_block_start`(tool_use) + `input_json_delta.partial_json` | `index`, `id` / `name` / `args` 分片 |
| DONE | `data: [DONE]` | `message_stop` | `finish_reason` / `stop_reason` |
| ERROR | `error` 对象、非 2xx、截断、坏 JSON | `event: error` + `error` 对象 | `code`, `message`, `retryable` |

- 一个 SSE 事件最多产生一个增量事件；`ping` / `message_start` / `message_delta` / `content_block_stop` 只改状态、不单独回调。
- 注释行、空 `data:`、未知 `event:`、未知 JSON 成员一律忽略且不报错（与 ct-sse R-A-03 / R-A-06 对齐）。
- 已知成员类型不符（如 `content` 是数字）→ 协议错误，不静默转换。
- 文本增量按到达序回调，不合并、不重排、不做 markdown 解释。

### 3.3 tool_call 增量拼接

| # | 冻结口径 | 理由 |
| --- | --- | --- |
| T1 | 归并键是 `index`；同 index 一个槽，按首次出现建槽 | 两家都有 index；顺序与语义一致 |
| T2 | `id` / `name` / `arguments` 各自按到达序**无条件字节拼接**：不插分隔符、不 trim、不中途解析 | `arguments` 是原始 JSON 文本分片，跨片会切断转义与 UTF-8 序列，中途解析必错 |
| T3 | 槽只有 `OPEN` / `CLOSED`；`CLOSED` 由 `finish_reason == tool_calls`、`[DONE]`、`message_stop` 或该槽 `content_block_stop` 触发 | 「参数何时完整」在流中没有可靠信号，只能在封口判定 |
| T4 | 封口时对拼好的 `arguments` 做一次 JSON 解析：成功 → `COMPLETE`，失败 → 槽级 `TOOL_ARGS_INVALID` 并保留原始串 | 可判定；便于调用方排障 |
| T5 | 枚举顺序按 `index` 升序，不按到达顺序 | OpenAI 的 index 就是最终顺序；乱序到达不得改变输出 |
| T6 | 同槽第二个非空 `id` / `name` 与已存不同 → 协议错误；相同 → 幂等忽略；缺 `index`、`index < 0` → 协议错误 | 不静默覆盖，避免拼出假工具调用 |
| T7 | 槽数有上限（默认 128，可配）；超限 → 错误，不继续建槽 | 防恶意流打穿内存 |

## 4 需求条目

### 4.1 A 组：契约与传输边界

| 编号 | 要解决的问题 → 改后口径 | 验收判据 | 非目标 | 阻塞 |
| --- | --- | --- | --- | --- |
| R-A-01 | 本包是否自己发 HTTP → 不发，只消费已打开响应 | `grep -rn 'curl_\|socket(\|Authorization\|https://' src` = 0；C/D 组全部断言只用字面量字节即可通过 | 不做传输 | 已支持 |
| R-A-02 | 跨块切分不能丢字节 → 字节入口接受任意切分 | 同一段 3 事件流「整块喂」与「逐字节喂」的事件序列、次数、字段值完全一致 | 不做分片调度 | 外部阻塞（ct-sse） |
| R-A-03 | 传输结束必须能进门 → `finish()`（正常 EOF）、`fail(code)`（失败 / 取消）两条入口 | EOF 截在半条 `data:` 后 `finish()` → 1 次 ERROR(TRUNCATED)、0 次 DONE；`fail()` 后不再解析 | 不做重连 | 已支持 |
| R-A-04 | 状态码与类型判定 → 非 2xx 直接 ERROR(HTTP_STATUS)；2xx + `text/event-stream` 走流式；2xx + `application/json` 走兜底；其它 ERROR(UNSUPPORTED_TYPE) | 四种输入各一条断言，错误码逐一对齐 | 不做内容协商 | 已支持 |
| R-A-05 | 缺 content-type → 按 Q-07 口径 | 口径 A：首字节 `{` 判 JSON，否则按 SSE；口径 B：一律按调用方声明。两条判据随口径给 | 不猜 MIME | 已支持 |
| R-A-06 | 两家方言差异 → 构造时显式指定，不做嗅探 | 用错方言必须产生 ERROR 或 0 个增量，不得静默产出错文本 | 不做自动嗅探 | 已支持 |

### 4.2 B 组：请求构造

| 编号 | 要解决的问题 → 改后口径 | 验收判据 | 非目标 | 阻塞 |
| --- | --- | --- | --- | --- |
| R-B-01 | 请求体字节 → 模型、messages（role/content）、tools、`stream: true`，写进调用方缓冲 | 两家方言各一份 golden，逐字节比对（字段顺序冻结） | 不做 URL / 鉴权 | 外部阻塞（ct-json） |
| R-B-02 | 长度查询 → `cap = 0` 返回所需字节数（含 NUL） | 先量后写：写出长度 == 量得 - 1，内容与 golden 一致 | 不做动态分配 | 外部阻塞（ct-json） |
| R-B-03 | 工具定义 → 名字 / 描述 / 参数 Schema 原样嵌入，不校验 Schema；重名 → 本地错误 | golden + 重名负例 | 不做 schema 校验 | 外部阻塞（ct-json） |
| R-B-04 | 内容转义 → 只做 JSON 转义；含裸 NUL 拒绝且不部分写入 | `"a\0b"` → 本地错误，输出缓冲零写入（哨兵比对） | 不解释 markdown | 外部阻塞（ct-json） |
| R-B-05 | 边界 → 鉴权 / URL / 请求头不在本包 | `grep -rn 'api-key\|Authorization\|Accept:' src` = 0 | 不做 HTTP | 已支持 |

### 4.3 C 组：流式解码

| 编号 | 要解决的问题 → 改后口径 | 验收判据 | 非目标 | 阻塞 |
| --- | --- | --- | --- | --- |
| R-C-01 | 文本增量 → `delta.content` 按到达序回调 | "he"+"llo"+" world" → 3 次回调，拼接 == "hello world" | 不合并 | 外部阻塞（ct-sse） |
| R-C-02 | role / 空 delta → `delta.role` 只记录不回调；空 delta 不回调；空 content 按 Q-04 | 首 chunk 只有 role → 0 次回调；`content: ""` 按口径断言 | 不暴露 role（Q-08） | 外部阻塞（ct-sse） |
| R-C-03 | finish_reason → 取最后一个非 null，原样暴露 | 表驱动：stop / length / tool_calls / content_filter | 不映射成语义枚举 | 外部阻塞（ct-sse） |
| R-C-04 | `[DONE]` → 恰一次 DONE；重复 `[DONE]` 忽略；之后的 data 全部忽略 | 双 `[DONE]` + 尾随 data → 1 次 DONE、0 次 TEXT | 不做流后校验 | 外部阻塞（ct-sse） |
| R-C-05 | JSON 跨 SSE 事件 → 按 Q-02，默认不支持 | 默认口径：半截 JSON → ERROR(BAD_JSON)；支持口径才允许通过 | 不做通用分组装 | 外部阻塞（ct-sse） |
| R-C-06 | 坏 JSON → ERROR(BAD_JSON)，停止解析 | `data: {` → 1 次 ERROR，随后合法 chunk 0 次回调 | 不恢复 | 外部阻塞（ct-sse） |
| R-C-07 | 未知成员 / 类型不符 → 未知忽略；类型不符 ERROR(BAD_FIELD) | 两条：未知成员 0 次错误；`content: 1` 1 次 ERROR | 不做宽松转换 | 外部阻塞（ct-sse） |
| R-C-08 | 文本与工具同流 → 两类增量按到达序交错，互不吞并 | 交错流的事件序逐项断言 | 不重排 | 外部阻塞（ct-sse） |

### 4.4 D 组：tool_call 归并

| 编号 | 要解决的问题 → 改后口径 | 验收判据 | 非目标 | 阻塞 |
| --- | --- | --- | --- | --- |
| R-D-01 | 归并按 index → 乱序到达不报错，枚举升序 | 0,2,1 到达 → 枚举 0,1,2，三个槽内容各自正确 | 不按到达序 | 外部阻塞（ct-sse） |
| R-D-02 | 分片拼接 → 三字段字节拼接 | `{"a":` + `1}` → `{"a":1}`；`\"` 被切在两片仍还原 | 不规整空白 | 外部阻塞（ct-sse） |
| R-D-03 | 封口时机 → 四个来源（T3） | `content_block_stop` 只封该槽，其它槽仍 OPEN | 不在中途封口 | 外部阻塞（ct-sse） |
| R-D-04 | 完整性 → 封口时 JSON 解析一次 | `{"a":1}` → COMPLETE；`{"a":` → TOOL_ARGS_INVALID 且原始串保留 | 不做 Schema 校验 | 外部阻塞（ct-json） |
| R-D-05 | 冲突 → id / name 不同则错误，相同则幂等 | 两条：冲突 1 次 ERROR；重复相同 0 次 ERROR | 不静默覆盖 | 外部阻塞（ct-sse） |
| R-D-06 | 边界 → 缺 index、index < 0、槽数超限 → 错误 | 三条 | 不自动分配 index | 外部阻塞（ct-sse） |
| R-D-07 | 访问面 → 按键取槽、读三字段与状态、枚举升序 | 公开符号表与冻结清单一致；槽内容在 delete 后失效（文档 + 负例） | 不提供跨槽合并 | 外部阻塞（ct-sse） |
| R-D-08 | 槽间隔离 → 一槽参数非法不影响其它槽判定 | 两槽一好一坏 → 好槽 COMPLETE、坏槽错误，错误计数 1 | 不做失败即停 | 外部阻塞（ct-json） |

### 4.5 E 组：错误、重试、取消、结束、非流式兜底

| 编号 | 要解决的问题 → 改后口径 | 验收判据 | 非目标 | 阻塞 |
| --- | --- | --- | --- | --- |
| R-E-01 | 错误分类 → 本地参数 / 传输 / HTTP 状态 / 协议 / 工具参数 / 截断 / 取消 七类，码稳定 | 表驱动逐类断言码；文案按 Q-09 口径（精确 / 前缀） | 不做 i18n | 已支持 |
| R-E-02 | 重试归属 → 本包不自动重试，只给 `retryable()`（429 / 5xx / 传输中断为真；协议 / 参数为假） | 表驱动；负例：协议错标 retryable 必须变红 | 不做退避 | 已支持（Q-03 可推翻） |
| R-E-03 | 错误与数据不混 → 一旦 ERROR，之后 0 次 TEXT / TOOL / DONE；`finish()` 返回该错误码 | 坏 JSON 后喂合法 chunk → 0 次额外回调 | 不恢复 | 已支持 |
| R-E-04 | 取消 → `cancel()` 后 `feed` 返回 CANCELLED；`is_cancelled()` 与 `has_error()` 分离；不释放资源 | 取消后喂 3 事件 → 0 次回调、`has_error() == 0`；`delete` 后 ASan 干净 | 不做跨线程取消 | 已支持 |
| R-E-05 | 结束 → DONE 后 `feed` 返回 0 不再解析；`finish()` 幂等；EOF 无 DONE → TRUNCATED | 三条 | 不自动 reset | 已支持 |
| R-E-06 | 非流式兜底 → 整包 JSON 在 `finish()` 时解析：文本 1 次 TEXT + 1 次 DONE；tool_calls 槽 COMPLETE + DONE | 两条 golden | 不做分页 / 续传 | 外部阻塞（ct-json） |
| R-E-07 | 兜底上限 → `max_body_bytes`（默认 4 MiB），超限 ERROR(BODY_TOO_LARGE) | 4 MiB + 1 → 1 次 ERROR；分配计数不随输入线性增长 | 不做磁盘暂存 | 已支持 |
| R-E-08 | 内存 → 正常 / 错误 / 取消 / 截断四路径 ASan 0 泄漏、无 double-free；回调内指针返回即失效 | 四条夹具；负例：回调存指针 → 喂下一事件 → 旧指针内容已变 | 不做借用检查 | 软阻塞（契约 + 负例） |

### 4.6 F 组：工程与交付

| 编号 | 要解决的问题 → 改后口径 | 验收判据 | 非目标 | 阻塞 |
| --- | --- | --- | --- | --- |
| R-F-01 | 包形态 → `ct.pkg` 含 name / version / licence / dep（http、sse、json） | 文件可核对；删任一必填键后 `ct get` 非零退出并给缺键文案 | 不 vendored | 已支持 |
| R-F-02 | 公开面 → 构造 / 方言 / 请求构造 / feed / finish / fail / cancel / 槽查询 / 错误查询；helper 不进生成头 | 生成头导出符号与冻结清单完全一致（脚本比对） | 不新增语言特性 | 已支持（Phase 42） |
| R-F-03 | 依赖断层 → README 状态行必须写「未立项 / 依赖未就绪」，不得出现声称可用的示例 | README 状态行可核对；示例目录不存在或标未验证 | 不假绿 | 已支持 |
| R-F-04 | 夹具离线 → 全部判据用字面量字节 + 假传输层跑通，不访问外网 | 断网跑全量夹具 0 失败（网络端到端另计） | 不做网络桩 | 外部阻塞（ct-sse） |
| R-F-05 | 双份 golden → OpenAI / Anthropic 各至少一段录制流（脱敏）逐字节入仓 | 文件存在且逐字节比对 | 不做隐私清洗工具 | 已支持 |
| R-F-06 | 版本 → `ct.pkg#version` 与 README 版本行一致；本轮 0.1.0，未立项不随 phase 移动 | 两处相等（g0 / `g4-version`） | 不 bump | 已支持 |

## 5 依赖缺口与推进判断

| 依赖 | 实际状态（证据） | 对本包的影响 | 级别 |
| --- | --- | --- | --- |
| ct-http | `src/` 只有 README（`ct-http/src/README.md`）；`ct.pkg` 为 0.1.0，H1 未开工 | 网络端到端不可用；本包以字节为输入，主体不被它卡住 | 外部阻塞（仅 R-F-04 端到端） |
| ct-sse | 只有 docs，无 `ct.pkg`、无 `src`；`ct-sse/README.md:3` 写「分析阶段完成，未立项」 | C / D 组无法落地，R-A-02 无法转绿 | 外部阻塞（主体） |
| ct-json | `json.ct` 全部符号 `static`（`grep -c '^pub '` = 0 命中）；写器入口 `impl JsonWrite for JsonOut` 在 `json.ct:739`；`ct.pkg` 的 `licence = GPL-3.0-only`；`json.ct:1` 要求 `ct-ast` | 跨单元不可导入、无动态缓冲 encoder、许可与 MIT 冲突 | 外部阻塞（B 组 + R-D-04 + R-E-06） |
| ct 编译器 | 本机 `command -v ct` 为空 | 判据本机无法运行 | 环境阻塞（不是语言阻塞） |

推进判断（本文裁定，理由随条）：

1. **先定契约、再等依赖**：§3 的字节入口就是夹具接口，用字面量流写全部判据；不等 ct-http。
2. **不复制依赖实现**：不在本包内自研 SSE 分帧或 JSON 词法。复制会把两套实现一起维护，并让 ct-json 的 GPL 传染进 MIT 包。
3. **C / D 组可先红后绿**：夹具按字节写；ct-sse 到位后只替换内部分帧，判据不变。
4. **B 组与 R-D-04 的处置**：若 ct-json 长期不提供跨单元面与 encoder，立项评审二选一——① 本包只输出结构化请求描述，由调用方序列化（缩能力）；② 该项标「本期不主张」。不得私拷 `json.ct`。
5. **不得做的事**：为了让判据变绿把 `dep` 指向假包、把依赖源码 vendored 进本仓、或把端到端判据标 skip。

**结论：语言硬阻塞 0 条；外部阻塞 3 个包（http 端到端、sse 主体、json 的 B/D/E 三组）；环境阻塞 1 项（本机无 `ct`）。**

## 6 待确认

| 编号 | 问题与可选口径 | 不确认会卡住什么 |
| --- | --- | --- |
| Q-01 | 两家方言：A 同时支持（构造时选）；B 先 OpenAI，Anthropic 下期 | 卡 R-A-06 与 C 组一半判据 |
| Q-02 | 跨 SSE 事件的 JSON：A 支持（要长度 / 括号状态机）；B 不支持（默认） | 卡 R-C-05 |
| Q-03 | 重试归属：A 本包不重试、只判 `retryable()`（默认）；B 本包内置 N 次重试（需幂等与重复 id 去重） | 卡 R-E-02 全部 |
| Q-04 | 空 content：A 回调（保真）；B 丢弃（省调用） | 卡 R-C-02 |
| Q-05 | 是否提供最终文本 / 最终参数 getter：A 提供；B 只回调增量，调用方自拼 | 卡 R-D-07 与 E 组一条 |
| Q-06 | 槽上限默认 128 是否可配、口径是「槽数」还是「槽数 + 总字节」 | 卡 R-D-06 / T7 |
| Q-07 | 缺 content-type：A 看首字节 `{`；B 一律按调用方声明 | 卡 R-A-05 |
| Q-08 | role / usage / finish 之外字段是否暴露 | 卡 C 组读取面 |
| Q-09 | 错误文案是否稳定契约：A 精确比对；B 只保证非空、前缀匹配 | 卡 R-E-01 写法 |
| Q-10 | reasoning 类增量（`reasoning_content` / `thinking_delta`）：A 本期支持并单列第三类增量；B 并入 TEXT；C 不主张 | 卡是否新增事件类型与公开面 |
| Q-11 | 取消粒度：只在 `feed` 边界生效，还是允许回调内重入立即生效 | 卡 R-E-04 |
| Q-12 | 原始分片是否对外暴露（调试用，借用、回调内有效） | 卡公开面与 R-D-07 |
| Q-13 | 非流式兜底触发：只看 content-type，还是 body 首字节兜底（与 Q-07 合并裁定） | 卡 R-E-06 |
| Q-14 | `dep` 包名（http / sse / json）与 ct-json 的 GPL-3.0-only 如何处置（上游改许可 / 上游包化 / 本包不依赖 json） | 卡 `ct.pkg` 与 B 组能否立项 |

## 7 不做什么

1. 不做模型路由、负载均衡、多模型选择、fallback。
2. 不做 prompt 模板、消息裁剪、上下文压缩、token 计数。
3. 不做本地推理、embedding、图片 / 音频等多模态。
4. 不做 HTTP 传输、连接池、重连、超时、代理、TLS、鉴权。
5. 不做 SSE 分帧（ct-sse 负责）、不做 JSON 词法（ct-json 负责）。
6. 不自动执行 tool_call、不校验 JSON Schema、不做参数类型转换。
7. 不做流式落盘 / 断点续传、不做多路复用、不做跨流会话状态。
8. 不新增 CT 语言能力。

## 8 CT 能力依赖汇总

| 能力 | 本包用途 | 证据 | 阻塞 |
| --- | --- | --- | --- |
| `pub struct` + 固有 impl + 静态方法 | 主体类型与构造 | `ct-lang/docs/package-guide.md:27-39` | 已支持 |
| 方法级 `pub`（默认私有） | helper 不进公开头 | `ct-lang/docs/roadmap.md` Phase 42（v0.42.0）；`package-guide.md:34-39` | 已支持 |
| 函数指针 typedef + 字段调用 | 增量事件回调 | `ct-lang/test/ast/27_fnptr_forms/prog.ct`；ct-sse R-B-02 | 已支持 |
| 显式 `impl Drop` + `delete` | 缓冲释放 | ct-http R-02 证据链 | 已支持 |
| 用户定义变参（自由函数） | 错误格式化 | ct-sse §4 #1（方法变参被硬拒） | 已支持（按自由函数） |
| 泛型 struct + Vec / Map | 槽表与缓冲 | `ct-lang/docs/roadmap.md` Phase 29（v0.30.0） | 已支持 |
| 闭包字面量当 C 回调 | 回调注册 | SH-67 限制 | 软阻塞（只收具名函数 / 函数指针） |
| 借用检查 | 回调内指针不逃逸 | 无 | 软阻塞（契约 + 负例，同 ct-sse R-B-04） |
| 自定义 `[]`（L1 / Phase 40） | 不使用 | `ct-lang/docs/design/phase40-draft.md` | 不涉及 |

## 9 未纳入本文件的内容

- 内部状态机、缓冲增长策略、方言解码器拆分、错误文案措辞：属设计层。
- phase 卡、roadmap、阶段切分、任务清单：属立项与排期。
- 版本号移动、tag、发布：本轮只声明 0.1.0，未立项不触发 6.7。
- 实现代码与测试夹具：本轮只做需求。
- 包外承接事项（`publisher` / `ns-root`、依赖全名、公开单元占位）与待决项：见 [backlog.md](backlog.md)。
