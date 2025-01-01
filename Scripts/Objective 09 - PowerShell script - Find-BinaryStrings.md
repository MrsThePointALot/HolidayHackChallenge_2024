```
function Find-BinaryStrings {
    param (
        [Parameter(Mandatory=$true)]
        [string]$FilePath,
        
        [Parameter(Mandatory=$false)]
        [string]$SearchString = ""
    )

    try {
        # Read the file as bytes
        $bytes = [System.IO.File]::ReadAllBytes($FilePath)
        
        # Array to store all found strings with their positions
        $foundStrings = @()
        
        # Find all valid length-prefixed strings
        for ($i = 0; $i -lt $bytes.Length - 2; $i++) {
            # Find sequence of printable characters
            $maxLen = [Math]::Min(64, $bytes.Length - $i - 1)
            $printableLen = 0
            
            for ($j = 0; $j -lt $maxLen; $j++) {
                $b = $bytes[$i + 1 + $j]
                if ($b -lt 32 -or $b -gt 126) {
                    break
                }
                $printableLen++
            }
            
            if ($printableLen -gt 0) {
                # Check if byte before sequence is the length
                if ($bytes[$i] -eq $printableLen) {
                    $stringBytes = $bytes[($i + 1)..($i + $printableLen)]
                    $string = [System.Text.Encoding]::ASCII.GetString($stringBytes)
                    $foundStrings += @{
                        Position = $i
                        String = $string
                    }
                }
                # Check if first byte of sequence is the length of remaining sequence
                elseif ($bytes[$i + 1] -eq ($printableLen - 1)) {
                    $stringBytes = $bytes[($i + 2)..($i + $printableLen)]
                    $string = [System.Text.Encoding]::ASCII.GetString($stringBytes)
                    $foundStrings += @{
                        Position = $i + 1
                        String = $string
                    }
                }
                
                # Skip ahead past this sequence
                $i += $printableLen
            }
        }
        
        # Find search string and return next string
        if ($SearchString) {
            for ($i = 0; $i -lt $foundStrings.Count - 1; $i++) {
                if ($foundStrings[$i].String -eq $SearchString) {
                    Write-Host "Output:" $foundStrings[$i + 1].String
                    return
                }
            }
        }

    } catch {
        Write-Error "Error reading file: $_"
    }
}

Find-BinaryStrings -FilePath "$HOME\Desktop\HHC2024\SantaSwipeSecure\base\resources.pb" -SearchString "ek"

Find-BinaryStrings -FilePath "$HOME\Desktop\HHC2024\SantaSwipeSecure\base\resources.pb" -SearchString "iv"

```