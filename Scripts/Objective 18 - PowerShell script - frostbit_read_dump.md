```
function frostbit_read_dump {
  $filePath = "$HOME\Downloads\frostbitartifacts\frostbit_core_dump.*"
  $patterns = @(
      'https://api.frostbit.app/view/[0-9a-zA-Z?=/-]*',
      'https://api.frostbit.app/.*/session',
      '"encryptedkey":"([^"]+)"'
  )
  $matchCounts = [ordered]@{}
  foreach ($pattern in $patterns) {
      $matchCounts[$pattern] = 0
  }
  $totalMatchCount = 0
  Write-Host "`n"  
  Select-String -Path $filePath -Pattern ($patterns -join '|') -AllMatches | ForEach-Object {
      $match = $_
      $matchedPattern = $patterns | Where-Object { $match.Matches.Value -match $_ }
      $matchedPattern | % {
          $match.Matches.Value
          Write-Host "`n"
          $matchCounts[$_]++
          $totalMatchCount++
          if ($matchCounts[$_] -ge 1 -and $totalMatchCount -ge $patterns.Count) {
              break
          }
      }
  }
}

frostbit_read_dump

```