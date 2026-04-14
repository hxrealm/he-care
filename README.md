# he-care（婴儿管家）

[![GitHub License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Skill Version](https://img.shields.io/badge/version-1.1.0-green.svg)](config.json)

通过 IMA（腾讯智能笔记）笔记和知识库系统，管理宝宝的全生命周期信息。

---

## ✨ 功能特性

| 场景 | 说明 |
|------|------|
| 🍼 **奶量记录** | 每日喂奶情况追踪（母乳/配方奶、毫升数） |
| 💉 **疫苗接种** | 接种历史和待接种计划（含国家免疫规划时间表） |
| 😴 **睡眠记录** | 睡眠规律分析（夜觉/午睡、时长统计） |
| 🏥 **就医记录** | 门诊/住院及用药记录 |
| 📦 **婴儿用品** | 用品采购清单（分类管理：喂养/出行/寝具/洗护/衣物/玩具） |
| 📋 **户口档案** | 宝宝基本信息和证件信息 |
| 📸 **成长照片** | 身高体重和里程碑打卡 |

---

## 📦 安装

### 方式一：SkillHub CLI

```bash
skillhub install he-care
```

### 方式二：手动安装

1. 下载 [he-care.zip](https://github.com/hxrealm/he-care/releases)
2. 解压至 `~/.qclaw/workspace/skills/he-care/`

### 方式三：Git 克隆

```bash
git clone https://github.com/hxrealm/he-care.git ~/.qclaw/workspace/skills/he-care
```

详细安装步骤请参阅 [INSTALL.md](INSTALL.md)

---

## 🚀 快速开始

### 1. 前置条件

在 QClaw 控制台 → **集成（Integrations）** 中完成 **IMA 授权** 连接。

### 2. 初始化

在 QClaw 对话中说：

> **"初始化婴儿管家"**

AI agent 将自动：
- 检查 IMA 可用性
- 收集宝宝基本信息（可选）
- 创建 7 篇 IMA 笔记
- 同步到知识库

### 3. 开始记录

| 你说 | 功能 |
|------|------|
| "记录今天奶量 150ml" | 追加奶量记录 |
| "宝宝今天睡了多久" | 查询睡眠记录 |
| "查上次打的疫苗" | 查询疫苗记录 |
| "添加疫苗：乙肝第二针" | 追加疫苗记录 |
| "修改用品记录" | 更新用品清单 |
| "导入笔记到知识库" | 同步笔记到知识库 |

---

## 📁 项目结构

```
he-care/
├── SKILL.md          # 技能主文件（AI agent 执行参考）
├── config.json       # 技能配置（slug、版本等）
├── INSTALL.md        # 安装手册
├── README.md         # 本文件
└── references/       # 各场景参考文件
    ├── init.md            # 初始化步骤
    ├── knowledge-base.md  # 知识库操作指南
    ├── milk.md            # 奶量记录模板
    ├── vaccine.md         # 疫苗接种模板
    ├── sleep.md           # 睡眠记录模板
    ├── medical.md         # 就医记录模板
    ├── supplies.md        # 婴儿用品模板
    ├── profile.md         # 户口档案模板
    └── photos.md          # 成长照片模板
```

---

## ⚙️ 配置

`config.json`：

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

## 📖 详细文档

- [安装手册](INSTALL.md) — 安装步骤、初始化流程、常见问题
- [SKILL.md](SKILL.md) — 技能完整规范（供 AI agent 参考）

---

## 📝 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.1.0 | 2026-04-14 | 更新宝宝信息为默认值（王小明/明宝） |
| 1.0.0 | 2026-04-14 | 首次发布 |

---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

---

## 📄 许可证

MIT License

---

## 👤 作者

- **hxrealm** — [GitHub](https://github.com/hxrealm)
