# 阿嬷的频道 — 项目交接文档

> 本文档由前一任开发者整理，供新开发者快速上手项目。

---

## 一、项目基本信息

| 项目 | 内容 |
|------|------|
| 项目名称 | 阿嬷的频道 |
| 项目定位 | 专为老年人设计的方言智能点播网站 |
| 当前版本 | 比赛演示版 |
| 仓库地址 | https://github.com/sagaxm-ops/ama-channel |
| 在线演示 | https://sagaxm-ops.github.io/ama-channel/ |
| 部署方式 | GitHub Pages（免费静态网站托管）|
| 技术栈 | 纯前端单文件 HTML（无框架、无后端、无数据库）|

### 文件结构

```
ama-channel/
├── index.html          ← 主文件（所有代码都在这里，约 8500 行）
├── ama-channel.html    ← 同步副本（解压体验包用，内容必须与 index.html 一致）
├── DEV_NOTES.md        ← 开发笔记（约束、踩坑记录、优化方向）
├── HANDOVER.md         ← 本交接文档
├── opera_images/       ← 35 张节目 AI 生成海报
├── audio/              ← 12 种演示音频（部分节目复用）
└── ama-channel-logo.png ← 项目 Logo
```

---

## 二、核心功能清单

### 已实现的 5 种点播方式

1. **浏览器语音输入**（可选）：点击麦克风，用普通话说出戏名，系统自动识别并匹配。
2. **预录入词条库 + 模糊匹配**：通过"录入莆仙话/闽南话"功能，预先录入长辈的习惯叫法，系统模糊匹配。
3. **搜索栏文字输入**：输入中文、方言文字或不完整戏名，五级候选搜索。
4. **听节目名选台（TTS）**：系统朗读节目名，长辈听到想看的按"就是这个"确认。
5. **浏览海报 + 二次确认**：分类浏览海报，点击后弹出确认卡片。

### 私人语音库

- 搜索栏有"🎤 录入"按钮，可以语音录入节目名并保存到私人语音库。
- 网页底部有"私人语音库"管理入口，可以查看、删除、清空已保存的语音指令。
- 数据保存在浏览器 `localStorage` 中，不会上传服务器。

---

## 三、关键配置与凭据

### 科大讯飞（iFlytek）语音听写 API

这是当前主用的语音识别服务，新用户有**每日 500 次免费调用额度**。

配置位置：`index.html` 中搜索 `XUNFEI_CONFIG`

```javascript
const XUNFEI_CONFIG = {
  appId: '你的APPID',
  apiKey: '你的APIKey',
  apiSecret: '你的APISecret'
};
```

**如何获取自己的凭据**：
1. 访问 https://www.xfyun.cn/
2. 注册账号 → 进入控制台
3. 创建应用 → 选择"语音听写"服务
4. 复制 AppID、APIKey、APISecret，替换到代码中

### GitHub Pages 部署

1. 代码推送到 GitHub 仓库的 `main` 分支
2. 仓库 Settings → Pages → Source 选择 "Deploy from a branch" → `main` / `root`
3. 每次推送后等待 1-2 分钟，用 **强制刷新**（`Cmd+Shift+R` 或 `Ctrl+F5`）查看更新

---

## 四、开发规范（务必遵守）

### 1. 语音识别服务切换

所有语音服务切换必须通过配置完成，**不能修改核心逻辑**。当前三层回退：

| 优先级 | 服务 | 触发条件 |
|--------|------|----------|
| 1 | iFlytek 科大讯飞 | 默认使用 |
| 2 | FunASR 公共 WebSocket | iFlytek 返回 11200（额度超限）或连接失败 |
| 3 | Google SpeechRecognition | FunASR 也不可用时最终兜底 |

### 2. 状态变量独立（极易出错！）

项目中有**两组完全独立**的语音识别状态，**绝对不能混用**：

| 用途 | 状态变量 | Session 变量 | 清理函数 |
|------|----------|-------------|----------|
| 麦克风按钮（语音选台） | `_isRecognizing` | `funasrSession` | `stopFunasrRecognition()` |
| 录入按钮（保存到私人语音库） | `_saveRecognizing` | `_saveSession` | `stopSaveRecognition()` |

**修改语音识别代码时，必须先确认自己在改哪一组状态。**

### 3. 文件同步规则

**每次修改 `index.html` 后，必须同步到 `ama-channel.html`**：

```bash
cp index.html ama-channel.html
git add -A
git commit -m "修改说明"
git push
```

### 4. 移动端适配

- 搜索栏使用 `flex-wrap: wrap` 自动换行
- 按钮最小触控区域 **44px**
- 在 Chrome 开发者工具的 Device Mode 中测试多种手机屏幕尺寸

---

## 五、常见问题速查

### Q1：推送代码后网页没有更新？
GitHub Pages 有缓存，推送后等待 1-2 分钟，然后用 **强制刷新**（Mac: `Cmd+Shift+R`，Windows: `Ctrl+F5`）。

### Q2：语音识别突然不能用了？
- 打开浏览器控制台（F12 → Console）查看错误代码
- 如果是 `11200`：iFlytek 今日额度已用完，明天自动恢复，或更换自己的 API 凭据
- 检查麦克风权限是否被浏览器拒绝

### Q3：FunASR 识别很慢/没有结果？
FunASR 公共免费服务算力有限，已经设置为备用方案。如果主服务正常，不会走到 FunASR。

### Q4：私人语音库数据会丢失吗？
数据保存在浏览器的 `localStorage` 中，只要不清理浏览器缓存就不会丢失。换电脑或换浏览器需要重新录入。

### Q5：想添加新节目怎么办？
在 `index.html` 中搜索 `operaDB` 数组，按照已有格式添加新节目对象即可。记得同时准备一张海报图片放到 `opera_images/` 文件夹。

---

## 六、后续开发建议

### 短期可做的优化

1. **接入更多国内语音服务**：阿里云语音识别、百炼大模型等，作为 iFlytek 的同级备选
2. **优化模糊匹配**：当前最低分数门槛为 15 分，可调整阈值或增加更多别名映射
3. **增加节目内容**：越剧、黄梅戏、豫剧、粤剧等全国戏曲
4. **电视大屏模式**：增加针对电视/平板的大字体、大按钮 UI 模式

### 中期可探索的方向

1. **部署 Whisper 模型**：开源多语言语音模型，支持闽南语识别（但不支持莆仙话）
2. **关注阿里 Qwen3-ASR**：新兴方言识别模型，准确率可能超越 Whisper
3. **接入授权内容库**：正式版本需要接入合法的戏曲、影视音源

### 长期目标

**莆仙话专属语音模型**：莆仙话受众约 300 万，各大平台暂无稳定识别方案。可通过采集莆仙话音频数据，微调 Whisper 等模型实现。需与地方文化机构、高校合作获取训练语料。

---

## 七、开发环境准备

1. 安装 [Git](https://git-scm.com/downloads)
2. 注册 [GitHub](https://github.com) 账号
3. 安装 [TRAE](https://www.trae.ai/)（或任意代码编辑器）
4. 克隆仓库到本地：
   ```bash
   git clone https://github.com/sagaxm-ops/ama-channel.git
   cd ama-channel
   ```
5. 用浏览器打开 `index.html` 即可本地预览
6. 修改代码 → 测试 → 提交 → 推送 → 强制刷新在线页面验证

---

## 八、重要联系人/资源

- 科大讯飞开发者平台：https://www.xfyun.cn/
- GitHub Pages 文档：https://docs.github.com/zh/pages
- 项目当前仓库：https://github.com/sagaxm-ops/ama-channel

---

> 祝开发顺利！有任何问题可以随时查看 `DEV_NOTES.md` 中的踩坑记录。
