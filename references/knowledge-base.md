# 知识库同步参考

> 本文件供 AI agent 执行知识库操作时参考。

## 凭证获取

每次 API 调用均通过 `get-token.ps1` 动态获取凭证：

```powershell
$gtp = "C:\Program Files\QClaw\resources\openclaw\config\skills\ima\get-token.ps1"
$creds = & $gtp | ConvertFrom-Json
$headers = @{
    "ima-openapi-clientid" = $creds.client_id
    "ima-openapi-apikey"   = $creds.api_key
}
```

---

## 知识库 ID 查找

### 从 MEMORY.md 读取

每次操作前先检查 MEMORY.md：

```powershell
$memPath = "C:\Users\admin\.qclaw\workspace\MEMORY.md"
$memContent = Get-Content $memPath -Raw
if ($memContent -match 'baby_knowledge_base_id:\s*(.+)') {
    $kbId = $matches[1].Trim()
} else {
    $kbId = $null
}
```

### 用名称搜索知识库

若 MEMORY.md 中无记录，或用户指定了其他知识库名称，用 `search_knowledge_base` 搜索：

```powershell
$body = @{ query = "知识库名称"; cursor = ""; limit = 10 } | ConvertTo-Json -Compress
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)

$resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/wiki/v1/search_knowledge_base" `
    -Method Post -Headers $headers `
    -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 15

if ($resp.data.info_list.Count -gt 0) {
    $kbId = $resp.data.info_list[0].kb_id
    $kbName = $resp.data.info_list[0].kb_name
    Write-Host "KB_ID: $kbId"
} else {
    Write-Host "KB_NOT_FOUND"
}
```

### 列出用户所有知识库（空 query）

```powershell
$body = @{ query = ""; cursor = ""; limit = 20 } | ConvertTo-Json -Compress
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)
$resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/wiki/v1/search_knowledge_base" `
    -Method Post -Headers $headers `
    -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 15
# $resp.data.info_list → 知识库列表
```

---

## 添加笔记到知识库

将已有笔记（通过 `doc_id`）添加到知识库，笔记仅作为引用存储，不复制内容：

```powershell
$body = @{
    media_type = 11
    note_info = @{ content_id = $docId }
    title = "笔记标题"
    knowledge_base_id = $kbId
} | ConvertTo-Json -Depth 3
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)

$resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/wiki/v1/add_knowledge" `
    -Method Post -Headers $headers `
    -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 20

# 响应处理
if ($resp.retcode -eq 0) {
    Write-Host "ADD_OK"
} elseif ($resp.retcode -eq 220001) {
    Write-Host "ADD_DUPLICATE"  # 笔记已在知识库中，无需处理
} else {
    Write-Host "ADD_FAILED: $($resp.errmsg)"
}
```

---

## 同步 7 篇笔记到知识库（批量）

初始化完成后，或用户要求导入时，执行此流程：

```powershell
# 7 篇笔记的 doc_id（从 list_note_by_folder_id 获取）
$babyNotes = @(
    @{ docId = "7449656703349819"; title = "📋 宝宝户口档案" },
    @{ docId = "7449656711714707"; title = "🍼 奶量记录" },
    @{ docId = "7449656720128477"; title = "💉 疫苗接种记录" },
    @{ docId = "7449656728515410"; title = "😴 睡眠记录" },
    @{ docId = "7449656732684979"; title = "🏥 就医记录" },
    @{ docId = "7449656741099809"; title = "📦 婴儿用品清单" },
    @{ docId = "7449656749464162"; title = "📸 成长照片打卡" }
)

$success = 0; $duplicate = 0; $failed = 0
foreach ($note in $babyNotes) {
    $body = @{
        media_type = 11
        note_info = @{ content_id = $note.docId }
        title = $note.title
        knowledge_base_id = $kbId
    } | ConvertTo-Json -Depth 3
    $utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)

    $resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/wiki/v1/add_knowledge" `
        -Method Post -Headers $headers `
        -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 20

    if ($resp.retcode -eq 0) { $success++ }
    elseif ($resp.retcode -eq 220001) { $duplicate++ }
    else { $failed++; Write-Host "Failed: $($note.title) - $($resp.errmsg)" }
    Start-Sleep -Milliseconds 300
}

Write-Host "同步完成: 成功=$success 重复=$duplicate 失败=$failed"
```

---

## 查看知识库内容

```powershell
$body = @{ knowledge_base_id = $kbId; cursor = ""; limit = 20 } | ConvertTo-Json -Compress
$utf8 = [System.Text.Encoding]::UTF8.GetBytes($body)

$resp = Invoke-RestMethod -Uri "https://ima.qq.com/openapi/wiki/v1/get_knowledge_list" `
    -Method Post -Headers $headers `
    -Body $utf8 -ContentType "application/json; charset=utf-8" -TimeoutSec 15

# $resp.data.knowledge_list → 知识库中的所有条目
# folder_id 存在 → 文件夹
# media_id 存在 → 文件/笔记引用
```

---

## 更新 MEMORY.md 中的知识库 ID

同步完成后，更新 MEMORY.md：

```powershell
$memPath = "C:\Users\admin\.qclaw\workspace\MEMORY.md"
$memContent = Get-Content $memPath -Raw

# 替换或追加 baby_knowledge_base_id
if ($memContent -match 'baby_knowledge_base_id:') {
    $memContent = $memContent -replace 'baby_knowledge_base_id:.*', "baby_knowledge_base_id: $kbId"
} else {
    $memContent = $memContent -replace '(## 婴儿管家)', "`$1`n- baby_knowledge_base_id: $kbId"
}
Set-Content -Path $memPath -Value $memContent
```

---

## 错误码参考

| retcode | 含义 | 处理 |
|---------|------|------|
| 0 | 成功 | 静默 |
| 220001 | 知识重复添加 | 静默，笔记已在知识库中 |
| 其他非0 | 失败 | 记录错误，可重试一次，仍失败则告知用户 |
