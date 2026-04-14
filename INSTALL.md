# he-care（婴儿管家）安装手册

## 简介

**he-care** 是一款婴儿管家技能（Skill），通过 IMA（腾讯智能笔记）笔记和知识库系统，管理宝宝的全生命周期信息。支持 7 个场景：

| 场景 | 说明 |
|------|------|
| 🍼 奶量记录 | 每日喂奶情况追踪 |
| 💉 疫苗接种 | 接种历史和待接种计划 |
| 😴 睡眠记录 | 睡眠规律分析 |
| 🏥 就医记录 | 门诊/住院及用药记录 |
| 📦 婴儿用品 | 用品采购清单 |
| 📋 户口档案 | 宝宝基本信息和证件 |
| 📸 成长照片 | 身高体重和里程碑打卡 |

---

## 前置条件

| 条件 | 说明 |
|------|------|
| **QClaw** | 已安装并配置 QClaw 运行环境 |
| **IMA 授权** | 在 QClaw 控制台 → **集成（Integrations）** 中完成 IMA 授权连接 |
| **Python 3**（可选） | 如需使用 SkillHub CLI 管理 skills |

> ⚠️ **未完成 IMA 授权将无法使用本技能。**

---

## 安装方式

### 方式一：通过 SkillHub CLI 安装（推荐）

如果您的 SkillHub 索引中已收录本技能：

```bash
skillhub install he-care
```

### 方式二：手动安装（下载 ZIP）

1. **下载 `he-care.zip`**（从 GitHub Releases 或 SkillHub）

2. **解压到 QClaw skills 目录**：
   ```
   # Windows (PowerShell)
   Expand-Archive -Path he-care.zip -DestinationPath "$env:USERPROFILE\.qclaw\workspace\skills\he-care" -Force
   ```
   
   或手动解压至：
   ```
   ~/.qclaw/workspace/skills/he-care/
   ```

3. **确认目录结构**：
   ```
   he-care/
   ├── SKILL.md          # 技能主文件（AI agent 执行参考）
   ├── config.json       # 技能配置
   └── references/       # 各场景参考文件
       ├── init.md
       ├── knowledge-base.md
       ├── medical.md
       ├── milk.md
       ├── photos.md
       ├── profile.md
       ├── sleep.md
       ├── supplies.md
       └── vaccine.md
   ```

### 方式三：从 GitHub 仓库安装

```bash
cd ~/.qclaw/workspace/skills/
git clone https://github.com/hxrealm/he-care.git
mv he-care he-care  # 如需重命名
```

---

## 初始化

安装完成后，在 QClaw 对话中触发初始化：

> **"初始化婴儿管家"** 或 **"帮我建宝宝管理知识库"**

AI agent 将自动执行以下步骤：

1. ✅ **检查 IMA 可用性**（未授权会引导配置）
2. 📝 **收集宝宝基本信息**（可选，跳过则使用默认值）
3. 🔍 **检测是否已初始化**
4. 📄 **创建 7 篇 IMA 笔记**
5. 📚 **同步到知识库**（如用户已创建）
6. 💾 **更新 MEMORY.md** 记录知识库 ID

---

## 使用方法

初始化完成后，可通过自然对话使用：

| 你说 | 功能 |
|------|------|
| "记录今天奶量 150ml" | 追加奶量记录 |
| "宝宝今天睡了多久" | 查询睡眠记录 |
| "查上次打的疫苗" | 查询疫苗记录 |
| "添加疫苗：乙肝第二针" | 追加疫苗记录 |
| "修改用品记录" | 更新用品清单 |
| "导入笔记到知识库" | 同步笔记到知识库 |

---

## 配置

`config.json` 内容：

```json
{
  "slug": "he-care",
  "name": "婴儿管家",
  "version": "1.0.0",
  "description": "通过 IMA 笔记和知识库管理宝宝的全生命周期信息",
  "author": "hxrealm",
  "repository": "https://github.com/hxrealm/he-care.git"
}
```

---

## 常见问题

### Q: IMA 授权失效怎么办？
A: 打开 QClaw 控制台 → **集成 → IMA**，点击重新授权。完成后重新使用技能即可。

### Q: 知识库搜不到笔记？
A: 确认知识库名称完全匹配。默认名称为「明宝成长手册」，可在初始化时自定义。

### Q: 如何修改宝宝信息？
A: 说"更新宝宝信息"或"修改户口档案"，AI agent 会引导您更新。

### Q: 支持多个宝宝吗？
A: 当前版本支持单宝宝。如需多宝宝支持，请联系作者。

---

## 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.1.0 | 2026-04-14 | 更新宝宝信息为默认值（王小明/明宝） |
| 1.0.0 | 2026-04-14 | 首次发布 |

---

## 许可证 & 联系

- **作者**: hxrealm
- **GitHub**: https://github.com/hxrealm/he-care
- **问题反馈**: 请在 GitHub 提交 Issues
