---
name: omi
description: Omi3.0 小红书图文笔记生产线。输入内容素材 + IP形象，自动产出分镜脚本JSON + 图片生成指令，适用于 Claude Code、OpenClaw、Codex、扣子等任何具备本地图片生成能力的 Agent 平台。触发词：omi、小红书笔记生成、omi笔记
---

# Omi3.0 小红书笔记生产线 Skill

## 使用方式

```
/omi <content> [--need <需求描述>]
```

示例：
```
/omi 《Claude Code完全指南》 --need 面向小白，突出省时省力
/omi GLM-5测评报告 --need 科技感风格，重点展示速度对比数据
/omi 飞书多维表格实战技巧
```

参数说明：
- `content`：原始素材内容（直接粘贴文本，或描述主题）
- `--need`：额外的风格/受众/侧重点需求（可选）

---

## 流水线说明

4 个步骤，**必须按顺序执行**：

```
[Step 1] 资产生成器 🍌
   输入：IP形象图 + 风格需求
   输出：Asset（品牌视觉资产看板，16:9）
      ↓
[Step 2] 笔记导演（LLM 执行）
   输入：原始素材 + 内容需求
   输出：full_json / cover_json / content_json
      ↓
[Step 3] 封面生成大师
   输入：cover_json + Asset图
   输出：封面图（3:4，4K）
      ↓
[Step 4] 内容生成大师
   输入：content_json + Asset图
   输出：内容页图（3:4，2K，可多张）
```

> **Step 1/3/4** 需要本地图片生成能力（图生图）。
> **Step 2（笔记导演）** 是纯 LLM 任务，由当前 Agent 直接执行。

---

## 环境检查（执行前必须）

收到 `/omi $ARGUMENTS` 后，**首先检查当前环境是否具备图片生成能力**：

- 检查是否有可用的图像生成工具（如 Gemini image API、DALL·E、Stable Diffusion 等）
- 如果**有**图片生成能力 → 直接执行全部 4 个步骤
- 如果**没有**图片生成能力 → 告知用户：

> ⚠️ 当前 Agent 环境不具备图片生成能力，无法执行 Step 1/3/4。
> 请为你的 Agent 接入图片生成工具后重试。
> 目前仍可执行 Step 2（笔记导演），输出分镜脚本 JSON 供你手动提交给其他图片生成工具使用。

---

## 执行流程

收到 `/omi $ARGUMENTS` 后，按以下步骤执行：

### Step 1：准备资产包（首次使用或更换IP时）

> 📄 **Prompt 参考：** `prompts/01-资产生成器.md`

询问用户：
- 是否已有 Asset 资产图？（有则跳过，直接提供图片）
- 如需生成：请提供 IP形象图 + 风格关键词（如：清新、多巴胺、极简、科技感）

使用 `prompts/01-资产生成器.md` 中的 Prompt，调用本地图片生成工具执行：
```
步骤名：「Omi3.0」资产生成器-🍌
参数：尺寸 16:9 | 质量 auto
输入：
  - 风格需求文本（need）：[根据用户描述生成]
  - IP形象图（ip_image）：[用户上传]
输出：asset 图片（保存备用）
```

---

### Step 2：生成分镜脚本（LLM 本地执行）

> 📄 **Prompt 参考：** `prompts/02-笔记导演.md`（含 System Prompt、User Prompt、后处理代码）

扮演"小红书金牌内容架构师"，根据原始素材生成结构化分镜脚本 JSON。

**输出格式严格遵守以下 JSON Schema：**

```json
{
  "meta": {
    "topic": "笔记核心主题",
    "keywords": ["关键词1", "关键词2"]
  },
  "storyboard": [
    {
      "page_index": 1,
      "type": "cover",
      "content_layer": {
        "main_title": "主标题（8字以内，极强冲击力）",
        "sub_title": "副标题（补充说明/结果承诺）",
        "text_layout_intent": "标题居中/标题悬浮/大字报风格"
      },
      "visual_layer": {
        "ip_role": "Hero",
        "ip_instruction": "详细描述IP的高张力动作（如：双手抱头震惊，拿着大喇叭喊话）",
        "scene_elements": "核心装饰元素（如：爆炸的背景线，巨大的问号）"
      }
    },
    {
      "page_index": 2,
      "type": "content",
      "content_layer": {
        "heading": "本页核心论点（简短有力）",
        "body_text_structured": [
          "1. 观点一：详细解释...",
          "2. 观点二：详细解释...",
          "3. 结论：..."
        ],
        "key_highlight": "本页最需要高亮的一句话"
      },
      "visual_layer": {
        "ip_role": "Guide",
        "ip_instruction": "IP位于画面右下角，动作微小（如：推眼镜，手指指向文字列表）",
        "visual_focus": "强调文字排版的清晰度，背景保持干净或微纹理"
      }
    }
  ]
}
```

**内容规则：**
- 封面：IP 动作幅度大，占比 > 40%，标题极具冲击力
- 内页：IP 动作幅度小，占比 < 15%，内容为王
- 内页数量：3-5页，确保原始素材关键信息无遗漏
- **禁止**：违禁词（极/最/第一/首个），外链，梯子/翻墙字样，单独福利领取页

生成完整 JSON 后，自动拆分输出：
- `cover_json`：`storyboard[0]`（封面数据）
- `content_json`：`storyboard[1:]`（内容页数据列表）

---

### Step 3：生成封面图

> 📄 **Prompt 参考：** `prompts/03-封面生成大师.md`

使用 `prompts/03-封面生成大师.md` 中的 Prompt，调用本地图片生成工具执行：
```
步骤名：「Omi3.0」封面生成大师
参数：尺寸 3:4 | 质量 4K
输入：
  - cover_json 文本：[Step 2 输出的 cover_json]
  - Asset图片（Asset）：[Step 1 生成的资产图]
输出：封面图（3:4，4K）
```

---

### Step 4：生成内容页图

> 📄 **Prompt 参考：** `prompts/04-内容生成大师.md`

使用 `prompts/04-内容生成大师.md` 中的 Prompt，调用本地图片生成工具执行：
```
步骤名：「Omi3.0」内容生成大师
参数：尺寸 3:4 | 质量 2K
输入：
  - content_json 文本：[Step 2 输出的 content_json，每次传入一页]
  - Asset图片（Asset）：[Step 1 生成的资产图]
输出：内容页图片（3:4，2K）
```

> 如果内容页超过 1 页，需要将 content_json 中的每个对象**逐个**传入，分别生成对应图片。

---

## 最终产出清单

```
✅ 分镜脚本 JSON（full_json）
✅ 封面图（3:4，4K）
✅ 内容页图片（3:4，2K，共 N 张）
```
