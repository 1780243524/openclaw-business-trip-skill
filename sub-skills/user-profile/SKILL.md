---
name: user-profile
description: 用户档案管理。当用户说"设置我的信息"、"录入个人信息"、"我叫XX"、"更新我的档案"、"查看我的档案"、"切换用户"时触发。支持新建/更新/查看/切换用户档案，档案信息会在创建出差档案、生成会议纪要和出差报告时自动使用，无需重复询问。
---

# User Profile — 用户档案管理

## 数据存储

所有用户档案存储于 skill 根目录下的 `profiles/` 目录。

**执行前先确定 SKILL_DIR：**
```bash
SKILL_DIR=$(dirname "$(find ~/.openclaw/workspace/skills /projects/.openclaw/skills -name 'SKILL.md' -path '*/business-trip-assistant/SKILL.md' 2>/dev/null | head -1)")
```

每个用户一个 JSON 文件：`$SKILL_DIR/profiles/{用户ID}.json`

**当前激活用户**记录在：
`$SKILL_DIR/profiles/active.json`

```json
{ "activeUserId": "zhangsan" }
```

---

## 用户档案结构

```json
{
  "userId": "zhangsan",
  "name": "李润玖",
  "company": "中金公司",
  "department": "股票业务部",
  "title": "投资经理",
  "email": "zhangsan@example.com",
  "phone": "",
  "team": ["同事A", "同事B"],
  "defaultSignature": "李润玖（中金公司股票业务部）",
  "notes": "其他备注信息",
  "createdAt": "ISO8601时间戳",
  "updatedAt": "ISO8601时间戳"
}
```

**字段说明：**
| 字段 | 用途 | 必填 |
|------|------|------|
| `name` | 出差人姓名、纪要"作者"字段 | ✅ |
| `company` | 所属公司 | 可选 |
| `department` | 部门，用于"中金参会人员"等字段 | 可选 |
| `title` | 职位 | 可选 |
| `defaultSignature` | 纪要/报告中"参会人员"的完整签名格式 | 可选 |
| `team` | 常用随行同事，供纪要参会人员字段参考 | 可选 |
| `notes` | 其他说明 | 可选 |

---

## 功能

### 新建 / 更新档案

触发词：「设置我的信息」「我叫XX」「录入个人信息」「更新档案」

1. 从用户描述中提取个人信息（自然语言解析，不足时询问 `name` 和 `company`）
2. 生成 `userId`：取姓名拼音小写（如"李润玖" → `zhangsan`）；若冲突则追加数字
3. 写入 `profiles/{userId}.json`
4. 写入 `profiles/active.json`，设为当前激活用户
5. 回复确认：
   > ✅ 档案已保存：**李润玖**（中金公司股票业务部）
   > 后续创建出差档案时将自动使用此信息，无需重复填写。

### 查看当前档案

触发词：「查看我的档案」「我的信息」「当前用户是谁」

读取 `profiles/active.json` → 读取对应用户档案 → 格式化展示。

### 切换用户

触发词：「切换到XX」「换成XX来操作」「帮XX创建」

1. 在 `profiles/` 目录中搜索匹配的用户档案（按 `name` 字段匹配）
2. 找到则更新 `profiles/active.json`
3. 未找到则提示用户先创建档案

### 列出所有档案

触发词：「有哪些用户」「所有档案」

列出 `profiles/` 目录下所有 `.json` 文件（排除 `active.json`），展示每人的姓名和公司。

---

## 与其他 skill 的集成规则

### trip-init 创建出差档案时

按以下优先级确定出差人：

1. **用户本次对话中明确指定了出差人** → 直接使用，并检查是否与已有档案匹配
2. **`$SKILL_DIR/profiles/active.json` 存在且有激活用户** → 自动使用激活用户，无需询问
   - 创建时提示：`出差人自动设为：{name}（来自用户档案），如需更换请说"切换到XX"`
3. **无激活用户** → 询问出差人姓名，询问后建议用户创建档案以免下次重复

### meeting-notes / report-gen 填充纪要和报告时

从以下来源获取出差人信息（按优先级）：
1. 当前出差档案 `$SKILL_DIR/data/{trip-id}/meta.json` 中的 `traveler` 字段
2. `$SKILL_DIR/profiles/{traveler对应的userId}.json` 中的完整信息（用于填充 `defaultSignature`、`department` 等详细字段）
3. 仅有姓名时，直接用姓名填充

**填充映射：**
| 模版字段 | 来源 |
|---------|------|
| 作者 | `name` |
| 中金参会人员（我方） | `defaultSignature` 或 `{name}（{company}{department}）` |
| 出差人员（报告） | `name` |

---

## 示例对话

```
用户：我叫李润玖，中金公司股票业务部
→ ✅ 档案已保存：李润玖（中金公司股票业务部）
   后续出差档案将自动使用此信息。

用户：帮我新建一个去北京的出差档案
→ 出差人自动设为：李润玖（来自用户档案）
  ✅ 出差档案已创建：TRIP-北京-20260325

用户：这次不是我去，帮同事张伟创建
→ 切换出差人为：张伟
  ✅ 出差档案已创建：TRIP-北京-20260325，出差人：张伟
```
