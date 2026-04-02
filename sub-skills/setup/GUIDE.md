---
name: setup
description: 出差全流程助手环境初始化。在首次使用 business-trip-assistant 时自动触发，或当用户说"初始化出差助手"、"检查出差助手环境"时触发。负责检查并安装所有依赖，创建必要目录结构，确保后续功能正常运行。
---

# Setup — 环境自检与初始化

## 何时执行

以下任一情况触发本 skill：
1. 用户明确说「初始化出差助手」、「检查环境」、「setup」
2. **首次使用检测**：trip-init / meeting-notes / report-gen 等子 skill 在执行前，先检查 `profiles/.setup_done` 文件是否存在；若不存在，先执行本 skill 再继续原任务

## 执行步骤

### Step 0：确定 SKILL_DIR

**每次执行前必须先确定 skill 根目录：**

```bash
SKILL_DIR=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills -name 'SKILL.md' -path '*/business-trip-assistant/SKILL.md' 2>/dev/null | head -1)")
echo "SKILL_DIR=$SKILL_DIR"
```

后续所有路径均基于 `$SKILL_DIR`。

### Step 1：检查并创建目录结构

```bash
mkdir -p "$SKILL_DIR/data"
mkdir -p "$SKILL_DIR/templates"
mkdir -p "$SKILL_DIR/profiles"
```

### Step 2：检查 Python 依赖

```bash
# defusedxml（OOXML 解析必须）
python3 -c "import defusedxml" 2>/dev/null || pip3 install defusedxml -q

# 标准库（通常内置，检查即可）
python3 -c "import zipfile, xml.sax.saxutils, re, shutil" 2>/dev/null \
  || echo "⚠️ Python 标准库异常，请检查 Python 安装"
```

### Step 3：检查 OOXML 脚本（来自 docx skill）

```bash
DOCX_SKILL=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills /usr/local/lib -name 'SKILL.md' -path '*/docx/SKILL.md' 2>/dev/null | head -1)")
UNPACK="$DOCX_SKILL/ooxml/scripts/unpack.py"
PACK="$DOCX_SKILL/ooxml/scripts/pack.py"

if [ ! -f "$UNPACK" ] || [ ! -f "$PACK" ]; then
  echo "⚠️ 缺少 docx skill（OOXML 脚本）"
  echo "请通过 skillhub 安装 docx skill："
  echo "  在 OpenClaw 对话中执行：skillhub install docx"
  echo "  安装后重新运行 setup"
  # 标记为不完整，后续模版填充功能不可用（但 Markdown 格式仍可用）
fi
```

> **说明**：docx skill 提供 `ooxml/scripts/unpack.py` 和 `pack.py`，是模版精准填充的核心依赖。
> 若不安装，会议纪要/出差报告仍可生成（降级为通用格式），但无法精准套用自定义 `.docx` 模版。

### Step 4：检查 Node.js docx 模块（无模版时自由生成用）

```bash
# 检查全局安装
node -e "require('docx')" 2>/dev/null \
  || NODE_PATH=$(npm root -g) node -e "require('docx')" 2>/dev/null \
  || npm install -g docx -q \
  && echo "docx npm 模块：OK"
```

### Step 5：检查 Whisper（音频转录用）

```bash
if command -v whisper &>/dev/null; then
  echo "✅ whisper：$(whisper --version 2>&1 | head -1)"
  # 检查 large-v3 模型是否已缓存（首次使用需下载约 3GB）
  if ls ~/.cache/whisper/ 2>/dev/null | grep -q "large"; then
    echo "✅ whisper large-v3 模型：已缓存（可直接使用）"
  else
    echo "ℹ️ whisper large-v3 模型：未缓存（首次转录时需下载约 3GB，请确保网络通畅）"
  fi
else
  echo "ℹ️ openai-whisper 未安装（音频文件自动转录功能不可用）"
  echo "  如需从录音直接生成会议纪要，可安装："
  echo "  brew install openai-whisper"
  echo "  或：pip3 install openai-whisper"
fi

# 可选：检查 ffmpeg（AMR 企微语音格式转换依赖）
if command -v ffmpeg &>/dev/null; then
  echo "✅ ffmpeg：可用（支持 AMR 企微语音转码）"
else
  echo "ℹ️ ffmpeg 未安装（企微 AMR 语音需手动转格式；wav/mp3/m4a 可直接转录）"
fi
```

> **说明**：Whisper 为可选依赖，安装后可直接发送录音文件路径，自动转录并生成会议纪要，无需手动转文字。

### Step 6：检查 wecom-doc skill（企微文档同步用）

```bash
WECOM_DOC=$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills /usr/local/lib -name "SKILL.md" -path "*/wecom-doc/*" 2>/dev/null | head -1)
if [ -z "$WECOM_DOC" ]; then
  echo "ℹ️ wecom-doc skill 未安装（企微文档同步功能不可用）"
  echo "  如需同步到企微文档，请在 OpenClaw 对话中执行：skillhub install wecom-doc"
fi
```

> **说明**：wecom-doc skill 为可选依赖，仅影响「同步到企微文档」功能，不影响本地生成和发送文件。

### Step 7：写入初始化完成标记

```bash
cat > "$SKILL_DIR/profiles/.setup_done" << EOF
{
  "setupAt": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "python": "$(python3 --version 2>&1)",
  "node": "$(node --version 2>&1)",
  "defusedxml": "$(python3 -c 'import defusedxml; print(defusedxml.__version__)' 2>/dev/null || echo 'unknown')",
  "docxSkill": $([ -f "$UNPACK" ] && echo 'true' || echo 'false'),
  "whisper": $(command -v whisper &>/dev/null && echo 'true' || echo 'false'),
  "ffmpeg": $(command -v ffmpeg &>/dev/null && echo 'true' || echo 'false'),
  "wecomDocSkill": $([ -n "$WECOM_DOC" ] && echo 'true' || echo 'false')
}
EOF
echo "✅ 初始化完成"
```

---

## 输出报告格式

执行完毕后，向用户展示一份简洁的检查报告：

```
✅ 出差全流程助手 — 环境检查完成

核心功能：
  ✅ Python 环境        正常
  ✅ defusedxml         已安装
  ✅ docx skill         已安装（模版精准填充：可用）
  ✅ Node.js docx 模块  已安装（自由格式生成：可用）

音频转录：
  ✅ openai-whisper     已安装（录音自动转录：可用）
  ✅ ffmpeg             已安装（企微 AMR 语音转码：可用）

可选功能：
  ℹ️ wecom-doc skill    未安装（企微文档同步：不可用）

目录结构：
  ✅ data/       出差档案存储
  ✅ templates/  模版目录
  ✅ profiles/   用户档案

一切就绪，可以开始使用！
发送「设置我的信息」先录入个人档案，或直接说「新建出差」开始。
```

若有依赖缺失，在对应行标记 ❌ 并给出安装指引。

---

## 首次使用自动触发逻辑

各子 skill（trip-init / meeting-notes / report-gen / schedule-gen）在**第一步**加入以下检查：

```python
import os, glob

# 动态找到 SKILL_DIR
matches = glob.glob(os.path.expanduser('~/.openclaw/workspace/skills/*/business-trip-assistant/SKILL.md')) + \
          glob.glob('/projects/.openclaw/skills/business-trip-assistant/SKILL.md')
SKILL_DIR = os.path.dirname(matches[0]) if matches else None

setup_flag = os.path.join(SKILL_DIR, 'profiles/.setup_done') if SKILL_DIR else None
if not setup_flag or not os.path.exists(setup_flag):
    print("⚙️ 首次使用，正在初始化环境...")
    # 执行 setup skill 流程
```

若 `.setup_done` 存在，直接跳过，不影响正常使用速度。
