---
name: trip-init
description: 创建新的出差ID并初始化出差档案。当用户说"新建出差"、"开始出差"、"我要去XX出差"、"创建出差ID"时触发。
---

# Trip Init — 创建出差档案

## 路径约定

执行前先确定 skill 根目录：
```bash
SKILL_DIR=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills -name 'SKILL.md' -path '*/business-trip-assistant/SKILL.md' 2>/dev/null | head -1)")
```
后续 `data/`、`profiles/`、`templates/` 等路径均基于 `$SKILL_DIR`。

## 步骤

### 1. 收集必要信息

需要从用户处获取（可通过自然语言提取，不足时询问）：
- **出差人姓名**（必须，见下方"确认出差人"规则）
- **目的地城市**（必须）
- **出发日期**（必须，格式 YYYYMMDD）
- **出差目的/背景**（可选，用于后续生成日程）
- **预计天数**（可选）

### 确认出差人（重要规则）

按以下优先级确定出差人，**不要重复询问**：

**优先级 1：用户本次明确指定**
- 如"帮同事张伟创建"、"张三去北京出差" → 直接使用，无需询问

**优先级 2：读取激活用户档案（自动使用，无需询问）**
- 读取 `$SKILL_DIR/profiles/active.json` → 获取 `activeUserId`
- 读取 `$SKILL_DIR/profiles/{activeUserId}.json` → 获取完整用户信息
- 存在激活用户时，**静默使用**，仅在回复中提示一行：
  > 出差人自动设为：{name}（来自用户档案）

**优先级 3：无激活用户时才询问**
- 简洁询问：「请问这次出差是谁出行？」
- 收到回答后继续执行，**不需要二次确认**
- 询问完成后建议：「可以发送"设置我的信息"创建档案，下次无需重复填写」

### 2. 生成 Trip ID

格式：`TRIP-{城市}-{YYYYMMDD}`

检查 `$SKILL_DIR/data/` 目录是否已存在同名目录，若存在则追加 `-2`、`-3` 等序号。

### 3. 创建档案目录

```bash
mkdir -p "$SKILL_DIR/data/{trip-id}/meetings"
```

### 4. 写入 meta.json

写入路径：`$SKILL_DIR/data/{trip-id}/meta.json`

```json
{
  "tripId": "TRIP-北京-20260320",
  "traveler": "张三",
  "travelerSignature": "张三（中金公司股票业务部）",
  "destination": "北京",
  "startDate": "2026-03-20",
  "purpose": "用户描述的出差目的",
  "estimatedDays": 3,
  "status": "planning",
  "plannedSchedule": null,
  "meetings": [],
  "createdAt": "ISO8601时间戳",
  "updatedAt": "ISO8601时间戳"
}
```

- `traveler`：出差人姓名（用于显示）
- `travelerSignature`：出差人完整签名（来自用户档案的 `defaultSignature`，无档案时同 `traveler`）
- 后续生成纪要时，「作者」用 `traveler`，「中金参会人员」用 `travelerSignature`

### 5. 回复用户

告知：
- 已创建的 Trip ID 和出差人
- 下一步建议（描述出差内容以生成日程）

示例回复：
> ✅ 出差档案已创建：**TRIP-北京-20260320**
> 出差人：张三
> 
> 接下来你可以：
> - 描述这次出差的内容，我帮你生成详细日程
> - 或者直接开始记录会议
