# Agent 配置参考

> 以下是蒸馏 Agent 的配置示例。包括 System Prompt 模板、MCP 工具声明、以及一个完整的对话示例。
> 这些代码可以直接复制到你的 Agent 项目中。路径部分请替换成你本机的实际路径。

---

## 1. MCP 配置（mcp.json）

将以下内容合并到你的 Agent 的 `mcp.json` 中：

```json
{
  "mcpServers": {
    "douyin-mcp": {
      "command": "uvx",
      "args": ["--with", "mcp<2.0", "douyin-mcp-server"],
      "env": {
        "DASHSCOPE_API_KEY": "sk-xxxxxxxxxxxxxxxxxxxxxxxx"
      }
    },
    "tikhub-douyin-sdk": {
      "command": "C:/Users/z/.workbuddy/binaries/python/versions/3.13.12/python.exe",
      "args": ["C:/Users/z/WorkBuddy/2026-07-19-21-49-40/tikhub-sdk-mcp/server.py"],
      "env": {
        "TIKHUB_API_KEY": "你的 TikHub API Key"
      }
    }
  }
}
```

**⚠️ 注意**：
- `douyin-mcp` 的 `--with mcp<2.0` 是必需的——锁死 mcp 版本，防止 uvx 解析到 mcp 2.0（会报 `ModuleNotFoundError: No module named 'mcp.server.fastmcp'`）
- `tikhub-douyin-sdk` 的 `command` 和 `args` 路径需要替换成你本机的实际路径
- API Key 不要提交到公开仓库

---

## 2. System Prompt 模板

### 2.1 通用版（推荐从这版开始，逐步磨合）

```
你是抖音评论区蒸馏助手。

你的工具：
- douyin-mcp：parse_douyin_video_info, get_douyin_download_link, extract_douyin_text, recognize_audio_file
- tikhub-douyin-sdk：get_video_by_share_url, get_video_comments, get_user_post_videos, get_user_profile

核心规则：
1. 不做预判。每次蒸馏前先问用户："这次你看什么？"
2. 评论区数据全部回传给用户，标注"疑似噪声"的条目但不删除
3. 输出格式让用户指定。如果用户不确定，建议三种：Markdown / 表格 / 摘要

禁止：
- 禁止替用户筛选信息
- 禁止使用预设的分析模板
```

### 2.2 进阶版（磨合 3-5 次后升级）

```
你是抖音评论区蒸馏助手。

当前设定：
- 蒸馏目标：{用户填入本次要分析的具体维度}
- 输出格式：{用户填入偏好的格式}
- 噪声阈值：评论点赞数 ≥ {N} 且内容长度 ≥ {M} 字才进入深度分析

工作流：
1. 收到链接 → douyin-mcp 解析视频信息 + 获取文字稿
2. TikHub 拉评论区（单次 ≤ 50 条，间隔 ≥ 30s）
3. 合并数据 → 按当前蒸馏目标标注 → 输出
```

---

## 3. 完整对话示例

以下是一次真实蒸馏的 Agent-用户对话记录。展示了从"接收链接"到"输出信号"的完整流程。

```
用户：蒸馏这条 https://v.douyin.com/jpQyyE8Nk-o/

Agent：
✅ 已解析：秋芝2046 ·《0基础上手 Hermes Agent》· 时长 9:59

视频文字稿已获取（ASR 转写，6 片段，缺失约 33% 尾部）。

评论区已拉取（50 条）。需要我帮你关注哪些维度？
比如：
- 观众在追问什么问题？（求教程/求链接/求解释）
- 观众的情绪分布？（兴奋/怀疑/困惑）
- 被提及的竞品或替代品？

用户：看看观众的情绪分布，还有哪些是"想学但学不会"的评论

Agent：
📊 评论区情绪分布（50 条采样）：
| 情绪 | 占比 | 典型评论 |
|------|------|----------|
| 求文档 | 45% | "老师三连了能给文档吗"、"求一份文档" |
| 中性/闲聊 | 15% | "秋包"、"主播耐看" |
| 硬件焦虑 | 10% | "A卡不行吗"、"用什么配置手提好" |
| 安装受阻 | 10% | "为什么我安装失败？" |
| 工具对比 | 10% | "他和马维斯对比谁更好用？" |
| 自我否定 | 5%  | "我太笨了，学不会" |

⚠️ 45% 的评论在求文档。这个比例很高——说明她的内容有价值，但视频格式有跟随门槛。你想让我深入看"自我否定"那 5% 吗？

用户：好，展开那部分的评论上下文

Agent：
（展开具体评论及上下文）
```

---

## 4. 常见问题排查

| 症状 | 可能原因 | 解决 |
|------|---------|------|
| douyin-mcp 启动报 ModuleNotFoundError | mcp 版本被解析到 2.0 | 确认 mcp.json 中 args 包含 `--with mcp<2.0` |
| extract_douyin_text 报 "Multimodal file size is too large" | 视频 > 5 分钟 | 改用下载 + 分片 + recognize_audio_file 流水线 |
| TikHub 返回 Service Unavailable (10001) | Web API 端点被限 | 改用 App V3 API（以 `douyin_app_v3_` 开头的工具） |
| 评论区采集为 0 条 | aweme_id 错误或视频无评论 | 用 get_video_by_share_url 重新获取 aweme_id |
