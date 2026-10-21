# astrbot_plugin_memory_recall

**长期记忆（注入侧）**：接管 AstrBot 的记忆注入，按「状态 / 偏好 / 事件」三层策略走。

版本 `0.2.0` · AstrBot `>= 4.28.2` · 需要 [astrbot_plugin_memory_digest](https://github.com/Lansheen3096/astrbot_plugin_memory_digest) 配合

---

## 为什么接管框架自带的注入

框架自带那条链是 `dense 50 + sparse 50 → RRF → rerank → top N` 一坨平铺，
做不到「状态全量 / 偏好过闸 / 事件时间递减」。所以把知识库的 `kb_names` 置空关掉自带注入，
由本插件接管。

`on_llm_request` 的 hook 点在 `build_main_agent()` **之后**，顺序上本来就是最后一道 ——
能拿到最终 prompt，也能把注入块挂上去。

## 闸门判据

```
地板 FLOOR   top1 ≥ 0.05         实测不相关的噪声区在 0.001 ~ 0.03
领先 LEAD    top1 ≥ LEAD × top2  避免「一堆都不相关、只是有一个略高」时误放
```

**为什么不直接用绝对高分（比如 0.75）：** 20 条真实消息打 48 条真实条目，
top1 中位数 0.013、最大 0.29 —— ≥0.75 是 0/20。rerank 分衡量的是「这条记忆是否真的答上了这句话」，
它的绝对值和直觉不在一个量级，所以判据用**相对领先**。

放行之后**逐条还要过一次地板** —— 否则 top1 勉强过闸时，会把 0.02 的噪声一起带进去。

`state` 层不走闸门（它本来就该每轮都在），只用 `state_max` 兜个底。

## 时间递减

`event` 层分数乘半衰期衰减：`0.5 ** (天数 / halflife_days)`，默认 30 天，有 `decay_floor` 兜底
（不会被压到 0）。`preference` 和 `state` **不衰减** —— 它们本来就是「不依赖某天」才被分到那两层。

## 查询预处理

拿去打分的不是整段 prompt，而是**纯代码清洗**出来的那一句用户输入：去括号壳（动作描写）、
去尾标点、人称归一、补主语。零延迟，实测有效：

```
相关对 top1   0.29 → 0.84
相关对 top1   0.12 → 0.60
```

人称归一：消息里对 bot 的称呼（`ADDRESS_WORDS`）先统一换掉再打分 —— 称呼词对检索没有信息量，
只会稀释相似度。归一目标是 `USER_REF` / `SELF_REF`，和写入侧同名常量必须一致：

```
USER_REF = "[用户]"    用户
SELF_REF = "我"        助手自己
```

⚠️ 代码里几处 `re.sub` 的第二个参数有两种，别看混：`""` 是**整块删掉**（它本身就是噪声），
`" "` 是**换成空格** —— 后者不能删，删了会把左右两个词粘成一个新词，反而污染检索。

## 注入形态

```
【你们现在的状态｜每轮都给】
- …（state 层全量）

【这轮可能用得上的回忆】
- …（过闸的 preference + event）
```

整块挂在 `req.extra_user_content_parts` 上并 `mark_as_temp()` —— **临时注入，不写进历史**，
否则下一轮它会被当成用户说过的话再检索一次，越滚越脏。总长有上限。

## 配置

| 配置项 | 默认 | 说明 |
|---|---|---|
| `kb_name` | `长期记忆` | 必须和写入侧同名配置一致 |
| `trace` | `true` | 记录每轮判据明细（为什么注入 / 为什么没注入） |
| `floor` | `0.05` | 相似度地板 |
| `lead` | `2.0` | 领先倍数 |
| `state_max` | `40` | 状态层最多注入几条 |
| `recall_max` | `3` | 偏好 + 事件最多注入几条 |
| `halflife_days` | `30.0` | 事件半衰期（天） |
| `decay_floor` | `0.35` | 衰减下限 |
| `new_discount` | `0.5` | 待审条目降权（`status="new"`） |

## 安装

```bash
# AstrBot 插件市场搜索「长期记忆（注入侧）」，或：
cd <AstrBot>/data/plugins
git clone --depth 1 --branch v0.2.0 https://github.com/Lansheen3096/astrbot_plugin_memory_recall
git clone --depth 1 --branch v0.2.2 https://github.com/Lansheen3096/astrbot_plugin_memory_digest
# docker 部署：拷进宿主机挂载的数据目录后重启容器
```

**两个插件必须一起装**，两边的 `kb_name` 必须一致；知识库要先建（WebUI → 知识库），
rerank provider 在知识库设置里配。没装 rerank 时只会注入状态层。

## 日志

`[recall]` 前缀。`trace` 打开时每轮会打：清洗后的查询、top 候选分数、闸门判定结果。
排查「为什么这句没被想起来」先看这里。

## 相关

- 复核台（清待审 / 存疑队列）：[astrbot-memory-layer](https://github.com/Lansheen3096/astrbot-memory-layer) 的 `tools/`
- 写入侧：[astrbot_plugin_memory_digest](https://github.com/Lansheen3096/astrbot_plugin_memory_digest)

## License

MIT
