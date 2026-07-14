# 阿嬷的频道 — 开发笔记

> 本文件记录项目关键约束、技术决策和踩坑经验，供后续开发者参考。

## 一、项目简介

单文件网页应用（index.html），专为老年人设计的方言智能点播网站。
- GitHub 仓库：https://github.com/sagaxm-ops/ama-channel
- GitHub Pages 地址：https://sagaxm-ops.github.io/ama-channel/
- 文件说明：`index.html`（主文件，所有功能都在这一个文件里），`ama-channel.html`（同步副本，解压体验包用）

## 二、硬约束（不可违反）

1. **语音识别必须用国内服务**，Google SpeechRecognition 被禁止（国内网络不通）。
2. **语音识别必须支持**：普通话、莆仙话、闽南话。
3. **iFlytek API 每日 500 次免费调用限制**，超限后必须回退到备用方案。
4. **语音服务切换必须通过配置完成**，不能修改核心逻辑。
5. **三层回退逻辑**：iFlytek → FunASR → Google SpeechRecognition（最终兜底）。
6. **GitHub Pages 部署**，分支为 main，根目录。

## 三、技术决策与代码约定

### 语音识别架构
- 主服务：**科大讯飞（iFlytek）WebSocket 语音听写 API**，使用 HMAC-SHA256 鉴权。
- 方言参数：
  - 中文普通话：`domain='iat', accent='mandarin'`
  - 莆仙话/闽南话：`domain='xfime-mianqie'`（方言免切模式）
- 回退服务：FunASR 公共 WebSocket（`wss://www.funasr.com:10095/`）
- 最终回退：浏览器原生 `webkitSpeechRecognition`

### 状态变量（非常重要）
- `_isRecognizing` / `funasrSession`：麦克风按钮（micBtn）的识别状态
- `_saveRecognizing` / `_saveSession`：搜索栏"录入"按钮（voiceSaveBtn）的识别状态
- **两组状态必须完全独立**，不能共用，否则会导致冲突和内存泄漏。

### 语音识别后匹配逻辑
- 若只有 1 个节目匹配：弹出节目卡片预览 1.5 秒，然后自动播放。
- 若有多个节目匹配：弹出卡片，显示"就是这个"和"换一个"按钮供选择。
- **文字搜索（打字）保持不变**，始终弹出确认卡片。

### 私人语音库
- 数据存储在 `localStorage` 中，键名：`privateVoiceLibrary`
- 家人可以预录长辈的习惯叫法，匹配时优先查询私人语音库。
- 管理界面在网页底部"私人语音库"按钮中。

### 搜索栏按钮
- 输入框 + 麦克风按钮 + 点播按钮 + 听节目名选台 + 录入按钮
- 移动端使用 `flex-wrap: wrap` 自动换行，按钮最小触控区域 44px。

## 四、踩坑记录

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| FunASR 识别能力差、速度慢 | 公共免费服务，算力有限 | 已替换为 iFlytek 作为主服务 |
| iFlytek 报错 11200 | 每日 500 次免费额度用完 | 实现三层回退机制 |
| 私人语音库按钮无法点击 | JS 在 HTML 弹窗之前执行，`getElementById` 找不到元素 | JS 必须放在 `</body>` 之前 |
| 搜索栏按钮手机端溢出 | `flex-wrap: nowrap` + 4 个按钮挤在一行 | 移动端改为 `flex-wrap: wrap`，简化按钮 |
| micBtn 双事件监听器冲突 | `initSpeechRecognition()` 里加了 `{once:true}` 监听器，和全局监听器冲突 | 移除 `initSpeechRecognition()` 里的降级监听器 |
| 语音识别状态冲突 | `funasrSession` 和 `_isRecognizing` 被 micBtn 和 voiceSaveBtn 共用 | 创建独立的 `_saveRecognizing` 和 `_saveSession` |
| 浏览器缓存不更新 | GitHub Pages 有缓存 | 强制刷新：`Cmd+Shift+R`（Mac）或 `Ctrl+F5`（Windows）|
| GitHub push 403 错误 | Personal Access Token 没有 `repo` 权限 | 重新生成带 `repo` 权限的 Token |

## 五、开发注意事项

1. **每次修改 index.html 后，必须同步到 ama-channel.html**，然后一起提交。
2. **提交后等 1-2 分钟**，GitHub Pages 才会更新，然后用强制刷新测试。
3. 修改语音识别相关代码时，务必检查：
   - 状态变量是否独立
   - WebSocket 和音频资源是否正确关闭
   - 两个按钮（micBtn / voiceSaveBtn）是否互斥
4. 不要在 `index.html` 中引入外部依赖（除了 iFlytek WebSocket 和 FunASR WebSocket）。
5. 所有自定义词条保存在浏览器本地，不会上传到服务器。

## 六、后续可优化方向

- 接入更多国内语音识别方案（阿里云、百炼）
- 训练莆仙话专属语音模型（长期目标，需地方文化机构合作）
- 增加更多节目内容（越剧、黄梅戏、豫剧等）
- 支持电视大屏模式
- 优化模糊匹配算法（当前最低分数门槛为 15 分）
