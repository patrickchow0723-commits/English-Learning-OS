# TypeWords 技术调研与 English Learning OS 借鉴方案

> 调研日期：2026-11-16  
> 上游项目：[zyronon/TypeWords](https://github.com/zyronon/TypeWords)  
> 本文是技术参考与适配方案，不是源码移植计划。

## 1. 执行摘要

TypeWords 值得借鉴的核心不是打字软件界面，而是把**主动拼写提取、听觉提示、分阶段练习、错词回流和间隔复习**组织成可重复的训练流程。

对 English Learning OS，建议：

1. 立即借鉴训练方法：跟写、听写、遮挡回忆、拼写默写、句子/文章逐句输入。
2. 借鉴内容组织：词汇与例句、搭配、同义词、词根/词源关联；错误词和收藏项可回流复习。
3. 第一阶段继续以 Markdown 实现：在课程中加入 Typing / Recall / Dictation / Sentence Reconstruction 练习，不开发前端，不复制 TypeWords 的 Vue 组件。
4. 为未来 Evidence 预留字段：记录尝试、正确性、错误类型、提示使用、耗时、复习评分和下次复习时间。
5. 暂不复制 FSRS 或 Supabase 实现：先验证现有学习流程和数据需求，再决定是否需要程序化调度。

## 2. TypeWords 项目概况

根据仓库 README、package manifest、核心类型与练习流程代码：

- 技术栈：Nuxt 4、Vue 3、TypeScript、Pinia。
- 练习方式：单词练习与文章逐句练习；支持跟写、听写、自测、默写等方式。
- 词汇信息：音标、英美发音、释义、例句、短语、同义词、相关词/词根、词源等。
- 学习集合：收藏、错词、已掌握。
- 数据：以浏览器本地存储为主；项目说明提及 IndexedDB / localStorage 与离线使用；设置页包含用户自配 Supabase 的同步配置。
- 依赖：包含 ts-fsrs；类型中包含 FSRS Rating 与 ratingMap。
- 许可证：仓库提供 GPL-3.0 LICENSE。

**重要边界：** README 对智能模式的描述、FAQ 对 FSRS 的说明、当前代码中的练习流程并不完全等价。当前能确认项目依赖 ts-fsrs，并在练习数据结构中保留 ratingMap；但本轮没有完整追踪调度器调用链、数据库 schema 和远端同步冲突策略。因此不能仅凭依赖存在就断言每个练习流程都已经正确使用 FSRS，也不能把“智能模式”理解为完整、透明且可复用的算法实现。

## 3. 代码结构观察

### 3.1 练习交互层

已检查的代表文件：

- app/components/word/TypeWord.vue
- app/components/article/TypingArticle.vue
- app/components/practice-sentences/TypingSentence.vue
- app/components/practice-sentences/TypingSentenceItem.vue

可借鉴的交互思想：

- 练习核心与外层 UI 分离；单词键入核心、识别面板、词汇信息面板分工明确。
- 练习模式决定答案是否可见，用户通过输入而非单纯点击完成提取。
- 听写模式可以先播放音频，用户输入后再显示结果。
- 文章被拆分为 section / sentence / word，支持逐句练习、自动发音与译文显示。
- 练习中的错误可触发回调，由流程控制器决定如何重试和推进。

### 3.2 练习流程层

已检查：

- app/core/types/enum.ts
- app/core/config/env.ts
- app/core/composables/practice-words/usePracticeWordNavigator.ts
- app/core/stores/practice.ts
- app/core/types/types.ts

代码中可见多种练习模式：System、Free、IdentifyOnly、DictationOnly、ListenOnly、Shuffle、Review、Custom 等。System 流程被拆成多个阶段，例如新词跟写/听写/默写，以及旧词自测/听写/默写。Navigator 使用 cursor、node、step、loop 等状态推进流程，并支持从 session snapshot 恢复练习位置。

对 English Learning OS 的启示：训练不应只是一张单词列表，而应当是一组有状态的学习步骤；但第一阶段用课程 Markdown 描述步骤即可，不必复刻其复杂的运行时导航器。

### 3.3 数据模型

TypeWords 的核心词条模型包含：

- word
- phonetic0 / phonetic1
- trans[]（词性、中文释义、频率）
- sentences[]
- phrases[]
- synos[]
- relWords（词根及相关词）
- etymology[]

文章模型包含标题、正文、译文、分段句子、句内词项、音频源和音频定位等信息。

练习数据包含当前索引、词表、错词、排除词、错误次数映射、评分映射和题目数据。统计记录包含学习开始时间、耗时、新词/复习词数量、错误数和学习时间片段。

这些字段适合作为 English Learning OS 未来词汇内容和 Evidence 数据设计的参考，但不建议原样照搬其领域模型。

### 3.4 存储与同步

app/core/utils/supabase.ts 显示项目允许用户在设置页提供自己的 Supabase URL / Key，并将连接配置保存在浏览器 localStorage 中。README/FAQ 说明主要数据在浏览器本地保存，并支持离线使用；跨设备同步需要自行配置 Supabase。

对 English Learning OS 的启示：

- Markdown 仍应作为课程内容的主数据源。
- 学习 Evidence 不应只保存在浏览器临时状态中。
- 如果未来需要跨设备同步，应由 English Learning OS 自己定义身份、权限、数据版本和冲突处理，不直接继承 TypeWords 的同步实现。

## 4. 值得借鉴的能力排序

| 能力 | 价值 | 建议 | 原因 |
|---|---|---|---|
| 主动拼写提取 | 很高 | P0 | 比重复阅读更能暴露拼写和词汇提取问题 |
| 听音拼写 | 很高 | P0 | 连接听力辨音与词汇输出 |
| 错词自动回流 | 很高 | P0 | 将错误转化为下一轮可执行的复习内容 |
| 例句与固定搭配 | 很高 | P0 | 直接服务商务语境，避免孤立背词 |
| 句子逐句跟写/听写 | 很高 | P1 | 适合商务表达、邮件句型和会议表达 |
| 分阶段练习流程 | 高 | P1 | 让学习者从识别逐步走向无提示输出 |
| 收藏/已掌握状态 | 中高 | P1 | 支持个性化清单，但掌握状态必须可复测 |
| FSRS 间隔调度 | 高 | P2 | 有真实复习 Evidence 后再接入 |
| 自定义 Supabase 同步 | 中 | P3 | 当前 Markdown 课程仓库不需要第二套应用后端 |
| 键盘音效/复杂快捷键 | 低 | 暂缓 | 对当前对话教学不是关键路径 |

## 5. English Learning OS 的适配设计

### 5.1 课程层：用 Markdown 定义练习

每个 Lesson 的词汇/表达区建议统一为以下结构：

- Expression：词或固定搭配
- Meaning：中文含义与使用边界
- Pronunciation：IPA（可选）
- Collocation：常见搭配
- Model sentence：贴近 Business × AI 工作的例句
- Contrast：容易混淆的表达
- Recall prompt：隐藏答案后的主动回忆提示
- Dictation prompt：听写或拼写提示
- Output task：要求学习者在新场景中使用

原创教学示例：

| Expression | Meaning | Business × AI example |
|---|---|---|
| align on priorities | 就优先级达成一致 | We need to align on priorities before finalizing the roadmap. |
| address a constraint | 处理一个约束条件 | We addressed the data-access constraint by introducing a permission-aware retrieval layer. |
| measurable business impact | 可衡量的业务影响 | We evaluate the solution based on measurable business impact. |

### 5.2 对话教学层：四步练习循环

1. **Recognize**：先理解词义、发音、搭配和例句。
2. **Recall**：隐藏英文，依据中文含义或场景提示主动说出/写出英文。
3. **Dictate**：由导师给出音频或逐步听写材料；若当前对话环境不能提供音频，必须明确标记为文本拼写练习，不能伪称听力完成。
4. **Transfer**：把目标表达用于一个新的会议、项目、邮件或面试场景。

每轮最多重点训练 5–10 个表达，避免一次性引入过多词汇。

### 5.3 Evidence：建议的最小数据结构

以下是 English Learning OS 的建议结构，不是 TypeWords 现有 schema：

~~~yaml
evidence_id: generated-id
learner_id: local-user
lesson_id: phase-01-lesson-xx
item_id: align-on-priorities
item_type: phrase
attempted_at: ISO-8601
mode: recall # recognize | recall | dictation | transfer
prompt_type: zh_to_en
response: "We need to align the priorities..."
expected_answer: "We need to align on priorities..."
result: partial # correct | partial | incorrect | skipped
error_tags:
  - missing_preposition
hint_used: false
duration_seconds: 18
confidence: medium
next_review_at: null
review_rating: null
~~~

字段说明：

- result 不应只用对/错二值，商务表达允许语义正确但措辞不同。
- error_tags 应区分拼写、词形、搭配、语法、语义和语用错误。
- hint_used 用于区分独立提取与提示后完成。
- next_review_at 和 review_rating 在真实调度器启用前允许为空。
- 原始回答必须保留，不能只保存纠正后的答案。

### 5.4 FSRS 接入边界

TypeWords 的依赖和类型表明其技术栈包含 FSRS 相关能力，但本轮尚未完整审计调度调用链。English Learning OS 应独立定义以下规则：

- 只有发生了可判定的主动回忆尝试，才产生复习评分。
- 单纯阅读、导师讲解或复制示范答案，不应计作一次成功复习。
- 语义等价但措辞不同的回答，需要由评分规则或导师判断后映射到合适等级。
- 发音/听力未实际测量时，不得写入对应能力的成功证据。
- 调度器先在小规模词汇集上验证，再扩展到课程级表达和句型。

## 6. 不建议直接迁移的部分

1. 不复制 TypeWords 的 Vue/Nuxt 组件、Pinia stores 或流程导航代码。
2. 不直接复制它的词库、文章、音频或翻译内容；需要逐项确认内容授权。
3. 不将 TypeWords 的 GPL-3.0 源码直接嵌入具有不同许可证或未来可能闭源的 English Learning OS。若未来确实要复用代码，应先评估 GPL-3.0 对衍生作品分发的义务；本方案优先采用独立实现和原创课程内容。
4. 不把“安装了 ts-fsrs”视为“已经有可靠的自适应学习闭环”。
5. 不在当前阶段引入 Supabase、账户系统或前端学习面板，避免偏离“课程 Markdown + 对话教学”的既定边界。

## 7. 分阶段实施计划

### Phase A — 课程训练增强（现在可做）

- [ ] 在课程规范中增加 Recognize / Recall / Dictate / Transfer。
- [ ] 在 Lesson 01 加入 5–8 个职业介绍与项目表达的主动回忆练习。
- [ ] 为每个表达补充易错搭配与常见错误标签。
- [ ] 增加每课错词/错句复测区。
- [ ] 明确文本听写与真实音频听力的区别。

验收：导师能够仅凭 Markdown 课程执行一轮练习，记录原始回答、错误类型、纠正版本和复测结果。

### Phase B — 统一 Evidence 规范（随后）

- [ ] 在 docs/learning/ 定义 Evidence schema。
- [ ] 定义 phrase / vocabulary / sentence / task 四种学习对象。
- [ ] 定义 correct / partial / incorrect / skipped 判定规则。
- [ ] 定义错误标签词表与掌握状态规则。
- [ ] 将课程练习与 Evidence 的 lesson_id / item_id 关联。

验收：同一表达在不同课程和不同场景中的练习记录可以被关联、复测和追踪。

### Phase C — 复习调度验证（有真实数据后）

- [ ] 先收集至少数轮真实主动回忆记录。
- [ ] 明确评分等级映射及人工评估边界。
- [ ] 对比固定间隔与 FSRS 调度效果。
- [ ] 验证遗忘、连续错误、提示后答对和语义部分正确等情况。
- [ ] 只有在测试通过后，才考虑程序化调度。

验收：复习计划能解释“为什么现在复习这个表达”，并可根据新的练习结果更新；不得把没有证据的掌握状态标为已掌握。

## 8. 当前结论

**English Learning OS 现阶段最值得吸收的是 TypeWords 的练习方法，而不是它的应用代码。**

先把打字/听写理念转化成 Markdown 课程里的主动回忆、拼写、句子重建和情境迁移练习；同时用统一 Evidence 结构保留未来 FSRS 调度所需的数据。这样既符合当前“不开发前端”的产品边界，也能为未来真正的自适应学习打好基础。

## 9. 本次检查的上游文件

- [TypeWords README](https://github.com/zyronon/TypeWords/blob/master/README.md)
- [TypeWords package.json](https://github.com/zyronon/TypeWords/blob/master/package.json)
- [Word practice component](https://github.com/zyronon/TypeWords/blob/master/app/components/word/TypeWord.vue)
- [Article typing component](https://github.com/zyronon/TypeWords/blob/master/app/components/article/TypingArticle.vue)
- [Practice navigator](https://github.com/zyronon/TypeWords/blob/master/app/core/composables/practice-words/usePracticeWordNavigator.ts)
- [Core types](https://github.com/zyronon/TypeWords/blob/master/app/core/types/types.ts)
- [Supabase configuration](https://github.com/zyronon/TypeWords/blob/master/app/core/utils/supabase.ts)
- [GPL-3.0 license](https://github.com/zyronon/TypeWords/blob/master/LICENSE)
