---
name: he-care
description: |
  婴儿管家技能，通过 IMA 笔记和知识库管理宝宝的全生命周期信息。
  触发场景：
  - 初始化婴儿管家（"初始化婴儿管家"、"帮我建宝宝管理知识库"）
  - 记录宝宝信息（"记录今天的奶量"、"添加疫苗记录"、"记一下今天睡了多久"）
  - 查询宝宝信息（"查上次打的疫苗"、"最近奶量怎么样"、"宝宝户口信息"）
  - 更新宝宝信息（"修改用品记录"、"更新宝宝体重"）
  - 知识库操作（"导入笔记到知识库"、"把笔记同步到知识库"、"创建知识库"）
  支持 7 个场景：每日奶量、疫苗接种、睡眠记录、就医记录、婴儿用品、户口档案、成长照片。
  【前置条件】使用本技能前必须完成 IMA 授权。
---

# 婴儿管家 (He Care)

通过 IMA 笔记系统管理宝宝的全流程信息，笔记可追加到 IMA 知识库形成结构化档案。

## 核心概念：笔记 vs 知识库

- **IMA 笔记**（notes）：宝宝数据的**主存储**，每次记录直接写入对应笔记
- **IMA 知识库**（knowledge-base）：笔记的**引用集合**，方便统一浏览和搜索
  - 笔记写入知识库后，知识库页面可以直接查看所有宝宝记录
  - 知识库仅存储笔记引用，删除知识库中的引用不影响笔记本身
  - ⚠️ IMA API 不支持"移动"笔记，只能将笔记**添加**到知识库

## 前置条件检查

> ⚠️ **使用本技能前必须先完成 IMA 授权。**

### 检查脚本（每次会话首次触发时执行）

```powershell
$gtp = "C:\Program Files\QClaw\resources\openclaw\config\skills\ima\get-token.ps1"
if (-not (Test-Path $gtp)) { Show-MissingImaGuide; exit }

$creds = & $gtp | ConvertFrom-Json
$body  = @{ search_type = 0; query_info = @{ title = "init-check" }; start = 0; end = 1 } | ConvertTo-Json -Compress
$utf8  = [System.Text.Encoding]::UTF8.GetBytes($body)
$resp  = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/note/v1/search_note_book" `
    -Method Post -Headers @{
        "ima-openapi-clientid" = $creds.client_id
        "ima-openapi-apikey"   = $creds.api_key
    } -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 15

if ($resp.code -eq 20004) { Show-ImaAuthExpiredGuide; exit }
# code=0 → IMA 可用，继续流程
```

### 引导话术

**IMA 未安装：**
> "婴儿管家依赖 IMA（腾讯智能笔记）来存储和管理宝宝信息，当前尚未配置。
> 请在 QClaw 控制台 → **集成（Integrations）** 中找到 IMA，点击授权连接。
> 完成后告诉我「配置好了」，我继续帮您初始化婴儿管家。"

**IMA 授权失效：**
> "IMA 授权已失效，需要重新授权。
> 请打开 QClaw 控制台 → **集成 → IMA**，点击重新授权。
> 完成后告诉我「好了」，我继续。"

---

## 知识库结构

### 笔记清单（7篇）

| 笔记标题 | 场景 | 参考文件 |
|---------|------|---------|
| 🍼 奶量记录 | 每日喂奶情况 | `references/milk.md` |
| 💉 疫苗接种 | 疫苗接种史 | `references/vaccine.md` |
| 😴 睡眠记录 | 每日睡眠情况 | `references/sleep.md` |
| 🏥 就医记录 | 门诊/住院记录 | `references/medical.md` |
| 📦 婴儿用品 | 用品采购清单 | `references/supplies.md` |
| 📋 户口档案 | 宝宝基本信息与证件 | `references/profile.md` |
| 📸 成长照片 | 照片打卡记录 | `references/photos.md` |

### 知识库

- **默认知识库名称**：「宁宝成长手册」
- **知识库 ID**（存储于 MEMORY.md，每次操作前读取）：
  ```
  baby_knowledge_base_id: <YJqmR_MloF-qNRLByVJWcfzW3vw9nfyGIiZGWZczZWk=>
  ```
- 若用户提到其他知识库名称，用 `search_knowledge_base` 按名称查找 ID

---

## 日常操作路由

### 意图 → 参考文件

| 用户说的关键词 | 路由到 |
|-------------|-------|
| 奶、喂奶、奶量、毫升 | `references/milk.md` |
| 疫苗、打针、接种、预防针 | `references/vaccine.md` |
| 睡觉、睡眠、入睡、醒来、午睡 | `references/sleep.md` |
| 看病、就医、医院、发烧、用药、诊断 | `references/medical.md` |
| 用品、奶瓶、纸尿裤、购买、推车 | `references/supplies.md` |
| 宝宝信息、户口、出生、身份证、档案 | `references/profile.md` |
| 照片、相片、成长、打卡 | `references/photos.md` |
| 知识库、导入、同步、移到知识库 | `references/knowledge-base.md` |
| 初始化、创建婴儿管家 | → 初始化流程 |

---

## 同步机制：每次记录后自动同步到知识库

> 核心原则：记录写入笔记后，立即尝试同步到知识库；知识库不存在或同步失败时静默跳过，不影响记录本身。

### 同步触发点

每次执行以下操作后，自动触发知识库同步：

| 操作 | 说明 |
|------|------|
| 新建笔记（初始化） | 创建完 7 篇笔记后，批量添加到知识库 |
| 追加记录（append_doc） | 记录追加成功后，将该笔记添加到知识库 |
| 用户明确要求同步 | 用户说"同步到知识库"时执行 |

### 同步前置条件

同步前从 MEMORY.md 读取知识库 ID：
- **有知识库 ID** → 执行 `add_knowledge`（笔记 → 知识库）
- **没有知识库 ID** → 检查用户是否提到知识库名称，有则查找 ID 后同步，无则静默跳过

### 同步脚本

```powershell
# 从 MEMORY.md 读取知识库 ID
$memPath = "C:\Users\admin\.qclaw\workspace\MEMORY.md"
$memContent = Get-Content $memPath -Raw
if ($memContent -match 'baby_knowledge_base_id:\s*(.+)') {
    $kbId = $matches[1].Trim()
} else {
    $kbId = $null
}

# 若无 KB ID，尝试按名称搜索
if (-not $kbId) {
    # 用 search_knowledge_base 接口搜索"宁宝成长手册"
    # 找到则更新 KB ID 并继续，找不到则退出
}

# 执行同步
$body = @{
    media_type = 11
    note_info = @{ content_id = $docId }
    title = $noteTitle
    knowledge_base_id = $kbId
} | ConvertTo-Json -Depth 3
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)

$resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/wiki/v1/add_knowledge" `
    -Method Post -Headers $headers `
    -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 20

# 错误处理
# - retcode=0：成功，静默
# - retcode=220001（知识重复添加）：笔记已在知识库中，正常
# - retcode≠0 且非220001：记录到日志，告知用户
if ($resp.retcode -ne 0 -and $resp.retcode -ne 220001) {
    Write-Host "KB_SYNC_FAILED: $($resp.errmsg)"
}
```

### 知识库操作详解

详见 `references/knowledge-base.md`。

---

## 初始化流程

> 仅当 IMA 可用性检查通过后执行。详见 `references/init.md`。

### 快速流程

1. **收集基本信息**（可选，跳过用默认值）
2. **检测是否已初始化**：搜索是否有「户口档案」笔记
3. **创建 7 篇笔记**：依次用 `import_doc` 创建
4. **同步到知识库**：搜索或创建「宁宝成长手册」知识库，批量添加 7 篇笔记
5. **更新 MEMORY.md**：记录知识库 ID

---

## 示例：用户已创建知识库，要求导入笔记

> 用户："我已经创建了一个叫宁宝成长手册的知识库，帮我把最新的内容和笔记导入到这个知识库"

**执行步骤：**

1. IMA 前置检查（同上）
2. 搜索知识库，获取 KB ID：
   ```powershell
   $body = @{ query = "宁宝成长手册"; cursor = ""; limit = 10 } | ConvertTo-Json -Compress
   $utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)
   $resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/wiki/v1/search_knowledge_base" `
       -Method Post -Headers $headers `
       -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 15
   # $resp.data.info_list[0].kb_id → 知识库 ID
   ```
3. 搜索已有笔记，获取 7 篇笔记的 doc_id：
   ```powershell
   # 用 list_note_by_folder_id 列出所有笔记
   $body = @{ folder_id = ""; cursor = ""; limit = 20 } | ConvertTo-Json -Compress
   $resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/note/v1/list_note_by_folder_id" `
       -Method Post -Headers $headers `
       -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 15
   # 筛选出 7 篇婴儿管家笔记，记录其 doc_id
   ```
4. 批量将笔记添加到知识库：
   ```powershell
   foreach ($note in $notes) {
       $body = @{
           media_type = 11
           note_info = @{ content_id = $note.docId }
           title = $note.title
           knowledge_base_id = $kbId
       } | ConvertTo-Json -Depth 3
       $utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)
       Invoke-RestMethod -Uri "https://ima.qq.com/openapi/wiki/v1/add_knowledge" `
           -Method Post -Headers $headers `
           -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 20
   }
   ```
5. 更新 MEMORY.md 中的 `baby_knowledge_base_id`
6. 告知用户完成

---

## 注意事项

- **UTF-8 编码**：所有 notes 写入操作前必须确保 content 为 UTF-8
- **追加分隔**：每次追加在内容前加 `---` 分隔线和时间戳
- **隐私保护**：户口档案含敏感信息，不在群聊中展示正文
- **同步失败不阻断**：知识库同步失败不影响笔记记录本身
- **宝宝信息**（来自 MEMORY.md）：
  - 姓名：易正宁，昵称：宁宝，出生：2024-04-20，男宝，约 23 个月
