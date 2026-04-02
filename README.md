# Business Trip Assistant — 出差全流程管理助手

一个 OpenClaw Skill，覆盖出差从申请到报告的完整流程，支持日程生成、会议纪要、出差报告自动生成，可精准填充 Word 模版并同步到企微文档。

## 功能

| 功能 | 说明 |
|------|------|
| 🗓️ 出差日程生成 | 用自然语言描述，自动生成带时间表的 Word 日程文件 |
| 📝 会议纪要 | 支持文字/录音/docx 输入，自动生成结构化纪要 |
| 📊 出差报告 | 汇总所有会议纪要，与申请日程对比分析，输出完整报告 |
| 📄 Word 模版填充 | 精准套用用户自定义 `.docx` 模版（OOXML 级别填充） |
| 🧠 偏好学习 | 从每次内容修改中提炼规则，下次生成自动应用 |
| 👤 多用户档案 | 支持多人档案管理，出差人信息自动填充 |
| 📤 企微文档同步 | 生成后自动同步到企业微信个人空间（可选） |

## 安装后的目录结构

```
business-trip-assistant/
├── SKILL.md              ← 主 skill（AI 读取此文件）
├── README.md             ← 本说明
├── sub-skills/
│   ├── setup/            ← 环境初始化
│   ├── user-profile/     ← 用户档案管理
│   ├── trip-init/        ← 创建出差档案
│   ├── schedule-gen/     ← 日程生成
│   ├── meeting-notes/    ← 会议纪要
│   ├── report-gen/       ← 出差报告
│   ├── template-mgr/     ← 模版管理
│   └── content-refine/   ← 内容微调与偏好学习
├── data/                 ← 【运行时自动创建】出差档案存储
├── templates/            ← 【用户自行上传】私人 Word 模版
├── profiles/             ← 【运行时自动创建】用户档案
└── preferences/          ← 【运行时自动创建】个人偏好规则
```

> ⚠️ `data/`、`templates/`、`profiles/`、`preferences/` 目录**不随 skill 分发**，均在本地运行时自动创建或由用户手动上传，确保隐私安全。

## 依赖

### 必须

- **Python 3** + `defusedxml`（`pip3 install defusedxml`）
- **docx skill**（OOXML 模版精准填充）— 通过 knot/skillhub 安装

### 可选

| 依赖 | 用途 | 安装 |
|------|------|------|
| `openai-whisper` | 录音文件自动转录 | `pip3 install openai-whisper` |
| `ffmpeg` | 企微 AMR 语音格式转换 | `brew install ffmpeg` |
| `docx`（npm） | 无模版时自由生成 Word | `npm install -g docx` |
| wecom-doc skill | 同步到企微文档 | 通过 skillhub 安装 |

## 快速开始

首次使用前，向 AI 说：

```
初始化出差助手
```

AI 会自动检查依赖并初始化目录结构，然后提示你录入个人档案。

### 典型使用流程

```
1. "设置我的信息：我叫张三，XX公司"
   → 创建用户档案，后续无需重复填写

2. "新建出差，下周去北京"
   → 创建出差档案 TRIP-北京-20260325

3. "这次出差要拜访A客户、参加B会议，共3天"
   → 自动生成出差日程 + Word 文件

4. （出差中）"开完会了：讨论了XX，结论是YY，Action Item 是ZZ"
   → 生成会议纪要

5. "出差结束了，生成出差报告"
   → 汇总所有会议纪要，与申请日程对比，输出完整报告

6. （可选）"帮我把第三点改短一点"
   → 微调内容，学习偏好规则
```

## Word 模版支持

如果你有公司标准的会议纪要或出差报告模版（`.docx` 文件），可以上传给 AI：

```
"上传会议纪要模版"（发送 .docx 文件）
```

AI 会解析模版结构，后续生成纪要时精准套用你的格式，保留所有样式和字体。

**注意：** 模版文件属于私人文件，存储在本地 `templates/` 目录，不会被 skill 分发。

## 子 Skill 说明

每个子 skill 都可以独立触发，也由主 skill 自动路由：

- **setup** — 环境检查与初始化
- **user-profile** — 管理出差人档案
- **trip-init** — 创建新出差档案（生成 Trip ID）
- **schedule-gen** — 生成出差日程
- **meeting-notes** — 生成会议纪要（支持文字/录音/docx）
- **report-gen** — 生成出差总结报告（含申请vs实际对比）
- **template-mgr** — 查看/上传/管理 Word 模版
- **content-refine** — 微调内容 + 学习偏好规则

## 版本

v1.0.0
