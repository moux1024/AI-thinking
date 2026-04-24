# 基于 `docs/dev-guide.md` 的项目核查与题目回答

## 检查范围

本次回答不是只复述文档，而是结合项目实现核查后给出的判断。主要查看了这些文件：

- `src/main/ipc.ts`
- `src/preload/index.ts`
- `src/main/services/agent-runtime.ts`
- `src/main/services/llm-client.ts`
- `src/main/services/config-store.ts`
- `src/main/services/project-store.ts`
- `src/main/services/workflow-engine.ts`
- `src/main/skills/*.ts`
- `src/renderer/src/stores/story.ts`
- `src/renderer/src/stores/agent.ts`
- `src/renderer/src/components/story/StoryCanvas.vue`
- `src/renderer/src/composables/useStoryTree.ts`
- `src/renderer/src/views/HistoryView.vue`

---

## 1. 为什么采用 `Renderer / Preload / Main` 三层，而不是让 Renderer 直接访问 Node 能力？

当前实现是标准的 Electron 边界设计：窗口创建时开启了 `contextIsolation: true`、关闭了 `nodeIntegration`，Renderer 只能通过 Preload 暴露的 `window.api` 访问主进程能力。对应代码在 `src/main/index.ts` 和 `src/preload/index.ts`。

这样做的核心价值有三点：

1. 安全。Renderer 是最接近用户输入和第三方 UI 依赖的一层，不直接给 Node 能力，能降低任意脚本拿到文件系统、网络、系统 API 的风险。
2. 职责清晰。Renderer 只关心 UI 和状态；文件、网络、Agent、Skill、持久化都集中在 Main。
3. 更可演进。后续要换 IPC 协议、补审计、加权限控制，都能在 Preload/Main 做，不必把 Node 逻辑散到组件里。

如果让 Renderer 直接访问 Node，短期开发会快一点，长期会迅速退化成“组件里直接写文件/请求/系统调用”的耦合代码，安全、测试和迁移成本都会显著上升。

## 2. `Preload + contextBridge + 白名单 API` 的真实价值是什么？如果省略它会怎样？

真实价值不是“多了一层样板代码”，而是把 Electron 的特权能力收口成一个稳定契约。现在 `src/preload/index.ts` 明确只暴露了 `agent / workflow / skill / config / project / story` 这些白名单 API。

如果省略它：

- 短期问题：Renderer 会直接依赖 Electron/Node 细节，页面组件和平台能力绑定。
- 长期问题：接口难统一、参数校验散落、很难审计哪些能力被前端使用，也不利于后续加 mock、埋点、权限和兼容层。

从工程上看，Preload 是“边界适配层”，不是简单转发层。

## 3. 什么逻辑该留在 IPC 层，什么逻辑该下沉到 `services/` 或 `skills/`？

当前 `src/main/ipc.ts` 基本是装配层，这个方向是对的：配置读写、Agent 调用、Skill 执行、Workflow 执行、项目保存都只是注册 handler 后转给服务层。

判断标准我会这样定：

- IPC 层只保留“通道注册、参数透传、事件回推、轻量装配”。
- 业务状态机、外部请求、复杂解析、重试/超时、数据变换，都应下沉到 `services/` 或 `skills/`。

比如现在 `agentRuntime.generateScene()` 放在服务层就是合理的；而如果把 LLM prompt 拼装、图片服务调用、项目文件格式处理都堆进 `ipc.ts`，那 IPC 就会变成无法测试、无法复用的巨石入口。

## 4. 四个业务域的划分是否合理？如何避免未来互相侵蚀？

当前划分为 `Story Tree / Agent / Workflow-Skill / Config-Persistence`，整体是合理的，因为它们分别对应：

- 用户核心创作画布
- 通用对话与工具执行
- 编排能力
- 配置与本地项目生命周期

避免侵蚀的关键不是“目录分开”，而是三条约束：

1. 跨进程契约统一走 `src/shared/`。
2. Renderer 的跨组件状态统一进 store，不让组件本地状态承担业务真相。
3. Main 的对外能力统一经由 IPC，不让 Skill、Workflow、Project 互相直接穿透 UI 层。

这套项目目前大方向是符合的，但闭环还不完全，例如完整故事树仍主要由 Renderer store 持有，Main 端并没有真正掌握树状态。

## 5. `story:get-tree` 返回 `null`，完整故事树仍由 Renderer store 管理，这是合理妥协还是架构债务？

两者都是。

对当前单窗口、单用户、本地创作型产品来说，这是可以接受的阶段性妥协，因为交互和布局都强依赖前端 store，先让故事树在 Renderer 侧跑通，开发效率高。

但它已经是明确的架构债务，因为：

- `src/main/ipc.ts` 里 `story:get-tree` 直接返回 `null`。
- 一旦要做多窗口、恢复、后台生成、协作同步，Main 或后端必须成为状态权威源。
- 现在项目持久化保存的是树快照文件，但 Main 不理解“当前运行中的树状态”。

所以结论是：当前阶段合理，但不能长期停留在这里。

## 6. 为什么 `generateScene` 走快捷路径，而不是完整 Agent tool-calling 循环？

从实现看，`generateScene()` 在 `src/main/services/agent-runtime.ts` 里直接做了两步：

1. 调 `llmClient.chat()` 生成描述和 prompt。
2. 直接拿 `image_generate` skill 生成图片。

它没有走 `run()` 那套完整的“system prompt + tool calling + 多轮循环”流程。

这样设计很合理，原因是：

- 这是高频主链路，时延比通用性更重要。
- 输出格式要稳定，便于直接回填 `description / imageUrl / prompt`。
- 减少模型自主决策，能降低 tool selection 不稳定导致的失败率。

也就是说，这里本质上是“把一个高频固定场景，从通用 Agent 抽成专用服务路径”。

## 7. 为什么 `treeContext` 对分支扩展很关键？

当前前端在热点扩展时通过 `storyStore.getPath(parentId)` 构造了从根到当前节点的路径描述，再传给 `story:generate-scene`。这比只传父节点描述更合理。

原因是子场景不是只依赖“当前节点是什么”，还依赖“它为什么走到这里”。如果只给父节点描述，模型只知道局部画面；给完整路径，模型才知道叙事上下文、视觉递进和空间/情节延续关系。

对故事树产品来说，`treeContext` 其实就是最轻量的叙事记忆。

## 8. `story.ts` 用 `Map<string, SceneNode>` 作为单一事实源，你认可吗？

我认可，但要看到代价。

收益：

- 按 `id` 查找和更新父子关系更直接，复杂度优于频繁扫数组。
- 对树节点的增删改查非常自然。

代价：

- 序列化时必须转数组，当前 `persist()` 就用了 `Array.from(nodes.value.values())`。
- Vue 对 `Map` 的深层变更和普通对象/数组不完全一样，自动追踪更容易踩坑。
- 现在 store 的自动保存只 `watch(() => nodes.value.size)`，深层字段变化并不靠 watch 保证，而是靠各个方法手动 `scheduleSave()`，维护成本更高。

所以 `Map` 作为内存主结构是合理的，但要配套更严格的持久化和变更跟踪策略。

## 9. 2 秒防抖自动保存会在哪些场景失效或带来副作用？怎么改进？

当前 `src/renderer/src/stores/story.ts` 的自动保存策略是 `scheduleSave()` + `setTimeout(..., 2000)`。

风险点有这些：

- 崩溃/强退时，最近 2 秒内改动会丢。
- 没有“窗口关闭前强制 flush”机制。
- `watch(() => nodes.value.size)` 只能捕获节点数量变化，很多深层字段更新靠手工调用 `scheduleSave()`，容易漏。
- 现在 `persist()` 固定保存空 `messages`，说明项目级持久化并不完整。

改进建议：

- 增加退出前同步 flush。
- 用统一的 mutation action 或 store patch 机制代替“分散手工 scheduleSave”。
- 给保存做串行队列，避免并发写盘。
- 引入脏版本号，避免重复保存。
- 明确区分“树快照”“Agent 会话”“配置快照”三类持久化对象。

## 10. `useStoryTree.ts` 这种布局适配层该如何保持可维护？

当前实现是比较干净的：`SceneNode[] -> ELK graph -> position Map`，业务数据和布局数据没有混在一起。

我会坚持三条原则：

1. 适配层必须是纯函数式转换，不持有业务状态。
2. 业务字段不直接泄漏给 ELK，只传布局需要的最小字段。
3. 给“输入节点树 -> 输出坐标”写稳定测试，避免未来改布局参数时破坏结构。

这个项目在这点上做得不错，`src/renderer/src/composables/useStoryTree.ts` 已经基本符合这个模式。

## 11. `SkillRegistry` 同时服务于 Agent tool-calling 和 `skill:execute` 直调，有什么好处和约束？

好处：

- 一套 Skill 定义可以同时给 Agent 和前端直调复用。
- `definition` 同时变成了 UI 展示、工具描述、参数契约的统一来源。
- 降低“Agent 能调、前端不能调”或“前端能调、Agent 不认识”的分裂风险。

约束：

- Skill 的入参和出参必须更稳定，因为它既是工具协议也是 UI 协议。
- description 的质量直接影响模型是否会调用该工具。
- 安全和校验要求更高，不能把“仅给模型看的内部工具”随意暴露成前端可执行能力。

当前项目在统一注册上是对的，但执行前的 schema 校验和权限分层还不够强。

## 12. 如何判断一个 Skill 是否已经足够进入主链路？

从代码看，只有 `image_generate` 明显接入了真实外部服务，且故事树主链路依赖它。其他几个 Skill 仍更像模板骨架：

- `prompt_optimize` 只是返回一个优化任务文本，没有真正做模型优化。
- `storyboard_create` 只是生成空骨架。
- `generation_plan` 是基于简单规则做轻量判断。

我会用这几个标准判断是否能进主链路：

1. 是否连接真实能力，而不是只返回模板。
2. 是否有清晰的错误语义和失败兜底。
3. 输出是否足够稳定，可被下游消费。
4. 是否有最小可观察性，至少能知道失败在参数、模型、网络还是服务端。
5. 是否有基本测试样例。

按这个标准，当前只有 `image_generate` 接近主链路质量，其余仍属于辅助或占位能力。

## 13. 为什么新增 Skill 要先定义 `shared schema`？

这是典型的契约优先。

因为 Skill 一旦进入系统，就会同时影响：

- Agent tool 定义
- `skill:execute` 参数结构
- 前端表单或调用方
- 后续文档和校验

先定义 schema，等于先把“输入输出边界”定住，再写实现。这样可以减少跨进程类型漂移，也方便以后做自动表单、参数校验和工具描述生成。

当前项目已经在这么做，这是对的。

## 14. 如何理解 `LLMClient` 的价值不是“换 SDK”，而是“隔离 OpenAI-compatible 差异”？

看实现就很清楚了。`src/main/services/llm-client.ts` 做的不是简单请求封装，而是兼容层：

- 统一请求地址为 `/chat/completions`
- 支持 `tools`
- 兼容 `thinking`
- 兼容 `reasoning_content`
- 处理流式 SSE
- 处理工具调用参数的增量拼接

这说明它解决的不是“SDK 调用写起来方便”，而是“不同 OpenAI-compatible 供应商返回格式并不完全一致”。把差异收口到这一层，业务层才能稳定使用。

## 15. 如果模型持续产生 tool calls，应该如何设计停机条件、超时和兜底？

当前实现只有一个硬上限：`maxAgentIterations`，默认 20 次。

这还不够。资深实现至少还要补：

- 总耗时上限
- 单次会话的最大 tool call 次数
- 同一工具相同参数重复调用检测
- 工具参数 JSON 解析失败的重试上限
- 单工具超时和熔断
- 用户取消要真正透传到底层请求

这里有一个实际问题：`AgentRuntime.cancel()` 虽然调用了 `AbortController.abort()`，但 `LLMClient.chat()` 里的 `fetch()` 没有传 `signal`，所以现在“取消”在网络请求层面并没有真正生效。这个风险比文档里写得更实。

## 16. `electron-store + 项目 JSON` 这套存储方案适合什么阶段？什么时候该升级？

这套方案很适合当前阶段：

- 单机
- 单用户
- 数据量不大
- 查询诉求弱
- 以“读整个项目 / 写整个项目”为主

从代码看，项目文件就存在 `userData/projects/*.json`，配置则用 `electron-store` 存全局设置。

需要升级到数据库或更复杂存储层的信号包括：

- 项目体积明显增大，需要增量读写
- 需要全文检索、历史版本、分支、回滚
- 需要多人协作或远端同步
- 需要资源索引、素材复用、审计

另外当前实现还有一个很重要的现状：项目保存时 `messages` 总是空数组，说明“项目持久化”现在主要只覆盖故事树，不覆盖 Agent 会话。这也意味着它更像本地草稿保存，而不是完整项目状态管理。

## 17. 为什么扩展节点字段要先改 `shared/types/story.ts`，而不是先改前端展示？

因为节点字段不是纯前端视图字段，而是跨层数据模型。

节点一旦加了新字段，它会同时影响：

- Renderer store 的创建、修改、展示
- 项目文件保存和加载
- IPC 参数和返回
- 未来可能接入的 Main 端处理逻辑

如果只先改前端展示，很容易出现“页面有字段，持久化没保存，加载后又丢”的假闭环。先改 shared type，本质上是先定义系统真相，再改各层消费逻辑。

## 18. 配置修改后立即刷新主进程运行时，如何避免旧实例残留或并发读到脏配置？

当前实现是 `config:set` 后立刻：

- `setConfig(partial)`
- `agentRuntime.updateConfig(getConfig())`
- `workflowEngine = new WorkflowEngine(getConfig())`

这说明项目已经有“配置更新后热刷新运行时”的意识，但还比较粗。

要更稳妥，应该做这些事：

- 用不可变配置快照，单次请求绑定当时版本。
- 长请求不要在中途切换配置。
- 新旧实例切换要可控，必要时 drain 旧请求。
- 外部客户端如 LLMClient、视频服务客户端要有明确生命周期。

当前风险是：运行中的会话可能使用旧配置，后发请求使用新配置；这在单机工具里可以接受，但在复杂并发下会导致难排查的不一致。

## 19. 用户反馈“UI 点击有反应，但故事树没更新”，最小排查路径是什么？

我会按这条链路排：

1. 组件事件有没有触发。看 `SceneNode/StoryCanvas` 的点击和 emit。
2. store 有没有实际更新。尤其是 `addHotspot`、`addChildScene`、`updateNodeImage` 是否执行。
3. 如果是依赖后端生成，`window.api.story.generateScene()` 是否正常返回。
4. 再看 `ipc.ts` 是否注册了 `story:generate-scene`。
5. 最后看 `generateScene()` 内部是 LLM 失败、图片失败还是解析失败。

这个项目里还有一个值得优先检查的真实问题：`StoryCanvas.vue` 在热点成功后调用了两次 `store.addChildScene(...)`。这会导致状态异常，不是纯链路失败，而是逻辑 bug。面对“点击了但树不对”这类反馈，我会把它列为高优先级嫌疑点。

## 20. 图片生成失败时，如何区分是配置错误、Skill 错误、外部服务异常还是主进程逻辑问题？

当前链路可以分层判断：

1. 配置错误：`image_generate` 先检查 `videoService.baseURL`，没配会直接返回明确错误。
2. 外部服务错误：服务返回非 2xx 时，会拼接状态码和响应文本。
3. Skill 逻辑错误：如果参数拼装、响应解析有问题，会落到 catch，返回 `图片生成失败: ...`。
4. 主进程逻辑问题：如果 `generateScene()` 没正确处理 Skill 失败，它可能返回空 `imageUrl`，但不一定抛错。

这最后一点很关键。当前 `generateScene()` 对 `image_generate` 的失败处理偏弱：只在 `result.success && url` 时赋值，否则返回空字符串。也就是说，有些失败不会被明确升级成异常，而会表现成“调用成功但没有图”。

## 21. 如果 Agent 没有调用工具，但普通聊天正常，第一时间怀疑什么？

优先怀疑三类问题：

1. 工具没有正确注入给模型。看 `skillRegistry.getToolDefinitions()` 是否有内容。
2. Skill 描述不够清晰，模型不知道何时应该调用。
3. 模型或接口兼容性问题，虽然能聊天，但不支持或不稳定支持 tool calls。

再细一点看当前项目：

- `run()` 每轮都会把 tools 传给 `llmClient.chat()`，链路本身是有的。
- `LLMClient` 也正确支持 `body.tools` 和 `tool_choice: 'auto'`。
- 但工具效果强依赖 `definition.description`，以及 system prompt 是否足够明确。

所以“不会调工具”在这个项目里更可能是工具描述质量或模型兼容性问题，而不是 Renderer 问题。

## 22. 如何评估 WorkflowEngine 是“值得保留的预埋能力”还是“应删除的过度设计”？

我的判断是：值得保留，但必须明确它还不是生产级能力。

保留的理由：

- 已经有独立执行器、校验、拓扑排序、按层并行执行。
- 和 Skill 体系已经接上了，不是完全空壳。
- 未来如果要做可视化编排，这部分可以直接复用。

不能高估它的原因：

- `condition` 节点的表达式几乎总是返回 `true`。
- `llm-decision` 节点只是选第一个分支。
- `workflowTimeout` 配置存在，但执行器里没真正用到。
- 还没有前端编辑/运行闭环。

所以它不是“该删的过度设计”，但也绝不能按“已完成引擎”对外承诺。

## 23. 如果把这个项目演进成“可协作的团队创作平台”，优先改哪三块架构？

我会优先改这三块：

1. 状态权威源。故事树不能继续只由 Renderer store 持有，必须迁到 Main 或远端服务，至少做到统一版本和冲突处理。
2. 持久化模型。项目 JSON 要升级成可增量、可版本化、可审计的存储方案，图片/素材也要有资产索引。
3. 协作协议。要补用户、权限、锁、评论、分支/合并、操作日志，否则“多人同时改树”无法成立。

如果这三块不先动，UI 再丰富，也只是单机工具外面包一层协作幻觉。

## 24. 如果带 3 人小组继续开发，先补哪些测试、日志和规范以降低未来两个月风险？

我会先补这些：

1. 测试
   `story store` 的节点 CRUD、路径构造、自动保存、项目加载。
   `AgentRuntime` 的 tool-calling 循环、maxIterations、工具失败回写。
   `useStoryTree` 的树到布局转换。
   `WorkflowEngine` 的 DAG 校验、拓扑排序、分支节点行为。

2. 日志
   给 IPC 请求打通道日志。
   给 LLM/Skill/图片服务打分层错误日志和耗时日志。
   给项目保存打成功/失败和文件路径日志。

3. 规范
   新增跨进程能力必须先改 `shared` 契约。
   Renderer 不直接做持久化和网络。
   Skill 必须有 schema、错误语义、最小测试。

如果只补一件事，我会先修“核心路径可观测性”，因为现在很多失败会被吞成空结果，很难第一时间看出是配置错、服务错还是逻辑错。

---

## 额外发现

这些不是题目本身，但和题目判断强相关：

1. `AgentRuntime.cancel()` 目前没有真正中断 `fetch`，取消能力不完整。
2. 项目保存时 `messages` 固定为空，Agent 会话没有纳入项目持久化。
3. `StoryCanvas.vue` 热点生成成功后，`addChildScene()` 被调用了两次，是实质性逻辑 bug。
4. `story:get-tree` 和 `HistoryView.vue` 都还是明显占位实现。
5. Workflow 的条件与决策节点当前更像 stub，不适合当成成熟能力对外描述。

## 总结

这份文档对项目主干方向的描述基本准确，但从代码看，当前仓库更接近“主路径已打通、边缘能力预埋、若干关键非功能特性未闭环”的阶段。

如果面向 5 年以上开发者，我更看重候选人能不能同时看见两件事：

- 这套分层设计本身是对的。
- 但它还没有完全走到“生产级工程闭环”，很多地方已经暴露出下一阶段必须处理的债务。
