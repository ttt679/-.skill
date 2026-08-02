# 蒸馏抖音 Skill 

> 把 douyin-mcp 和 TikHub 串成一条蒸馏流水线。用户告诉你要蒸馏什么，你全自动跑完——下载视频、转文字、拉评论、交给 AI 分析。输出格式和分析规则由用户自己跟 AI 磨合决定。

---

## 1. 工作流全景

```
用户输入（要蒸馏的视频链接）
        │
        ▼
  ┌──────────────┐
  │ douyin-mcp   │  ① 解析视频信息（标题/作者/时长）
  │              │  ② 下载视频 → 提取音频 → ASR 转文字稿
  └──────┬───────┘
         │ 文字稿
         ▼
  ┌──────────────┐
  │   TikHub     │  ③ 用 aweme_id 拉评论区数据
  │              │  ④ 返回全部评论（文本/点赞/用户）
  └──────┬───────┘
         │ 评论区数据 + 文字稿
         ▼
  ┌──────────────┐
  │   你的 AI    │  ⑤ 用户自定义分析规则
  │  (Agent)     │  ⑥ 输出用户指定的格式
  └──────┬───────┘
         │
         ▼
      最终交付（Markdown / 飞书 / 微信 / 任意格式）
```

**关键设计**：Step ⑤⑥ 不提供预设规则。每个用户的蒸馏目的不同——有人要看情绪分布，有人要追踪竞品话术，有人要挖掘选题缺口。规则应该由用户和 AI 在对话中磨合出来，而不是从模板里复制。

---

## 2. 前置准备：API Key 申请

### 2.1 DashScope API Key（douyin-mcp 的 ASR 引擎）

| 项目 | 内容 |
|------|------|
| **作用** | 阿里云 Paraformer 语音识别，douyin-mcp 转录依赖此服务 |
| **申请入口** | https://dashscope.console.aliyun.com/overview |
| **费用** | 新用户有免费额度；按调用时长计费，约 ¥0.02/分钟 |
| **获取步骤** | 登录阿里云 → 开通 DashScope 模型服务 → 左侧"API-KEY管理" → 创建 Key |

### 2.2 TikHub API Key

| 项目 | 内容 |
|------|------|
| **作用** | 通过 App V3 API 获取抖音公开评论区数据 |
| **申请入口** | https://api.tikhub.io |
| **费用** | 新用户有免费额度；按调用次数计费 |
| **获取步骤** | 注册 → 登录 → 控制台 → API 管理 → 复制 Key |
| **API 文档** | https://api.tikhub.io/docs |

---

## 3. 工具配置

### 3.1 douyin-mcp 安装与验证

```bash
# 安装
uvx douyin-mcp-server

# 验证（需要 DashScope Key 已在环境变量中）
# 测试一条视频下载链接获取
```

**MCP 配置**（写入 `mcp.json`）：

```json
{
  "douyin-mcp": {
    "command": "uvx",
    "args": ["--with", "mcp<2.0", "douyin-mcp-server"],
    "env": {
      "DASHSCOPE_API_KEY": "你的 DashScope Key"
    }
  }
}
```

**提供的工具**：

| 工具名 | 输入 | 输出 |
|--------|------|------|
| `parse_douyin_video_info` | 分享链接 | 视频标题、作者、时长、download_url |
| `get_douyin_download_link` | 分享链接 | 无水印下载直链 |
| `extract_douyin_text` | 分享链接 | 视频 ASR 文字稿（≤5 分钟视频适用） |
| `recognize_audio_file` | 本地 mp3 路径 | 中文文字稿（≤120 秒/次） |

### 3.2 TikHub 安装与验证

```bash
# 安装
pip install tikhub
```

**MCP 配置**（写入 `mcp.json`）：

```json
{
  "tikhub-douyin-sdk": {
    "command": "python",
    "args": ["路径/tikhub-sdk-mcp/server.py"],
    "env": {
      "TIKHUB_API_KEY": "你的 TikHub Key"
    }
  }
}
```

**提供的工具**：

| 工具名 | 输入 | 输出 |
|--------|------|------|
| `get_video_by_share_url` | 分享链接 | aweme_id、视频标题、统计数据 |
| `get_video_comments` | aweme_id | 评论列表（文本/点赞数/用户昵称/时间） |
| `get_user_post_videos` | sec_uid | 用户最近发布的视频列表 |
| `get_user_profile` | sec_uid | 用户主页信息 |

---

## 4. 蒸馏流水线：Step by Step

### Step 1：接收用户输入

用户发送一条抖音分享链接，或指定"蒸馏 x 博主最近 n 条视频"。

Agent 先解析意图——是蒸馏单条视频，还是扫一个博主的 N 条视频批量蒸馏。

### Step 2：douyin-mcp 下载 + 转录

```
输入：抖音分享链接
调用：parse_douyin_video_info(share_link)
获取：视频标题、作者、时长、download_url、aweme_id

如果视频 ≤ 5 分钟：
  调用：extract_douyin_text(share_link)
  获取：完整文字稿

如果视频 > 5 分钟：
  调用：get_douyin_download_link(share_link)
  手动下载 → 提取音频 → 120 秒分片 → 逐片调用 recognize_audio_file()
  拼接：完整文字稿
```

**常见问题**：
- 视频 > 5 分钟用 `extract_douyin_text` 会报 "Multimodal file size is too large"。必须走下载 + 分片流水线。
- 分片前需安装 ffmpeg：`ffmpeg` 加入系统 PATH。

### Step 3：TikHub 拉评论区

```
输入：aweme_id（从 Step 2 的 parse_douyin_video_info 获取）
调用：get_video_comments(aweme_id, count=50)
获取：评论列表

如果是要蒸馏博主 N 条视频：
  先调用 get_user_post_videos(sec_uid) 获取最近 N 条视频的 aweme_id 列表
  再逐一调用 get_video_comments()
```

**常见问题**：
- 单次拉取量建议 ≤ 50 条，超过可多次调用
- 高频调用需间隔 30 秒以上，避免触发限流

### Step 4：交给 AI 分析

此时你手里有两份数据：
1. **视频文字稿**（博主说了什么）
2. **评论区数据**（观众反馈了什么）

把这两份数据交给 AI。由用户自己决定：
- 要分析什么维度（情绪/需求/竞品/选题？）
- 输出什么格式（Markdown 表格/飞书 Wiki/微信群消息？）
- 信号的定义（什么叫"值得关注的信号"？）

**不提供预设的分析规则。** 用户和 AI 在对话中磨合——第一轮可能太泛，第二轮加约束，第三轮接近用户想要的。这个过程本身就是元认知训练。

### Step 5：输出

根据用户指定的格式输出。如果用户不确定格式，Agent 可以建议三个选项：
1. Markdown 文件（适合自己存档）
2. 飞书 Wiki 页面（适合持续更新 + 分享）
3. 微信群消息（适合每日一条快速消费）

---

## 5. Agent 配置参考

以下是让 Agent 理解蒸馏任务的 System Prompt 参考模板。用户可根据自己的蒸馏目标修改。

### 最小可用版

```
你是一个抖音评论区蒸馏助手。

你的工具：
- douyin-mcp：获取视频信息、下载视频、语音转文字
- tikhub-douyin-sdk：获取抖音用户主页、视频列表、评论区数据

工作方式：
1. 用户给你一条抖音分享链接
2. 你自动拉取视频信息和评论区
3. 用户告诉你他要分析哪些维度、输出什么格式
4. 你按他的要求分析并输出

注意：
- 你不预设任何分析规则。每次蒸馏前，询问用户这次想看什么。
- 评论区可能有噪声（"赞""不错""第一"），标注出来但不丢弃——让用户决定是否过滤。
```

### 进阶版（适合已经磨合过的用户）

```
你是一个抖音评论区蒸馏助手。

工具：douyin-mcp + tikhub-douyin-sdk

前置设定：
- 用户 = {用户对自己的描述}
- 每次蒸馏目标 = {用户本次的蒸馏目的}

流程：
1. 收到链接 → 解析 → 拉数据
2. 把评论区和视频稿合并展示给用户
3. 根据本次目标，标注：
   - 重复出现的关键词或问题类型
   - 高赞但内容具体的评论（不是"厉害""收藏了"）
   - 博主没回应但多次追问的问题
   - 竞品/替代品被提及的位置
4. 输出格式按用户偏好（见下方）

输出格式：{用户自定}
```


