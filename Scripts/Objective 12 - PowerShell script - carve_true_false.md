
```
function carve_true_false ($csv_filename) {
	# Read the CSV file
	$csvData = Import-Csv -Path $csv_filename

	# Get all column names
	$columns = $csvData[0].PSObject.Properties.Name

	Write-Host "Original columns count: $($columns.Count)"

	# Find columns with unique values, excluding longitude and latitude
	$columnsToKeep = @()
	foreach ($column in $columns) {
		# Skip longitude and latitude columns
		if ($column -eq "OSD.longitude" -or $column -eq "OSD.latitude") {
			Write-Host "Skipping column '$column' (specified for removal)"
			continue
		}

		# Get unique values for this column (excluding empty or null)
		$uniqueValues = $csvData.$column | Where-Object { $_ -ne $null -and $_ -ne '' } | Select-Object -Unique
		
		# If there's more than one unique value, keep the column
		if ($uniqueValues.Count -gt 1) {
			$columnsToKeep += $column
			Write-Host "Keeping column '$column' (has varying values)"
		} else {
			$value = if ($uniqueValues.Count -eq 1) { $uniqueValues[0] } else { "<empty>" }
			Write-Host "Removing column '$column' with constant value: $value"
		}
	}

	Write-Host "`nColumns to keep: $($columnsToKeep.Count)"

	# Create new CSV with only varying columns
	$newCsv = $csvData | Select-Object $columnsToKeep

	# Convert to CSV without header and save
	$outputFile = "ELF-HAWK-dump-cleaned.csv"
	$newCsv | ConvertTo-Csv -NoTypeInformation | Select-Object -Skip 1 | Out-File $outputFile

	Write-Host "Saved cleaned CSV to $outputFile without header"
}

# carve_true_false "$HOME\Downloads\ELF-HAWK-dump.csv"

```