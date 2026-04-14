# 初始化步骤

> 本文件供 AI agent 执行初始化时参考。完整流程见父目录 `../SKILL.md`「初始化知识库流程」章节。

## ⚠️ 正式开始前：IMA 可用性检测

> **必须执行，不得跳过。** 未通过检测时停止初始化流程，向用户展示引导信息。

### 检测脚本（PowerShell）

```powershell
$gtp = "C:\Program Files\QClaw\resources\openclaw\config\skills\ima\get-token.ps1"

# Step 1: 检查文件是否存在
if (-not (Test-Path $gtp)) {
    Write-Host "IMA_SKILL_NOT_INSTALLED"
    exit
}

# Step 2: 获取凭证
try {
    $creds = & $gtp | ConvertFrom-Json
} catch {
    Write-Host "IMA_TOKEN_FAILED: $($_.Exception.Message)"
    exit
}

# Step 3: 测试 IMA API（search_note_book 最轻量）
$body = @{ search_type = 0; query_info = @{ title = "init-check" }; start = 0; end = 1 } | ConvertTo-Json -Compress
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)
try {
    $resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/note/v1/search_note_book" `
        -Method Post `
        -Headers @{
            "ima-openapi-clientid" = $creds.client_id
            "ima-openapi-apikey"   = $creds.api_key
        } `
        -Body $utf8 `
        -ContentType "application/json; charset=utf-8" `
        -TimeoutSec 15
} catch {
    Write-Host "IMA_API_FAILED: $($_.Exception.Message)"
    exit
}

# Step 4: 判断结果
if ($resp.code -eq 20004) {
    Write-Host "IMA_AUTH_EXPIRED"
} elseif ($resp.code -eq 0 -or $null -ne $resp.code) {
    Write-Host "IMA_OK"
} else {
    Write-Host "IMA_UNKNOWN_ERROR: $($resp | ConvertTo-Json)"
}
```

### 检测结果处理

| 输出 | 含义 | 操作 |
|------|------|------|
| `IMA_SKILL_NOT_INSTALLED` | IMA 技能未安装 | 引导用户在 QClaw 控制台安装 |
| `IMA_TOKEN_FAILED` | 凭证获取失败 | 引导用户重新配置 IMA 授权 |
| `IMA_AUTH_EXPIRED` | IMA 授权失效 | 引导用户重新授权 IMA |
| `IMA_OK` | IMA 可用 | 继续执行初始化 |
| `IMA_UNKNOWN_ERROR` | 其他错误 | 将错误信息告知用户，询问是否继续 |

### 引导话术模板

**IMA 未安装：**
> "初始化婴儿管家需要先配置 IMA（腾讯智能笔记）授权。
> 请在 QClaw 控制台 → **集成（Integrations）** 中找到 IMA，点击授权连接。
> 完成后告诉我「配置好了」，我继续帮您初始化。"

**IMA 授权失效：**
> "IMA 授权已过期，需要重新授权。
> 请打开 QClaw 控制台 → **集成 → IMA**，点击重新授权。
> 完成后告诉我「好了」，我继续初始化婴儿管家。"

---

## 第一步：收集宝宝基本信息（可选）

可询问用户以下信息（跳过时留空，后续可补充）：

- 宝宝姓名（昵称）：默认「王小明（明宝）」
- 出生日期：默认 2024-04-20（约 23 个月）
- 性别：默认男宝

若用户跳过，直接使用默认信息初始化档案。

---

## 第二步：检测是否已初始化

在创建笔记前，先搜索是否已有「户口档案」或「奶量记录」：

```powershell
$body = @{ search_type = 0; query_info = @{ title = "户口档案" }; start = 0; end = 5 } | ConvertTo-Json -Compress
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)
$resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/note/v1/search_note_book" `
    -Method Post `
    -Headers $headers `
    -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 15

if ($resp.data.total_count -gt 0) {
    Write-Host "ALREADY_INITIALIZED"
    # 告知用户婴儿管家已就绪，列出已有笔记
} else {
    Write-Host "NEED_INIT"
    # 继续创建笔记
}
```

---

## 第三步：搜索或创建知识库

初始化前先检查是否已有「明宝成长手册」知识库：

```powershell
# 用名称搜索知识库
$body = @{ query = "明宝成长手册"; cursor = ""; limit = 10 } | ConvertTo-Json -Compress
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)
$resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/wiki/v1/search_knowledge_base" `
    -Method Post -Headers $headers `
    -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 15

if ($resp.data.info_list.Count -gt 0) {
    $kbId = $resp.data.info_list[0].kb_id
    Write-Host "KB_FOUND: $kbId"
} else {
    Write-Host "KB_NOT_FOUND"
    # 用户说已创建但搜不到 → 告知用户知识库名称是否完全匹配
    # 用户未提到 → 初始化完成后提示用户手动创建知识库
}
```

**用户说"我已经创建了知识库"但搜不到时：**
> "我搜索了「明宝成长手册」，没有找到。可能知识库名称有差异，请确认一下知识库的准确名称，告诉我即可继续导入。"

---

## 第四步：依次创建 7 篇笔记

使用 `import_doc` 接口创建笔记。按以下顺序（先创建档案作为主索引）：

### 凭证获取（每次调用前重新获取）

```powershell
$gtp = "C:\Program Files\QClaw\resources\openclaw\config\skills\ima\get-token.ps1"
$creds = & $gtp | ConvertFrom-Json
$headers = @{
    "ima-openapi-clientid" = $creds.client_id
    "ima-openapi-apikey"   = $creds.api_key
}
```

### 笔记内容模板

#### 1. 📋 户口档案

```
# 📋 宝宝户口档案

## 基本信息
- 姓名：王小明
- 昵称：明宝
- 性别：男宝
- 出生日期：2024-04-20
- 出生医院：
- 出生体重：
- 出生身长：

## 证件信息
- 出生医学证明编号：
- 户口本页码：
- 身份证号（办理后填写）：
- 医保卡号：

## 家长信息
- 父亲姓名：
- 母亲姓名：
- 家庭住址：
- 紧急联系电话：

---
_此档案由婴儿管家初始化创建_ | 2024年
```

#### 2. 🍼 奶量记录

```
# 🍼 奶量记录

> 记录格式：日期 时间 | 类型（母乳/配方奶）| 量（ml）| 备注

## 记录

---
_此笔记由婴儿管家初始化创建_ | 2024年
```

#### 3. 💉 疫苗接种

```
# 💉 疫苗接种记录

> 记录格式：接种日期 | 疫苗名称 | 剂次 | 接种机构 | 下次接种时间 | 反应/备注

## 接种记录

---

## 待接种计划

| 月龄 | 疫苗 | 状态 |
|-----|------|------|
| 出生 | 乙肝（第1针）、卡介苗 | ⬜ |
| 1月 | 乙肝（第2针） | ⬜ |
| 2月 | 脊灰（第1剂）、五联/四联（第1针） | ⬜ |
| 3月 | 脊灰（第2剂）、五联/四联（第2针） | ⬜ |
| 4月 | 脊灰（第3剂）、五联/四联（第3针） | ⬜ |
| 6月 | 乙肝（第3针）、A群流脑（第1针） | ⬜ |
| 8月 | 麻风（第1针） | ⬜ |
| 9月 | A群流脑（第2针） | ⬜ |
| 12月 | 乙脑（第1针） | ⬜ |
| 18月 | 麻腮风、甲肝、五联/四联（第4针） | ⬜ |

_此笔记由婴儿管家初始化创建_ | 2024年
```

#### 4. 😴 睡眠记录

```
# 😴 睡眠记录

> 记录格式：日期 | 入睡时间 | 醒来时间 | 时长 | 类型（夜觉/午睡）| 备注

## 记录

---
_此笔记由婴儿管家初始化创建_ | 2024年
```

#### 5. 🏥 就医记录

```
# 🏥 就医记录

> 记录格式：日期 | 医院/科室 | 主诉症状 | 诊断结果 | 处方用药 | 复诊时间 | 备注

## 记录

---
_此笔记由婴儿管家初始化创建_ | 2024年
```

#### 6. 📦 婴儿用品

```
# 📦 婴儿用品清单

> 记录格式：类别 | 品名/品牌型号 | 购买日期 | 价格 | 购买渠道 | 备注/评价

## 清单

### 喂养类
（待记录）

### 出行类
（待记录）

### 寝具类
（待记录）

### 洗护类
（待记录）

### 衣物类
（待记录）

### 玩具早教类
（待记录）

---
_此笔记由婴儿管家初始化创建_ | 2024年
```

#### 7. 📸 成长照片

```
# 📸 成长照片打卡

> 文字记录格式：日期 | 月龄 | 身高 | 体重 | 头围 | 描述/里程碑事件

## 成长记录

---

## 里程碑打卡

| 里程碑 | 日期 | 备注 |
|-------|------|------|
| 第一次微笑 | | |
| 第一次翻身 | | |
| 第一次坐稳 | | |
| 第一次爬行 | | |
| 第一次站立 | | |
| 第一次走路 | | |
| 第一次叫爸爸/妈妈 | | |
| 第一次长牙 | | |

_此笔记由婴儿管家初始化创建_ | 2024年
```

---

## 第五步：同步笔记到知识库

初始化完成后，将 7 篇笔记添加到知识库。详见 `references/knowledge-base.md`「同步 7 篇笔记到知识库」章节。

完成后更新 MEMORY.md 中的 `baby_knowledge_base_id`。

---

## 第六步：确认完成

所有笔记创建完成后，向用户展示：

```
✅ 婴儿管家初始化完成！

已创建以下 7 篇笔记：

📋 户口档案 — 宝宝基本信息和证件
🍼 奶量记录 — 每日喂奶情况追踪
💉 疫苗接种 — 接种历史和待接种计划
😴 睡眠记录 — 睡眠规律分析
🏥 就医记录 — 门诊/住院及用药记录
📦 婴儿用品 — 用品采购清单
📸 成长照片 — 身高体重和里程碑打卡

已同步到知识库「明宝成长手册」✅

下一步可以开始记录宝宝信息啦！试试说：
「记录今天奶量 150ml」或「宝宝今天睡了多久」
```

---

## 第七步：更新 MEMORY.md

初始化完成后，更新 MEMORY.md 中的婴儿管家信息：

```
## 婴儿管家
- 状态：✅ 已初始化（YYYY-MM-DD）
- baby_knowledge_base_id: <知识库ID>
- 知识库「明宝成长手册」：7 篇笔记已导入
  - 📋 宝宝户口档案（doc: 7449656703349819）
  - 🍼 奶量记录（doc: 7449656711714707）
  - 💉 疫苗接种记录（doc: 7449656720128477）
  - 😴 睡眠记录（doc: 7449656728515410）
  - 🏥 就医记录（doc: 7449656732684979）
  - 📦 婴儿用品清单（doc: 7449656741099809）
  - 📸 成长照片打卡（doc: 7449656749464162）
```

---

# IMA API 调用参考

## import_doc 接口

```powershell
# 每次调用前重新获取凭证
$gtp = "C:\Program Files\QClaw\resources\openclaw\config\skills\ima\get-token.ps1"
$creds = & $gtp | ConvertFrom-Json
$headers = @{
    "ima-openapi-clientid" = $creds.client_id
    "ima-openapi-apikey"   = $creds.api_key
}

# 创建笔记
$payload = @{
    content_format = 1  # Markdown
    content        = "# 标题`n`n正文内容"
} | ConvertTo-Json -Depth 5
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($payload)

$resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/note/v1/import_doc" `
    -Method Post -Headers $headers `
    -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 20

# 返回 doc_id 用于后续追加
$resp.data.doc_id
```

## search_note_book 接口

```powershell
$body = @{
    search_type = 0  # 0=标题，1=正文
    query_info   = @{ title = "关键词" }
    start        = 0
    end          = 20
} | ConvertTo-Json -Compress
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)

$resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/note/v1/search_note_book" `
    -Method Post -Headers $headers `
    -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 15

# $resp.data.docs 包含匹配笔记列表
# $resp.data.docs[].doc.basic_info.docid → 笔记 ID
# $resp.data.total_count → 总数
```

## 错误处理

| 错误码 | 含义 | 处理 |
|--------|------|------|
| 20004 | apikey 鉴权失败 | IMA 授权失效，引导用户重新授权 |
| 100001 | 参数错误 | 检查 JSON 格式和必填字段 |
| 100003 | 服务器内部错误 | 等待 5 秒重试，最多 3 次 |
| 100009 | 超过大小限制 | 拆分内容为多次写入 |
