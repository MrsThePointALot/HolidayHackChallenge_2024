
```
function rot13 {
    param (
        [Parameter(Position=0, Mandatory=$true)]
        [string]$ciphertext
    )
    
    try {
        $alphabet = "abcdefghijklmnopqrstuvwxyz"
        $upperAlphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
        $vowels = "aeiouAEIOU"
        $likelyWords = @()

        # Function to check if a word looks like English
        function Test-EnglishLike {
            param([string]$word)
            
            # Convert to lowercase for checking
            $word = $word.ToLower()
            
            # Rule 1: No more than 3 consonants in a row
            if ($word -match "(?i)[^aeiou]{4,}") {
                return $false
            }
            
            # Rule 2: Must contain at least one vowel
            if (-not ($word -match "[aeiou]")) {
                return $false
            }
            
            # Rule 4: Common English letter pairs
            $commonPairs = "th|ch|sh|ph|wh|tr|cr|br|fr|dr|sp|st|pl|cl|bl|sl"
            if ($word -match $commonPairs) {
                return $true  # Bonus points for common pairs
            }
            
            return $true
        }

        Write-Host "Analyzing '$ciphertext'..."
        Write-Host "Likely English words marked with [*]`n"
        
        for ($shift = 0; $shift -lt 26; $shift++) {
            $result = ""
            foreach ($char in $ciphertext.ToCharArray()) {
                if ($alphabet.Contains($char.ToString().ToLower())) {
                    # Handle lowercase
                    $pos = $alphabet.IndexOf($char.ToString().ToLower())
                    $newPos = ($pos + $shift) % 26
                    $newChar = if ([char]::IsUpper($char)) {
                        $upperAlphabet[$newPos]
                    } else {
                        $alphabet[$newPos]
                    }
                    $result += $newChar
                } else {
                    # Non-alphabetic characters remain unchanged
                    $result += $char
                }
            }
            
            # Check if result looks like English
            $marker = if (Test-EnglishLike $result) { 
                $likelyWords += "ROT$shift : $result"
                "[*]" 
            } else { 
                "   " 
            }
            
            Write-Host "ROT$($shift.ToString().PadLeft(2)) : $($result.PadRight(10)) $marker"
        }
        
        if ($likelyWords.Count -gt 0) {
            Write-Host "`nMost likely candidates:"
            $likelyWords | ForEach-Object { Write-Host $_ }
        }
    }
    catch {
        Write-Host "Error: $_"
        Write-Host $_.ScriptStackTrace
    }
}

# rot13 ciphertext

```