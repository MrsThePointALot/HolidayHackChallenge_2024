```
function windiff {
    param (
        [Parameter(Position=0)]
        [string]$File1,
        
        [Parameter(Position=1)]
        [string]$File2,
        
        [Parameter()]
        [switch]$apply,
        
        [Parameter()]
        [switch]$backup = $true
    )

    # Function to create diff
    function Create-Diff {
        param($src, $dst)
        $output = New-Object System.Text.StringBuilder
        $output.AppendLine("--- $src") | Out-Null
        $output.AppendLine("+++ $dst") | Out-Null

        $srcLines = Get-Content $src
        $dstLines = Get-Content $dst

        for ($i = 0; $i -lt [Math]::Max($srcLines.Count, $dstLines.Count); $i++) {
            if ($i -ge $srcLines.Count) {
                $output.AppendLine("@@ -$i,0 +$($i+1),1 @@") | Out-Null
                $output.AppendLine("+$($dstLines[$i])") | Out-Null
                $output.AppendLine("") | Out-Null
            }
            elseif ($i -ge $dstLines.Count) {
                $output.AppendLine("@@ -$($i+1),1 +$i,0 @@") | Out-Null
                $output.AppendLine("-$($srcLines[$i])") | Out-Null
                $output.AppendLine("") | Out-Null
            }
            elseif ($srcLines[$i] -ne $dstLines[$i]) {
                $output.AppendLine("@@ -$($i+1),1 +$($i+1),1 @@") | Out-Null
                $output.AppendLine("-$($srcLines[$i])") | Out-Null
                $output.AppendLine("+$($dstLines[$i])") | Out-Null
                $output.AppendLine("") | Out-Null
            }
        }
        return $output.ToString()
    }

    # Function to apply diff
    function Apply-Diff {
        param($original, $diff)
        if ($backup) {
            Copy-Item $original "$original.bak"
        }

        $diffContent = Get-Content $diff
        $fileLines = Get-Content $original

        $i = 2  # Skip headers
        while ($i -lt $diffContent.Count) {
            if ($diffContent[$i] -match '@@ -(\d+),(\d+) \+(\d+),(\d+) @@') {
                $startLine = [int]$Matches[1] - 1
                $linesCount = [int]$Matches[2]
                
                $i++
                $changes = @()
                while ($i -lt $diffContent.Count -and -not ($diffContent[$i] -match '@@ ')) {
                    if ($diffContent[$i] -match '^[+-]') {
                        $changes += $diffContent[$i]
                    }
                    $i++
                }
                
                $newLines = @()
                foreach ($change in $changes) {
                    if ($change.StartsWith('+')) {
                        $newLines += $change.Substring(1)
                    }
                }
                
                $fileLines = $fileLines[0..($startLine-1)] + 
                            $newLines + 
                            $fileLines[($startLine + $linesCount)..($fileLines.Count-1)]
            }
            else { $i++ }
        }
        $fileLines | Set-Content $original
    }

    # Main logic
    if ($apply) {
        if (-not $File1) { Write-Error "Original file path required"; return }
        if (-not $File2) { Write-Error "Diff file path required"; return }
        Apply-Diff $File1 $File2
    }
    else {
        if (-not $File1) { Write-Error "Source file path required"; return }
        if (-not $File2) { Write-Error "Target file path required"; return }
        return Create-Diff $File1 $File2
    }
}
#
# Usage examples:
#
# windiff original.js modified.js > changes.diff
# windiff -apply original.js changes.diff
# windiff -apply -backup:$false original.js changes.diff

```