```
function decode_true_false ($csv_filename) {
		
	# Read the CSV file without headers
	$data = Get-Content -Path $csv_filename

	Write-Host "Number of lines in file: $($data.Count)"
	Write-Host "First few lines of data:"
	$data | Select-Object -First 3 | ForEach-Object {
		Write-Host $_
	}

	# Convert TRUE/FALSE to binary string
	$binaryString = ""
	$trueCount = 0
	$falseCount = 0

	foreach ($line in $data) {
		$values = $line.Trim() -split ','
		
		foreach ($value in $values) {
			$trimmedValue = $value.Trim()
			
			if ($trimmedValue -eq '"TRUE"') {
				$binaryString += "1"
				$trueCount++
			} 
			elseif ($trimmedValue -eq '"FALSE"') {
				$binaryString += "0"
				$falseCount++
			}
		}
	}

	Write-Host "`nStatistics:"
	Write-Host "TRUE values found: $trueCount"
	Write-Host "FALSE values found: $falseCount"
	Write-Host "Binary string length: $($binaryString.Length) bits"

	if ($binaryString.Length -gt 0) {
		# Convert binary to text (8 bits per character)
		$text = ""
		for ($i = 0; $i -lt $binaryString.Length; $i += 8) {
			if ($i + 8 -le $binaryString.Length) {
				$byte = $binaryString.Substring($i, 8)
				$charValue = [Convert]::ToInt32($byte, 2)
				$text += [char]$charValue
			}
		}

		Write-Host "`nDecoded text:"
		Write-Host $text
	}

}

# decode_true_false "$HOME\Downloads\ELF-HAWK-dump-cleaned.csv"

```