

<div align="center">
  <img src="./LOGO.png" alt="Scholar Labs Search" width="660"/>
</div>

# Google Scholar Searcher

[English](./README.md) | [中文](./README_zh_CN.md)

---
通过 BrowserOS MCP 搜索 Google Scholar 的 Codex/Claude Code 技能。提取论文元数据与 Easy Scholar 徽章等级，按 CCF/SCI 级别和时间筛选，结果保存为独立的 Markdown 文件。

衍生自 [@atsyplenkov](https://github.com/atsyplenkov) 的 [Scholar Labs Search](https://github.com/atsyplenkov/scholar-labs-search.git)，在原版基础上大幅增强：集成 Easy Scholar 徽章提取、CCF/SCI 筛选、时间过滤、分页支持、批量 Markdown 导出。



## 相比原版的重大改进

| 功能 | 原版 | 本 Fork |
|------|------|--------|
| 徽章提取 | 无 | Easy Scholar Chrome 扩展（level 1–5） |
| CCF/SCI 筛选 | 无 | Level 1/2 徽章 + 时间筛选 |
| 时间过滤 | 无 | Since 2022（4年窗口） |
| 翻页 | 仅第一页 | 最多6页（60篇论文） |
| URL 提取 | Scholar 重定向链接 | 直接出版社链接（IEEE Xplore、ScienceDirect 等） |
| 输出格式 | 单一组合输出 | 每篇论文单独 `.md` 文件 |
| 单次提取 | 否 | 是 — 每页一次 `evaluate_script` |
| IEEE 全文抓取 | 无 | `/ieee-paper-fetch` 集成 |

---

## 环境要求

- **BrowserOS** 本地运行，已启用 MCP
- **Easy Scholar** Chrome 扩展（徽章等级提取必需）
- **Codex** 或 **Claude Code** 支持本地 skills
- 网络可访问 `https://scholar.google.com`

### BrowserOS MCP 配置

```toml
[mcp_servers.browseros]
url = "http://127.0.0.1:9001/mcp"
```

在 BrowserOS 中启用：`Settings` → `BrowserOS as MCP`。详见 [BrowserOS 文档](https://docs.browseros.com)。

---

## Easy Scholar Chrome 扩展

**本技能需要安装 [Easy Scholar](https://chrome.google.com/webstore/detail/easy-scholar/alhpnkhbmobokehfejbhjnbkjpjklepp) Chrome 扩展。**

Easy Scholar 在每个 Google Scholar 结果卡片上叠加中国学术分区徽章：

| 徽章 | 等级 | 说明 |
|------|------|------|
| CCF A | 1 | 中国计算机学会 A 类 |
| SCI Q1 | 1 | Web of Science Q1 期刊 |
| CCF B | 2 | 中国计算机学会 B 类 |
| SCI Q2 | 2 | Web of Science Q2 期刊 |
| EI检索 | 2 | 工程索引 |
| 计算机科学TOP | 1 | 中科院计算机科学 TOP 期刊 |
| SCI升级版 计算机科学1区 | 1 | 中科院升级版 1 区 |

**徽章数据不在 accessibility tree 中显示** — 必须通过页 DOM 上的 `evaluate_script` 提取。技能自动处理此过程。

### 徽章等级参考

```
Level 1: CCF A, SCI Q1, 计算机科学TOP, SCI基础版 工程技术1区, XJU 一区, SWJTU A++
Level 2: CCF B, SCI Q2, EI检索, SCI升级版 计算机科学2区, XJU 二区
Level 3: CCF C, SCI Q3
Level 4: SCI Q4
Level 5: 简介, 期刊全名（无筛选价值）
```

徽章来源从 `title` 属性提取：`数据来源为：(.+?)(]|$)`

---

## 安装

```text
~/.codex/skills/scholar-labs-search/
```

或 Claude Code：

```text
~/.claude/skills/scholar-labs-search-master/
```

**必需文件：**
```
scholar-labs-search/
├── SKILL.md              # 主 skill 文件
└── references/
    ├── workflow.md       # BrowserOS 执行流程
    └── ranking.md       # 论文排名规则
```

---

## 使用方式

### 基本搜索

```
/scholar-labs-search
```

出现提示后输入主题。

### 自定义搜索

```
搜索关于 "graph attention network scheduling edge computing" 的论文
```

### 工作流程示例

1. 激活 skill → 输入主题 → 指定论文数量（默认10篇，最多60篇）
2. 自动应用 "Since 2022" 时间筛选
3. 每页通过单次 `evaluate_script` 提取元数据 + 徽章
4. 按 Level 1/2 徽章 + year ≥ 2022 过滤
5. 按徽章优先级 + 引用数排序
6. 每篇论文保存为 `{徽章标签}-{年份}-{标题}.md`
7. 提示：询问是否抓取 IEEE 论文全文

---

## 输出格式

### Markdown 文件命名

```
{徽章标签}-{年份}-{slugified-title}.md
```

**徽章标签优先级：** CCF A > SCI Q1 > CCF B > SCI Q2 > CCF C > SCI Q3 > SCI Q4

无 CCF/SCI 徽章的论文，文件名以年份开头。

**命名示例：**
```
CCF-A-2022-weighted-feature-fusion-cnn-gat-hyperspectral.md
SCI-Q1-2024-graph-attention-networks-comprehensive-review.md
2023-novel-task-offloading-method.md
```

### Markdown 内容格式

```markdown
# 论文标题

- **Authors:** Author1, Author2, Author3
- **Year:** 2024
- **Venue:** IEEE Transactions on Image Processing
- **Citations:** 594
- **Link:** [Title](url) | [PDF](pdf_url) | [IEEE Xplore](ieee_url)
- **Badges:** `CCF A` `SCI Q1` `IF 13.7`
- **Reason:** 一句话选择理由

> 摘要/片段（来自 Scholar 的 1-3 句摘要）。
```

---

## 筛选规则

### 论文选择（Regular Scholar 模式）

| 条件 | 规则 |
|------|------|
| 年份 ≥ 2022 | 有 ≥1 个 Level 1 或 Level 2 徽章则保留 |
| 年份 < 2022 | 仅在 Level 1 徽章 **且** 引用数 ≥ 2000 时保留 |
| 无徽章数据 | 排除 |

### 文件名徽章优先级

1. **CCF A** — 最高优先级
2. **SCI Q1**
3. **CCF B**
4. **SCI Q2**
5. **CCF C**
6. **SCI Q3**
7. **SCI Q4**
8. **其他** — 不用于文件名

同等级时 CCF 优先于 SCI（例如同时有 CCF A 和 SCI Q1 → 文件名以 `CCF-A` 开头）。

---

## 技术细节

### 单次 DOM 提取

每页一次 `evaluate_script` 调用提取**全部**元数据：

```javascript
document.querySelectorAll('.gs_r').forEach(card => {
  // title + 直接 URL（从 <a href> 获取）
  const titleEl = card.querySelector('h3') || card.querySelector('.gs_rt');
  const url = titleEl?.querySelector('a')?.href;

  // authors + year + venue（从 .gs_a 文本解析）
  const authorText = card.querySelector('.gs_a')?.innerText;
  // 按 '-' 分割 → 作者 | 年份 | 期刊

  // 引用数 from a[href*="cites"]
  // 摘要 from .gs_rs or .gs_sm
  // 徽章 from .easyscholar-ranking
});
```

**URL 提取：** `card.querySelector('h3')?.querySelector('a')?.href` 返回**直接出版社链接**（IEEE Xplore、ScienceDirect、MDPI 等），而非 Scholar 重定向链接。这在大多数情况下消除对 Exa 链接验证的需求。

### 翻页

- 每次页面重载后元素 ID 会变化 — 点击前必须 `take_snapshot`
- "Since 2022" 时间筛选在翻页后保持有效 — 无需重新点击
- 每次搜索最多 6 页（60 篇论文）

### 中断处理

技能在检测到 Scholar 拦截（CAPTCHA、登录）时会暂停，等待用户在 BrowserOS 中解决后恢复。

---

## 文件结构

```
scholar-labs-search/
├── LOGO.png                        # 技能图标
├── SKILL.md                       # 主 skill 入口
├── README.md                      # 英文文档
├── README_zh_CN.md                # 中文文档
├── LICENSE                       # MIT 协议（来自原仓库）
└── references/
    ├── workflow.md              # BrowserOS 执行流程
    └── ranking.md               # 论文排名规则
```

---

## 原版项目

本技能衍生自 [@atsyplenkov](https://github.com/atsyplenkov) 的 [Scholar Labs Search](https://github.com/atsyplenkov/scholar-labs-search.git)。

原版设计面向 Scholar Labs（含注释/关键点）。本 Fork 改用 **Regular Google Scholar**（更稳定、更不易被拦截），并集成 **Easy Scholar 徽章**用于论文质量筛选。

---

## 协议

MIT — 见 [LICENSE](./LICENSE)。原版作者 [@atsyplenkov](https://github.com/atsyplenkov)。
