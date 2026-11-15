# llm-sse

OpenAI / Anthropic 兼容的流式对话解码包。

- 状态：**需求层梳理完成，未立项**。仓库只有文档与清单，没有源码。
- 依赖面未核实完：[需求规约 §5.2](docs/requirements.md) 记的是上一次核对结果（三个依赖都没有本包可消费的发布面，JSON 依赖的许可还与本包不兼容），可用 `ct get` 与发布索引复核。
- 版本：0.1.0（需求壳，未立项，不随 phase 移动）。

## 职责

- 把**已打开的响应流**解码成增量事件：文本增量、tool_call 增量、结束、错误。
- 按 `index` 归并 tool_call 分片，流结束时给出可解析的完整参数。
- 构造 chat / messages 请求体（模型、消息、工具定义）。

## 不属于本包

HTTP 传输与重连、SSE 分帧、JSON 词法、模型路由、prompt 模板、本地推理、工具执行。

## 文档

- [需求规约](docs/requirements.md)
- [会话映射](docs/session-map.md)
