# 小红书智能找搭子 Skill

Claude Code skill，用于在小红书自动搜索、筛选、评论和监控回复，帮助你高效找到合适的搭子。

## 功能特性

- 🔍 **智能搜索**：支持关键词搜索和地理位置筛选
- 🎯 **质量筛选**：自动过滤营销帖，分析作者活跃度
- 💬 **智能评论**：Agent 结合你的 profile 和帖子内容自然生成评论
- 📊 **回复监控**：定期检查评论回复，及时通知
- 🚫 **黑名单过滤**：支持自定义关键词黑名单
- 📝 **历史记录**：CSV 格式保存所有互动记录

## 安装

### 1. 安装前置依赖

需要先安装 [xiaohongshu-cli](https://github.com/jackwener/xiaohongshu-cli)：

```bash
pip install xiaohongshu-cli
xhs login --qrcode
```

### 2. 添加 Skill 到 Claude Code

将本仓库的 `SKILL.md` 文件添加到你的 Claude Code skills 目录：

```bash
# 克隆仓库
git clone https://github.com/mixiazhiyang/zhaodazi.git

# 复制 skill 文件到 Claude Code skills 目录
# macOS/Linux:
cp zhaodazi/SKILL.md ~/.claude/skills/xhs-find-dazi.md

# Windows:
copy zhaodazi\SKILL.md %USERPROFILE%\.claude\skills\xhs-find-dazi.md
```

或者直接从 GitHub 下载 `SKILL.md` 文件放到 skills 目录。

## 使用方法

在 Claude Code 中运行：

```
/xhs-find 搭子关键词
```

### 首次使用

首次运行会引导你配置个人信息：

```
你好！我是 [姓名/昵称]
我在 [城市]
我的兴趣是 [兴趣爱好]
我想找 [找搭子的目的]
```

配置会保存到 `./dazi/config.json`（相对于当前工作目录），后续可以修改。

### 回复监控

定期检查评论回复：

```
/xhs-check-replies
```

建议每天运行 1-2 次，或设置定时任务。

## 配置文件

所有数据保存在 `./dazi/` 目录（相对于执行命令的目录）：

- `config.json` - 用户配置（profile、黑名单、偏好）
- `interactions.csv` - 互动历史记录
- `replied_comments.txt` - 已回复的评论 ID

### 配置示例

```json
{
  "user_profile": {
    "name": "小明",
    "city": "北京",
    "interests": "摄影、徒步、咖啡",
    "purpose": "周末一起拍照、爬山"
  },
  "preferences": {
    "min_likes": 5,
    "min_comments": 2,
    "author_recent_days": 30,
    "prefer_same_city": true
  },
  "blacklist": {
    "keywords": ["广告", "代购", "微商"]
  }
}
```

## 微信/飞书集成

支持通过微信 Bot 或飞书 Bot 使用，详见 [WECHAT-ADAPTER.md](./WECHAT-ADAPTER.md)。

## 工作流程

1. **搜索**：根据关键词搜索小红书帖子
2. **筛选**：
   - 过滤黑名单关键词
   - 检查帖子质量（点赞数、评论数）
   - 分析作者活跃度
   - 优先同城帖子
3. **评论**：Agent 生成自然评论并发布
4. **记录**：保存互动信息到 CSV
5. **监控**：定期检查评论回复

## 注意事项

- 评论频率建议控制在每天 10-20 条，避免被限流
- 首次使用前务必配置个人 profile，让评论更自然
- 定期检查回复，及时响应感兴趣的搭子
- 黑名单关键词可根据实际情况调整
- 确保 xiaohongshu-cli 已登录（`xhs login --qrcode`）

## 许可证

MIT

## 致谢

本项目基于 [xiaohongshu-cli](https://github.com/jackwener/xiaohongshu-cli) 构建，感谢 [@jackwener](https://github.com/jackwener) 提供的强大底层工具。

## 相关链接

- [xiaohongshu-cli](https://github.com/jackwener/xiaohongshu-cli) - 小红书命令行工具
- [Claude Code](https://claude.ai/code) - Claude 官方 CLI 工具
