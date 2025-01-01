```
$filePath = "$HOME\Downloads\frostbitartifacts\frostbit_core_dump.*"
$patterns = @(
    'POST /api/v1/bot/([a-z0-9-]+)/',
    '"encryptedkey":"([^"]+)"',
    'CLIENT_TRAFFIC_SECRET_0\s+(\w+)\s+(\w+)',
    'SERVER_TRAFFIC_SECRET_0\s+(\w+)\s+(\w+)',
    'CLIENT_HANDSHAKE_TRAFFIC_SECRET\s+(\w+)\s+(\w+)',
    'SERVER_HANDSHAKE_TRAFFIC_SECRET\s+(\w+)\s+(\w+)'
)
$matchCounts = [ordered]@{}
foreach ($pattern in $patterns) {
    $matchCounts[$pattern] = 0
}
$totalMatchCount = 0
Select-String -Path $filePath -Pattern ($patterns -join '|') -AllMatches | ForEach-Object {
    $match = $_
    $matchedPattern = $patterns | Where-Object { $match.Matches.Value -match $_ }
    $matchedPattern | ForEach-Object {
        $match.Matches.Value
        $matchCounts[$_]++
        $totalMatchCount++
        if ($matchCounts[$_] -ge 1 -and $totalMatchCount -ge $patterns.Count) {
            break
        }
    }
}

```