# shzqh-quant-tools
Open source tools for futures data analysis and quantitative research.
Write-Host "=== 数智shzqh.cn 自动化推广推送 ===" -ForegroundColor Cyan

if (!(Test-Path $IndexFile)) {
    Write-Host "❌ 找不到 _index.json，请先运行发布脚本 publish-hermes.ps1" -ForegroundColor Red
    exit
}

# 1. 读取今日数据
$json = [System.IO.File]::ReadAllText($IndexFile, $Utf8NoBom)
$data = $json | ConvertFrom-Json
# 取融合排序前 5 名（多发一点丰富文案）
$items = $data.items | Sort-Object { [int]$_.map_rank } | Select-Object -First 5

if ($items.Count -eq 0) {
    Write-Host "⚠️ 今日无数据，跳过推送。" -ForegroundColor Yellow
    exit
}

# 2. 组装今日推广链接池（大页面天天推，新页面也不能少）
$urlsToPush = @(
    "$Domain/action_maps/action_map_latest.html",
    "$Domain/action_maps/fusion_scan_latest.html",
    "$Domain/member.html#pool",
    "$Domain/",
    "$Domain/strategies/pool_history.html",
    "$Domain/strategies/"
)

# ★ 2.1 动态抓取今天生成或修改的 HTML 页面 (提升 IndexNow 覆盖面)
# 获取网站目录下最近 24 小时内修改过的 .html 文件
$todayFiles = Get-ChildItem -Path $Local -Filter *.html -Recurse -ErrorAction SilentlyContinue | 
              Where-Object { $_.LastWriteTime -gt (Get-Date).AddHours(-24) }

foreach ($file in $todayFiles) {
    # 排除非内容页面，避免推送无效链接
    if ($file.Name -match '^(404\.html|index\.html)$' -or $file.FullName -match '\\node_modules\\') {
        continue
    }
    
    # 将本地绝对路径转为标准 URL 路径
    $relPath = $file.FullName.Replace("$Local\", "").Replace("\", "/")
    $urlsToPush += "$Domain/$relPath"
}

# 3. 生成社群推广文案（合规版）
$promoText = "【数智量化 · 今日关注池复盘】`n"
$promoText += "📅 数据基准日：$((Get-Date).ToString('yyyy-MM-dd'))`n"
$promoText += "📊 融合评分模型输出 Top 5：`n"
$promoText += "------------------------`n"

$i = 1
foreach ($it in $items) {
    # 安全取值
    $code = if ($it.code) { $it.code } else { "--" }
    $name = if ($it.name) { $it.name } else { "--" }
    # 合规化方向：偏多/偏空/观望
    $rawDir = if ($it.direction) { $it.direction } else { "震荡" }
    if ($rawDir -eq "多") { $dirTxt = "方向偏多" }
    elseif ($rawDir -eq "空") { $dirTxt = "方向偏空" }
    else { $dirTxt = "方向震荡" }

    $sc   = if ($it.score -ne $null) { $it.score } else { "--" }
    
    # 提取核心钩子（关注区和风险位，去掉过长的字符串，只取前 20 个字符）
    $entryRaw = if ($it.entry_src) { $it.entry_src } else { "" }
    $entryShort = if ($entryRaw.Length -gt 25) { $entryRaw.Substring(0, 25) + "..." } else { $entryRaw }
    
    # 单品种详情页链接
    $detailUrl = "$Domain/fusion_scan/$($code.ToLower()).html"
    
    $promoText += "$i. 【$code $name】融合分 $sc ｜ $dirTxt`n"
    if ($entryShort) { $promoText += "   🎯 结构参考区：$entryShort`n" }
    $promoText += "   👉 完整风控分析与图表：$detailUrl`n`n"
    
    # ★ 将最新的详情页和简报页追加到队列，保证不会丢
    $urlsToPush += $detailUrl
    if ($it.latest_file) { 
        $urlsToPush += "$Domain/strategies/$($it.latest_file)" 
    }
    $i++
}

$promoText += "------------------------`n"
$promoText += "💡 解锁全品种关注池与历史复盘快照：`n"
$promoText += "$Domain/member.html#pool`n`n"
$promoText += "⚠️ 风险提示：以上数据为量化模型历史推演与结构观测，仅供研究复盘参考，不构成任何交易建议。市场有风险，投资需谨慎。"

# ★ 去重，保持原顺序（大页面优先，今日新生成页面其次，Top 5 细节兜底）
$urlsToPush = $urlsToPush | Select-Object -Unique

# ★ 严格截断：百度不管配额多少，我们按优先级取前 9 条 (Top页面优先)
$baiduUrls = $urlsToPush | Select-Object -First 9

# ★ 写入本地 txt 并自动复制到剪贴板
[System.IO.File]::WriteAllText($TxtOutput, $promoText, $Utf8NoBom)
Set-Clipboard -Value $promoText
Write-Host "✅ 推广文案已生成并复制到剪贴板！直接去微信群/朋友圈 Ctrl+V 即可。" -ForegroundColor Green
