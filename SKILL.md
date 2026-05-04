---
name: xhs-find
description: 小红书智能找搭子 - 搜索帖子、质量筛选、智能评论、回复监控、表格记录
---

# 小红书 AI 找搭子

根据用户需求，自动在小红书搜索相关帖子，智能筛选真人帖子，生成个性化评论引流到私信，监控回复，记录追踪防止重复。

**底层工具**: `xiaohongshu-cli` ([jackwener/xiaohongshu-cli](https://github.com/jackwener/xiaohongshu-cli)) — 逆向 API，不用浏览器，内置反检测。

**微信/飞书集成**: 参考 [WECHAT-ADAPTER.md](./WECHAT-ADAPTER.md) 了解如何通过即时通讯工具使用本功能。很重要，因为这个skill很重要的功能就是和微信、飞书等工具打通使用，需要了解适配的技能。

## 前置要求

### 1. 安装 xiaohongshu-cli

```bash
pip install xiaohongshu-cli
```

### 2. 登录小红书账号

使用二维码登录：

```bash
xhs login --qrcode
```

扫码后 Cookie 会自动保存，有效期约 7 天。过期后重新执行登录命令即可。

详细说明参考 [xiaohongshu-cli 文档](https://github.com/jackwener/xiaohongshu-cli)。

## 触发条件

当用户提到以下任何关键词或场景时，应该触发本 skill：

**核心触发词**：
- 搭子、找搭子、搭子app
- 小红书、xhs、红书
- 找人、找朋友、找伴
- 组队、约、一起

**活动类型**：
- 爬山、徒步、登山、户外、露营、野餐
- 探店、吃饭、美食、火锅、咖啡、奶茶
- 看展、展览、博物馆、艺术展、画展
- 运动、健身、跑步、瑜伽、游泳、打球、羽毛球、篮球、网球
- 看电影、电影、观影、影院
- 旅游、旅行、自驾、周边游
- 摄影、拍照、写真
- 剧本杀、桌游、密室逃脱
- 学习、自习、图书馆、考研
- 遛狗、撸猫、宠物

**意图表达**：
- "帮我找XX搭子"
- "小红书找XX"
- "想找XX的人"
- "有人一起XX吗"
- "周末想XX"
- "找个人一起XX"
- "求XX搭子"
- "XX组队"
- "约XX"
- "一起去XX"
- "谁要XX"
- "有没有人XX"
- "找伴XX"
- "寻找XX伙伴"

**示例触发语句**：
- "帮我在小红书找爬山搭子"
- "想找人一起看展"
- "周末有人约咖啡吗"
- "北京找个跑步搭子"
- "求组队去露营"
- "找搭子"
- "小红书搭子"
- "xhs找人"

**宽松匹配原则**：只要用户提到"找"+"活动"，或"搭子"+"任何词"，或"小红书"+"找/搭子"，都应该触发。

## 首次运行配置

第一次使用时，引导用户配置并保存到 `config.json`：

```json
{
  "user_profile": {
    "name": "小明",
    "age": 25,
    "city": "北京",
    "interests": ["爬山", "探店", "摄影"],
    "bio": "周末喜欢户外活动，想认识更多朋友"
  },
  "search_defaults": {
    "time_range_days": 3,
    "max_comments_per_run": 20,
    "comment_interval_seconds": [3, 5]
  },
  "filters": {
    "min_comment_count": 0,
    "max_comment_count": 10,
    "keyword_blacklist": []
  }
}
```

询问用户：
1. 你的昵称、年龄、所在城市？
2. 你的兴趣爱好？（用于生成自然评论）
3. 简单介绍下自己？（用于评论个性化）
4. 默认搜索最近几天的帖子？（默认3天）
5. 每次最多评论多少条？（默认20条）

## 工作流

### Step 1: 解析需求

从用户输入中提取：
- **活动类型**: 爬山、探店、打球、看展、徒步...
- **地点**: 城市/区域，优先同城，默认不限
- **时间范围**: 找最近几天的帖子，默认读取 config.json
- **其他要求**: 人数、性别偏好等

### Step 2: 生成搜索词

根据需求生成 2-3 组搜索词：
- 直接型: "爬山搭子 北京"
- 组队型: "周末爬山 组队"
- 笼统型: "户外徒步 找搭子"

如果用户配置了城市，优先加上城市关键词。

### Step 3: 搜帖子

对每组搜索词，用 `xhs search` 搜索（按最新排序）：

```bash
PYTHONIOENCODING=utf-8 xhs search "搜索词" --sort latest --json
```

从 JSON 输出提取每个帖子的：
- `id` → feed_id
- `xsec_token` → 用于构造链接
- `note_card.display_title` → 标题
- `note_card.user.nick_name` → 作者昵称
- `note_card.user.user_id` → 作者ID
- `note_card.corner_tag_info[0].text` → 发布时间（如"8小时前"、"1天前"）
- `note_card.interact_info.comment_count` → 评论数

### Step 4: 查重

读取 `commented-posts.csv`，检查 feed_id 是否已存在。跳过已评论的帖子。

### Step 5: 时间筛选

只保留发布时间在范围内的帖子。`corner_tag_info[0].text` 格式：
- "X小时前" → 当天
- "X天前" → 对应天数
- 超过 N 天的跳过

### Step 6: 帖子质量筛选

**目标**: 过滤营销帖，只保留真人真诚找搭子的帖子。

Agent 需要判断每个帖子：
1. **标题和内容分析**:
   - 是否包含真实活动细节（时间、地点、活动内容）
   - 语气是否自然、真诚
   - 是否有明显营销痕迹（推广、代发、引流到其他平台）

2. **评论数筛选**:
   - 读取 `config.json` 中的 `min_comment_count` 和 `max_comment_count`
   - 过滤掉评论数过多的热门帖（竞争激烈）
   - 过滤掉评论数为 0 的冷门帖（可能是僵尸号）

3. **关键词黑名单**:
   - 读取 `config.json` 中的 `keyword_blacklist`
   - 如果标题或内容包含黑名单词，跳过

4. **地理位置优先**:
   - 如果帖子内容提到城市，且与用户配置的城市匹配，优先保留
   - 如果帖子带定位信息，优先同城

### Step 7: 作者画像分析

对通过质量筛选的帖子，检查作者活跃度：

```bash
PYTHONIOENCODING=utf-8 xhs user-posts <user_id> --json
```

分析作者：
- 最近发帖频率（最近7天发了几条）
- 粉丝数和获赞数（判断是否真实用户）
- 如果作者最近30天没发过帖，或粉丝数异常（过高或过低），跳过

**注意**: 这一步会增加 API 调用，每个作者查一次。如果帖子数量多，可以只对前 N 个候选帖子做作者分析。

### Step 8: 智能评论生成

**不再使用固定模板**。Agent 需要：

1. 读取用户 profile（`config.json` 中的 `user_profile`）
2. 读取帖子标题和内容
3. 结合用户的兴趣、城市、个人介绍，生成自然的评论

**评论要求**：
- 长度 15-30 字
- 语气自然、真诚
- 体现共同兴趣或同城优势
- 引导私信但不要太生硬
- 避免千篇一律

**示例**：
- 帖子: "周末想去香山爬山，有人一起吗？"
- 用户 profile: 北京，喜欢户外
- 生成评论: "我也在北京！周末正好有空，私信聊聊？"

直接用 note_id 评论，**必须加 `--json` 获取 comment_id**：

```bash
PYTHONIOENCODING=utf-8 xhs comment <note_id> -c "生成的评论内容" --json
```

返回示例：
```json
{
  "ok": true,
  "data": {
    "comment": {
      "id": "69f81f1e000000002203e66c",
      "content": "我也在北京！周末正好有空",
      "note_id": "69f5680500000000350221a7",
      "create_time": 1777868574413
    }
  }
}
```

提取 `data.comment.id` 作为 `comment_id` 保存到 CSV。

**注意**: 每条评论间隔 3-5 秒（读取 `config.json` 中的 `comment_interval_seconds`）。

### Step 9: 记录

将每条评论追加到 `commented-posts.csv`：

```
comment_time,feed_id,comment_id,post_url,author_name,author_id,post_content,post_time,our_comment,keyword_used,status
2026-05-01 14:30:00,69f5680500000000350221a7,69f81f1e000000002203e66c,https://www.xiaohongshu.com/explore/xxx,小明,user123,周末爬山求组队,8小时前,我也在北京！周末正好有空,爬山搭子 北京,commented
```

**关键字段**：
- `comment_id`: 我们评论的 ID（用于精确查找回复）
- `author_id`: 帖主 ID（用于判断是否是作者回复）

### Step 10: 回复监控

用户可以运行 `/xhs-check-replies` 检查已评论的帖子是否有回复。

**工作流**：
1. 读取 `commented-posts.csv`，筛选 `status=commented` 的记录
2. 对每个 `feed_id`，用 `xhs comments <feed_id> --json` 获取所有评论
3. 在返回的 `comments` 列表中，找到 `id` 匹配我们 `comment_id` 的评论
4. 检查该评论的 `sub_comment_count` 是否 > 0
5. 如果有回复，检查 `sub_comments` 列表：
   - 如果有 `user_info.user_id` 等于 `author_id` 的回复，说明作者回复了
   - 更新 CSV 状态为 `replied`，记录回复内容和时间
   - 通知用户："帖主 XX 回复了你的评论：'回复内容'"

**返回示例**：
```json
{
  "comments": [
    {
      "id": "69f81f1e000000002203e66c",
      "content": "我也在北京！周末正好有空",
      "sub_comment_count": "1",
      "sub_comments": [
        {
          "id": "69f820ab000000001f03c123",
          "content": "好啊！私信你了",
          "user_info": {
            "user_id": "user123",
            "nickname": "小明"
          },
          "create_time": 1777868800000
        }
      ]
    }
  ]
}
```

**注意**: 这一步需要大量 API 调用，建议每天运行一次，或用户手动触发。

### Step 11: 输出报告

告知用户：
- 搜索了多少帖子
- 质量筛选后剩余多少
- 作者分析后剩余多少
- 评论了多少帖子
- 跳过了多少（已存在/时间不符/质量不合格）
- 评论了哪些帖主
- 生成的评论示例
- 提示留意私信

## 反检测

工具内置以下防护（无需手动处理）：
- Gaussian jitter 请求间隔
- 一致浏览器指纹 (macOS Chrome)
- 验证码触发后退避 5→10→20→30s
- 所有 API 请求带 `x-s` / `x-s-common` / `x-t` 签名

## 限制策略

- 单次最多评论数读取 `config.json` 中的 `max_comments_per_run`（默认 20 条）
- 评论间隔读取 `config.json` 中的 `comment_interval_seconds`（默认 3-5 秒）
- 同一帖子不重复评论（CSV 防重）
- 只评最近 N 天的帖子（读取 `config.json` 中的 `time_range_days`）

## Cookie 管理

Cookie 由 xiaohongshu-cli 自动管理，存储在 `~/.xiaohongshu-cli/cookies.json`，有效期约 7 天。

过期后执行 `xhs login --qrcode` 重新登录即可。

## 文件结构

**工作目录**: `./dazi/`（相对于 skill 执行目录）

所有生成的文件都保存在 skill 目录下的 `dazi` 子目录，与 skill 文件分离。

### config.json

用户配置文件，首次运行时生成在 `./dazi/config.json`：

```json
{
  "user_profile": {
    "name": "小明",
    "age": 25,
    "city": "北京",
    "interests": ["爬山", "探店", "摄影"],
    "bio": "周末喜欢户外活动，想认识更多朋友"
  },
  "search_defaults": {
    "time_range_days": 3,
    "max_comments_per_run": 20,
    "comment_interval_seconds": [3, 5]
  },
  "filters": {
    "min_comment_count": 0,
    "max_comment_count": 10,
    "keyword_blacklist": []
  }
}
```

### commented-posts.csv

追踪表，记录所有评论过的帖子，保存在 `./dazi/commented-posts.csv`：

| 字段 | 说明 |
|------|------|
| comment_time | 评论时间 |
| feed_id | 帖子 ID |
| comment_id | 我们评论的 ID（用于精确查找回复） |
| post_url | 帖子链接 |
| author_name | 帖主昵称 |
| author_id | 帖主 ID（用于判断是否作者回复） |
| post_content | 帖子摘要 |
| post_time | 帖子发布时间 |
| our_comment | 我们评论的内容 |
| keyword_used | 使用的搜索词 |
| status | commented / replied / ignored |

**注意**: 首次运行时会自动创建 `dazi` 目录。

## 命令

### /xhs-find

主命令，执行完整找搭子流程。

### /xhs-check-replies

检查已评论帖子是否有回复，更新 CSV 状态。

### /xhs-config

查看或修改 `config.json` 配置。
