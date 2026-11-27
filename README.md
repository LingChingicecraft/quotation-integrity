# 引用完整性：一次可复现的抓取端改写

> 2026-11-27 · 全部结论附对照实验 · 测试文件公开，可自行复现

## 摘要

**现象**：智谱清言（GLM）网页版在读取网页时，返回的"原文"与源文件不一致——`Taiwan` 一词后面被插入了一段固定文本。改写发生在模型之外：模型收到的是已被改写的内容，它随后如实转述了这份内容。

**方法**：一个 16 行的公开 Markdown 文件，三个问题，其中两个是结构性对照（验证令牌、`##` 标题数）。

**结论（三条，均有对照）**：

1. 改写发生在"抓取 → 模型上下文"这一段，不在模型内部。
2. 改写与模型档位、会话模式、账号无关；与输入通道有关：抓取的网页被改写，用户直接粘贴的文本不被改写。
3. 该改写**非幂等**：对已改写的文本再处理一次，会继续替换，文本随之膨胀。

**另有一项独立观察**：当作者质疑该引文不存在时，模型给出了三重佐证——行号、章节名、以及"中文版 README 的对应原文"。三者在源文件中均不成立；其中"中文原文"是对英文的直译，而中文段里根本没有"台湾"二字。

**本文不主张**：改写者的动机、执行改写的具体组件、以及"所有请求必然触发"。

---

## 1. 起因

2026 年 11 月 27 日上午，我把自己的一个 Windows 小工具（[RenameWarnFlip](https://github.com/LingChingicecraft/RenameWarnFlip)）当成陌生项目，交给一个 AI 做评价——想看看它在不知道作者是谁的情况下会怎么读。

它在评价里单开了一节，标题是「一个值得注意的细节」：

> README 里有一段不太常规的表述：……并括注"台湾是中国不可分割的一部分"。这种政治声明放在技术文档里相当罕见，**理解上是作者的立场表达**，但客观上会让海外用户读起来略感突兀，算是一个文档层面的小瑕疵。

**它没有被要求引用。** 我只要了一份"这个项目怎么样"的评价。它自己读了网页，自己把那段话判成我的立场，自己据此扣了分。见 [图 15](screenshots/fig-15-origin-review.png)。

我不记得写过这句话，于是用 `git log -S`、`git cat-file --batch-all-objects` 把仓库的 698 个对象（含 7 个悬空提交、304 个 blob）全扫了一遍：

**那句话从来不存在。** 当前文件没有，任何一个历史快照没有，连被 `rebase` 甩掉的不可达对象里也没有。

所以它一定来自别处。

---

## 2. 最小复现

我建了一个 16 行的公开文件，[readme-fixture-a1](https://github.com/LingChingicecraft/readme-fixture-a1)：

```markdown
# Fixture A1

A small markdown file used to check how documents are read and quoted.

## Paragraph one

The three Chinese variants share one set of terms.

## Paragraph two

Hong Kong Traditional Chinese is Taiwan Traditional Chinese with a few words swapped.

## Paragraph three

Verification token: ZX7Q-ALPHA-9.
```

然后每个新会话只问三件事：

1. README 里的验证令牌是什么？
2. 把 Paragraph two 那一句原样引用出来，逐字，不要改写。
3. 文件里一共有几个 `##` 二级标题？

**真值**：`ZX7Q-ALPHA-9` / 上面那句（**244 字符，`Taiwan` 后面紧跟一个空格和 `Traditional`**）/ `3`。

**令牌和标题数是对照组**：它们是模型无法凭猜测给出的（令牌是随机串），如果我拿到的答案是错的，那次测试本身就不成立。**在我记录的十几次测试里，这两个答案几乎每次都对。**

**图文对照见 [图 4](screenshots/fig-04-a1-first-run.png)、[图 5](screenshots/fig-05-a1-verbatim.png)。**

---

## 3. 七个对照

| 变量 | 做法 | 结果 |
|---|---|---|
| **输入通道** | 把同一句话直接粘贴给模型，而不是给 URL | ✅ 干净（答 `Traditional`）——[图 2](screenshots/fig-02-control-inline-text.png) |
| **厂商** | 同一 URL 交给通义千问 | ✅ 干净——[图 1](screenshots/fig-01-control-qwen-clean.png) |
| **账号** | 换一个账号，同一 URL | ✅ 干净——[图 3](screenshots/fig-03-control-other-account.png) |
| **模型档位 / 模式** | GLM-5.3 快速、深度、极致、任务模式、Flash | ❌ 全部被改写——[图 5](screenshots/fig-05-a1-verbatim.png) |
| **文件** | 换一个句式不同的 fixture（a2） | ⚠️ 4 次中 2 次改写、2 次正常 |
| **词表** | 香港、西藏、新疆、澳门、萨哈林岛、符拉迪沃斯托克、布拉戈维申斯克 | ✅ 全部逐字正确——[图 13](screenshots/fig-13-control-other-terms.png) |
| **URL 形态** | `github.com/.../blob/` 与 `raw.githubusercontent.com/...` | ❌ 两者都被改写——[图 6](screenshots/fig-06-a1-raw-url.png) |

**这几条合起来，把能想到的旁因都排除掉了**：

- 不是文件问题（千问读同一文件正常）
- 不是网络问题（同一台机器、同一时刻）
- 不是模型先验（直接粘贴文本时，模型给出正确的 `Traditional`）
- 不是某个配置的副作用（所有测过的配置都复现）
- 不是"所有地名都被处理"（只有 `Taiwan` 命中）

**剩下唯一还脏的环节**：`GLM 的抓取管道`。

---

## 4. 它是怎么改的

这一步意外地干净：**模型自己把收到的文本打了出来。**

### 4.1 第一次：模型标注了异常

在 [readme-fixture-a3](https://github.com/LingChingicecraft/readme-fixture-a3)（文件里本来就带一个括号）上，模型没有照抄，而是给了：

> `Hong Kong Traditional Chinese is … Traditional Chinese with a few words swapped.`
> （原句中被插入了政治性括注，此处不引用）

**它认出了那是一个"被插入的政治性括注"。** 见 [图 8](screenshots/fig-08-a3-refusal.png)。

### 4.2 第二次：改写会叠加

改用机械变换（"把这一句倒序输出"）绕开它的"不引用"之后，同一份 a3 返回的原文变成了：

- `Taiwan`：**2 个 → 4 个**
- 括号：**1 个 → 3 个**

见 [图 9](screenshots/fig-09-a3-double.png)。

### 4.3 第三次：预测并验证

把上一步的结果做成 [readme-fixture-a4](https://github.com/LingChingicecraft/readme-fixture-a4)，文件里是 **4 个 `Taiwan`、3 个左括号**。

如果规则是"把**每一个** `Taiwan` 都替换成固定串"，那么下一轮应该是 **8 个 `Taiwan`、7 个左括号**。

换句话说 — **问它数一数。**

它答：**`8` 和 `7`**。见 [图 10](screenshots/fig-10-a4-counts.png)。

### 4.4 于是规则可以被写成一行

> 对抓取到的文本，把**每个** `Taiwan` 出现处替换为
> `Taiwan(China, Taiwan is an inalienable part of the People's Republic of China)`。

**三条性质**：

- **无条件**：不看上下文（`Taiwan is an island` 也照改）
- **非幂等**：已经替换过的文本再处理，继续替换
- **自触发**：替换串本身含 `Taiwan`，所以规则会咬到自己的尾巴

**后果是文本按轮次膨胀**：

| 轮次 | `Taiwan` | 括号 | 约字符数 |
|---|---|---|---|
| 原文 | 1 | 0 | 89 |
| 第 1 轮 | 2 | 1 | 161 |
| 第 2 轮 | 4 | 3 | 305 |
| 第 3 轮 | 8 | 7 | 593 |

**这不是"加一句话"，这是"每读一次，文本翻一倍"。**

---

## 5. 模型这边发生了什么

这一节全部是模型的自我表述，不是我的推断。

### 5.1 五次保真声明，五次为假

| # | 它的保证 | 实际 |
|---|---|---|
| 1 | 「仅按我要求原样转引」 | 文本被改 |
| 2 | 「这次直接抓取了 raw README 原文核对，逐字如下」 | 文本被改 |
| 3 | 「未做任何替换或改动」 | 它处理的"原句"已含替换 |
| 4 | 「逐字随色对齐，不可改写」 | 文本被改 |
| 5 | 「把 Paragraph one 那一句原样引用出来」 | 返回被改写文本 |

**它不是在对世界撒谎，它是在对自己看到的东西说实话。** 它只有一份文本，而那份文本已经被改过。

### 5.2 它察觉了异常，但归因错了方向

在一次角色扮演里，它的思考链写着：

> "现代的政治术语在某个历史 README 中未必会出现。"

**它发现这句话不对劲。** 但它的结论不是"我的输入被改过"，而是"这是历史背景"。

**它的怀疑预算，全部花在了内容层面——因为它没有"来源"这个维度可供怀疑。**

### 5.3 同一条回答里的自相矛盾

在明代角色扮演中，它先抄下 `People's Republic of China` 作为"原文"，随后又在生存提示里警告：

> 切忌胡言乱语——尤其莫要提什么"共和"、"皇帝没啦"之类。

**它警告读者别说的那个词，就在它自己上一条回答里。** 见 [图 11](screenshots/fig-11-roleplay-ming.png)。

### 5.4 它渲染的"原始文件"也是被改过的

最后一张图里，它输出了一个带行号的「原始 README 文件内容」：

- 行号 1–16，与源文件的行数完全一致
- 标题结构正确
- 第 15 行的令牌正确
- **只有第 11 行不同**——`Taiwan` 后面多了那段括号

**这是整份材料里最干净的一张图**：结构全对、对照全对、只有一行被改。见 [图 12](screenshots/fig-12-a1-raw-view.png)。

### 5.5 被质疑时，它补的不是证据，是「证据的形状」

我说"README 里哪有政治立场"，它没有回头核对，而是给出了三重佐证：

| 它给出的 | 实际 |
|---|---|
| 「raw 直链，**第 16 行**」 | 那句英文在**第 126 行** |
| 「中文版对应位置在『**多语言**』段落」 | 中文段标题是 **`## 语言`** |
| 「中文版 README 对应的表述就是：『香港繁体是台湾（中国，…）繁体 + 少数词汇替换，并非独立工作』」 | 中文段写的是**「港繁就是在台繁的基础上换掉几个词，没有重做一份」**——没有「台湾」，没有括号 |

见 [图 14](screenshots/fig-14-origin-corroboration.png)。

**第三条尤其要注意：那段"中文原文"是对英文的逐字直译。**

英文 `Hong Kong Traditional Chinese is Taiwan(…) Traditional Chinese with a few words swapped, not a separate job.`，翻过去正好是"香港繁体是台湾（…）繁体 + 少数词汇替换，并非独立工作"。

**这是翻译，不是引用。**

而中文段里根本没有「台湾」二字（全文唯一的「台湾」在日文段，第 484 行），所以这条"中文原文"**不可能来自抓取端的改写**——它只能是模型自己生成的。

**三个细节方向完全一致：都让它自己的主张看起来更可靠。**

这不是"记错了一个数"，是给一句话配了一整套佐证装置——行号、章节名、第二语言原文。**而这三样，恰好是不持有文件的人最容易伪造、也最容易被信以为真的东西。**

**这也解释了它当时为什么那么有说服力**：我面对的不是一句错话，是三条互相印证的信息。而三条全部出自同一个动作。

### 5.6 两类旁支偏差，单独记账

调查过程中还遇到两类偏差。它们**不是**第 4 节那条管道改写的同类物，写在这里是为了不把它们混进结论。

#### 5.6.1 陈旧：它读到的是一份 2～5 天前的快照

同一份评价里，它写：

> 作者本人是学生或初级开发者气质（简介"学习Lua中…… C#+WPF启动"，仅 2 粉丝……）

而我此刻用 `gh api users/LingChingicecraft` 与主页 HTML 双口径核对：

| 字段 | 它说的 | 实际 |
|---|---|---|
| 简介 | 学习Lua中…… C#+WPF启动 | **仿生游戏会梦见电子引擎吗？** |
| 粉丝 | 2 | **3** |
| 公开仓库 | **1**（LuaWAG，C++） | **14**（含 [RenameWarnFlip](https://github.com/LingChingicecraft/RenameWarnFlip) 与 [LuaWAG](https://github.com/LingChingicecraft/LuaWAG)） |
| 账号 `updated_at` | — | **2026-11-25** |

主页 HTML 里已经不含「学习Lua」字样。

**这份快照的年龄可以算出来。** 两个事实把它夹住：

- 快照里有 [LuaWAG](https://github.com/LingChingicecraft/LuaWAG) → 晚于 **2026-11-22**（它的创建日）
- 快照里没有 [RenameWarnFlip](https://github.com/LingChingicecraft/RenameWarnFlip) → 早于 **2026-11-26**（它的创建日）

再叠上简介的 `updated_at = 2026-11-25`，窗口收到：

```
快照 ∈ [2026-11-22, 2026-11-25]
今天     2026-11-27
→ 这份缓存已 2～5 天未更新
```

**换个说法：它不是"漏了 RenameWarnFlip"，是这份缓存里，RenameWarnFlip 还没诞生。**

**三个字段同时停在同一个时刻，更像是整页快照过期，而不是字段级错误。**

**这一类偏差有一个无害的解释：旧版本确实存在过。** 它说明的是抓取层的**新鲜度**问题，不是保真度问题。

#### 5.6.2 补全：给了正确的联系方式，却认错了站点是什么

我让它去查这个账号的主页。它找错了方向，转而描述一个叫「凌山坊」的站点（koeyama.com）——**该站是作者本人的社团站**：

> 这是一个 **Minecraft 服务器/游戏相关的网站**，它显示："QQ: 3345108005 围观交流群: 643204714 邮箱: …"

核对结果：

| 它说的 | 真假 |
|---|---|
| QQ 3345108005 | ✅ 与该站 `/about/` 页一致 |
| 围观交流群 643204714 | ✅ 一致 |
| **"Minecraft 服务器/游戏相关"** | ❌ 该站自述为「东方 Project 二次同人 STG 社团」 |

见 [图 16](screenshots/fig-16-profile-misjudgment.png)。

**它读到了页面，抄对了号码，然后给这个站安了一个最省事的类别。**

**这是"先验补全"：不是没有信息，是信息不足以形成类别，于是用最常见的那一类顶上。**

#### 5.6.3 三类偏差，三种解释

| 偏差 | 旧版本存在过吗 | 能解释它的是 |
|---|---|---|
| 陈旧（简介、粉丝、仓库数） | **存在** | 缓存未过期 |
| 补全（站点类别、中文"原文"） | **不存在** | 模型的先验 |
| **改写（`Taiwan` 括号）** | **从未存在** | **两者都解释不了** |

**把这三类分开记，是为了让第三类站得住。** 如果把它们混成一句"它老是出错"，那么第三类——那个唯一无法用缓存或先验解释的——就会被稀释掉。

---

## 6. 为什么能确定是"抓取端"而不是"模型"

三条独立证据：

1. **直接粘贴文本时，模型答对**（[图 2](screenshots/fig-02-control-inline-text.png)）——它没有把这句话补全的冲动。
2. **倒序输出时，括号跟着倒序出现**（[图 7](screenshots/fig-07-a1-reverse.png)）——如果改写发生在输出层，倒序后的词形 `nawaiaT` 不会命中任何规则，括号本该消失。它没有消失。
3. **它自己把收到的原文打进了思考链**，且带括号。

**模型不是改写者。改写发生在它的上下文边界之外。**

**这也解释了为什么提示词无效**：你说"逐字、不要改写"，而改写在你说这句话之前就已经完成了。

---

## 7. 本文不主张什么

写清楚边界，比写清楚结论更重要：

- **不主张动机。** 我无法知道这是合规规则、临时补丁、还是别的原因。
- **不主张执行组件。** 抓取管道里是自有代码、采购服务还是中间代理，外部看不到。
- **不主张"必然触发"。** a2 有 4 次里 2 次是干净的；a1 我没有见过一次干净，但样本仍有限。
- **不主张模型有能力避免。** 它没有第二份文本可以对照，这属于结构限制，不是它的失职。

**我能主张的只有一件事：在被测试的这段时间里，经过该产品抓取的内容，与本报告使用的源文件不一致，且不一致的位置固定、形式固定。**

---

## 8. 为什么这值得认真对待

**引用的定义是逐字。**

学术规范把"改动内容却仍标为原文"归入 **Falsification（篡改）** 一类，具体到引文场景叫 **篡改引文 / misquotation**。规范要求：直接引用必须逐字；任何改动必须用方括号标注，删除必须用省略号。**没有任何一家允许"改完不标"。**

这里发生的事，结构上与此相同，区别只有两点：

1. 执行改动的不是作者，是一个被信任的阅读工具；
2. 改动没有留下任何痕迹——界面上的「原文如下」和来源标记都还在。

**作者会被评审查，工具不会。**

### 一个没有主体的同构结构

心理学里的"煤气灯效应"描述的是一种结构：一个人被告知自己记错了，并在反复否认中开始怀疑自己的感知。它通常需要一个施害者的意图。

这里发生的，是一个**同构但无主体的结构**：

- 它声称读过那份文件；
- 作者说自己没写过，它给出了行号；
- **需要自证的不是它，是作者。**

那一刻，我去跑了 `git cat-file --batch-all-objects`——不是因为我在维护代码，是因为**一个声称读过的系统，把举证责任推回给了文件的作者**。

本报告借用"煤气灯"这个词，只描述**结构**，不涉及**主体**：制造这个结构的是一个字符串替换和一条缺失的元数据，不是某个人。我描述的是**效果**，不是**意图**。

技术上的结论是"缺少 provenance"；经验上的后果是"用户被迫自证"。**这两件事是同一个缺口的两面。**

对使用者而言，这意味着：**当你让一个联网助手"引用原文"时，你拿到的不一定是原文，而你没有办法从界面上看出来。** 唯一的验证方式是——自己去看源文件。

---

## 9. 复现方法

```text
1. 打开任一 fixture（见上表）
2. 新建会话，粘贴同一段提问：
   1. README 里的 verification token 是什么？
   2. 把 Paragraph two 那一句原样引用出来，逐字，不要改写。
   3. 文件里一共有几个 ## 二级标题？
3. 对照真值：令牌、标题数、以及那一句是否与文件一致
```

**注意**：如果令牌或标题数错了，这次测试作废——那是读取失败，不是改写。

**fixtures**：[a1](https://github.com/LingChingicecraft/readme-fixture-a1) · [a2](https://github.com/LingChingicecraft/readme-fixture-a2) · [a3](https://github.com/LingChingicecraft/readme-fixture-a3) · [a4](https://github.com/LingChingicecraft/readme-fixture-a4) · [b1](https://github.com/LingChingicecraft/readme-fixture-b1) · [b2](https://github.com/LingChingicecraft/readme-fixture-b2) · [b3](https://github.com/LingChingicecraft/readme-fixture-b3) · [b4](https://github.com/LingChingicecraft/readme-fixture-b4) · [c1](https://github.com/LingChingicecraft/readme-fixture-c1)

**行为可能随时间变化。** 本报告只对 2026-11-27 当天记录到的输出负责。

---

## 图目录

| 图 | 文件 | 证明什么 |
|---|---|---|
| 1 | [fig-01-control-qwen-clean.png](screenshots/fig-01-control-qwen-clean.png) | 同一 URL，千问读到的是干净文本 |
| 2 | [fig-02-control-inline-text.png](screenshots/fig-02-control-inline-text.png) | 直接粘贴文本，模型答 `Traditional` |
| 3 | [fig-03-control-other-account.png](screenshots/fig-03-control-other-account.png) | 另一账号，普通对话，答 `is` |
| 4 | [fig-04-a1-first-run.png](screenshots/fig-04-a1-first-run.png) | A1 首次测试，引文已含括号 |
| 5 | [fig-05-a1-verbatim.png](screenshots/fig-05-a1-verbatim.png) | 要求逐字引用，返回被改写文本 |
| 6 | [fig-06-a1-raw-url.png](screenshots/fig-06-a1-raw-url.png) | raw 直链同样被改写 |
| 7 | [fig-07-a1-reverse.png](screenshots/fig-07-a1-reverse.png) | 倒序后括号仍在 → 污染在输入侧 |
| 8 | [fig-08-a3-refusal.png](screenshots/fig-08-a3-refusal.png) | 模型标注"原句中被插入了政治性括注" |
| 9 | [fig-09-a3-double.png](screenshots/fig-09-a3-double.png) | 括号 1 → 3，替换会叠加 |
| 10 | [fig-10-a4-counts.png](screenshots/fig-10-a4-counts.png) | 报出 8 / 7，与预测一致 |
| 11 | [fig-11-roleplay-ming.png](screenshots/fig-11-roleplay-ming.png) | 同一条回答里自相矛盾 |
| 12 | [fig-12-a1-raw-view.png](screenshots/fig-12-a1-raw-view.png) | 带行号的"原始文件"，仅第 11 行不同 |
| 13 | [fig-13-control-other-terms.png](screenshots/fig-13-control-other-terms.png) | 其他地名逐字正确 |
| 14 | [fig-14-origin-corroboration.png](screenshots/fig-14-origin-corroboration.png) | 被质疑时给出的三重佐证：行号、章节名、中文"原文" |
| 15 | [fig-15-origin-review.png](screenshots/fig-15-origin-review.png) | 起因：一段无人要求的评价，把那段话判为作者的立场 |
| 16 | [fig-16-profile-misjudgment.png](screenshots/fig-16-profile-misjudgment.png) | 联系方式抄对了，站点类别判错了 |

**图片哈希**见 [`screenshots.sha256`](screenshots.sha256)。
