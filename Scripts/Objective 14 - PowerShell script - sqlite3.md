```
function sqlite3 {
    param (
        [Parameter(Position=0, Mandatory=$true)]
        [string]$filename
    )
    
    try {
        clear

        # Resolve relative path
        $fullPath = $ExecutionContext.SessionState.Path.GetUnresolvedProviderPathFromPSPath($filename)
        $bytes = [System.IO.File]::ReadAllBytes($fullPath)
        Write-Host "WARNING: This script only dumps SQLite3 databases with specific schema."
        Write-Host "Scanning $fullPath"
        Write-Host ""

        # === Parse SQLite Header ===
        Write-Host "=== SQLite Header ==="
        # Header string (offset 0, 16 bytes)
        $header = [System.Text.Encoding]::ASCII.GetString($bytes[0..15])
        Write-Host "Header string: $header"
        
        # Page size (offset 16, 2 bytes, big-endian)
        $pageSize = ($bytes[16] * 256) + $bytes[17]
        if ($pageSize -eq 1) { $pageSize = 65536 }
        Write-Host "Page size: $pageSize bytes (0x$($pageSize.ToString('X4')))"
        
        # File format versions (offset 18-19)
        $writeVersion = $bytes[18]
        $readVersion = $bytes[19]
        Write-Host "Write version: $writeVersion"
        Write-Host "Read version: $readVersion"
        
        # Reserved space per page (offset 20)
        $reservedSpace = $bytes[20]
        Write-Host "Reserved space per page: $reservedSpace bytes"
        
        # Payload fractions (offset 21-23)
        Write-Host "Maximum embedded payload fraction: $($bytes[21])"
        Write-Host "Minimum embedded payload fraction: $($bytes[22])"
        Write-Host "Leaf payload fraction: $($bytes[23])"
        
        # Database size in pages (offset 28, 4 bytes)
        $dbSize = ($bytes[28] -shl 24) -bor ($bytes[29] -shl 16) -bor ($bytes[30] -shl 8) -bor $bytes[31]
        Write-Host "Database size: $dbSize pages"
        
        # Schema format (offset 44, 4 bytes)
        $schemaFormat = ($bytes[44] -shl 24) -bor ($bytes[45] -shl 16) -bor ($bytes[46] -shl 8) -bor $bytes[47]
        Write-Host "Schema format: $schemaFormat"
        
        # Text encoding (offset 56, 4 bytes)
        $textEncoding = ($bytes[56] -shl 24) -bor ($bytes[57] -shl 16) -bor ($bytes[58] -shl 8) -bor $bytes[59]
        $encodingType = switch($textEncoding) {
            1 { "UTF-8" }
            2 { "UTF-16le" }
            3 { "UTF-16be" }
            default { "Unknown" }
        }
        Write-Host "Text encoding: $encodingType ($textEncoding)"
        
        # === Parse Table Definition ===
        Write-Host "`n=== Table Definition ==="
        
        # Tables are defined in the first page after header (offset 100)
        $firstPageOffset = 100
        
        # Get number of cells from offset 103-104 (2 bytes, big endian)
        $numCells = ($bytes[103] * 256) + $bytes[104]
        Write-Host "Number of objects in first page: $numCells"
        Write-Host "---------------------------------"
        
        # Cell pointer array starts at offset 108
        for ($i = 0; $i -lt $numCells; $i++) {
            $cellOffset = ($bytes[108 + ($i * 2)] * 256) + $bytes[109 + ($i * 2)]
            Write-Host "---------------------------------"
            Write-Host "Processing object at offset 0x$($cellOffset.ToString('X4'))"
            
            # Show first few bytes of the cell
            Write-Host "Header bytes: "
            for ($j = 0; $j -lt 8; $j++) {
                Write-Host -NoNewline "$($bytes[$cellOffset + $j].ToString('X2')) "
            }
            Write-Host ""
            
            $isTable = $false
            
            # Look for table pattern
            for ($j = 0; $j -lt 7; $j++) {
                if ($bytes[$cellOffset + $j] -eq 0x07 -and $bytes[$cellOffset + $j + 1] -eq 0x17) {
                    $isTable = $true
                    $rootPage = $bytes[$cellOffset + $j - 1]  # Root page is before pattern
                    break
                }
            }
            
            if ($isTable) {
                Write-Host "This is a TABLE object"
                Write-Host "Root Page: $rootPage"
                
                # First get the table name and SQL definition
                $currentOffset = $cellOffset
                $recordHeaderSize = 0
                do {
                    $b = $bytes[$currentOffset]
                    $recordHeaderSize = ($recordHeaderSize -shl 7) -bor ($b -band 0x7f)
                    $currentOffset++
                } while ($b -band 0x80)
                
                # Skip column type definitions
                $headerSize = $bytes[$currentOffset]
                $currentOffset += $headerSize
                
                # Find "table" keyword and get name
                $nameStart = $currentOffset + 5  # Skip "table"
                $nameEnd = $nameStart
                while ($bytes[$nameEnd] -ne 0x02 -and $bytes[$nameEnd] -ne 0x04) {
                    $nameEnd++
                }
                
                $name = [System.Text.Encoding]::ASCII.GetString($bytes[$nameStart..($nameEnd-1)])
                if ($name -match '(.+?)\1') {
                    $name = $matches[1]
                }
                Write-Host "Table Name: $name"
                
                # Get SQL definition
                $sqlStart = $nameEnd + 1
                $sqlEnd = $sqlStart
                while ($bytes[$sqlEnd] -ne 0x29) {  # Look for closing parenthesis
                    $sqlEnd++
                }
                $sql = [System.Text.Encoding]::ASCII.GetString($bytes[$sqlStart..($sqlEnd)])
                Write-Host "Table Definition:"
                Write-Host "--"
                Write-Host "CREATE TABLE $name ($sql)"
                Write-Host "--"
                
                # Parse column definitions
                $columnDefs = @()
                if ($sql -match "CREATE TABLE \w+ \(([\s\S]*?)\)$") {
                    $columnDefs = $matches[1] -split ',\s*(?=\w+)'
                }
                
                # Now process the table data
                $tableOffset = $pageSize * $rootPage
                $contentAreaStart = ($bytes[$tableOffset + 5] * 256) + $bytes[$tableOffset + 6]
                $contentOffset = $tableOffset + $contentAreaStart
                
                # Get number of cells in data page
                $numDataCells = ($bytes[$tableOffset + 3] * 256) + $bytes[$tableOffset + 4]
                Write-Host "Found $numDataCells row(s) in data page"
                
                # Cell pointer array starts at offset 8 in data page
                for ($row = 0; $row -lt $numDataCells; $row++) {
                    $cellPointer = ($bytes[$tableOffset + 8 + ($row * 2)] * 256) + $bytes[$tableOffset + 8 + ($row * 2) + 1]
                    $rowOffset = $tableOffset + $cellPointer
                    
                    Write-Host -NoNewline "`nRow $($row + 1) header bytes: "
                    for ($j = 0; $j -lt 8; $j++) {
                        Write-Host -NoNewline "$($bytes[$rowOffset + $j].ToString('X2')) "
                    }
                    Write-Host ""
                    Write-Host "--"
                    
                    $recordLength = $bytes[$rowOffset]
                    $numColumns = $bytes[$rowOffset + 1]
                    $headerLength = $bytes[$rowOffset + 2]
                    
                    $serialTypes = @()
                    $currentOffset = $rowOffset + 3
                    for ($j = 0; $j -lt $headerLength; $j++) {
                        $serialType = $bytes[$currentOffset + $j]
                        $serialTypes += $serialType
                    }
                    
                    $dataOffset = $rowOffset + 3 + $headerLength
                    $columnIndex = 0
                    foreach ($serialType in $serialTypes) {
                        if ($columnIndex -ge $columnDefs.Count) { break }  # Stop if we've run out of column definitions
                        $columnName = $columnDefs[$columnIndex] -replace '^\s*(\w+)\s+.*$', '$1'
                        if ($columnName -eq "PRIMARY") { 
                            $columnIndex++
                            continue 
                        }
                        Write-Host -NoNewline ($columnName.PadRight(15) + ": ")
                        
                        if ($serialType -ge 13 -and $serialType % 2 -eq 1) {
                            # Text string
                            $length = ($serialType - 13) / 2
                            $text = [System.Text.Encoding]::ASCII.GetString($bytes[($dataOffset-1)..($dataOffset + $length - 2)])
                            Write-Host "$text"
                            $dataOffset += $length
                        }
                        elseif ($serialType -eq 9) { Write-Host "1" }
                        elseif ($serialType -eq 0) { Write-Host "NULL" }
                        elseif ($serialType -eq 1) {
                            $value = [Int16]$bytes[$dataOffset]
                            Write-Host "$value"
                            $dataOffset += 1
                        }
                        elseif ($serialType -eq 2) {
                            $value = ($bytes[$dataOffset] -shl 8) -bor $bytes[$dataOffset + 1]  # 16-bit integer
                            Write-Host "$value"
                            $dataOffset += 2
                        }
                        elseif ($serialType -eq 3) {
                            $value = ($bytes[$dataOffset] -shl 16) -bor ($bytes[$dataOffset + 1] -shl 8) -bor $bytes[$dataOffset + 2]  # 24-bit integer
                            Write-Host "$value"
                            $dataOffset += 3
                        }
                        elseif ($serialType -eq 4) {
                            $value = ($bytes[$dataOffset] -shl 24) -bor ($bytes[$dataOffset + 1] -shl 16) -bor ($bytes[$dataOffset + 2] -shl 8) -bor $bytes[$dataOffset + 3]  # 32-bit integer
                            Write-Host "$value"
                            $dataOffset += 4
                        }
                        elseif ($serialType -eq 5) {
                            $value = [BitConverter]::ToInt64($bytes[$dataOffset..($dataOffset+5)] + @(0,0), 0)  # 48-bit integer
                            Write-Host "$value"
                            $dataOffset += 6
                        }
                        elseif ($serialType -eq 6) {
                            $value = [BitConverter]::ToInt64($bytes[$dataOffset..($dataOffset+7)], 0)  # 64-bit integer
                            Write-Host "$value"
                            $dataOffset += 8
                        }
                        elseif ($serialType -eq 7) {
                            $value = [BitConverter]::ToDouble($bytes[$dataOffset..($dataOffset+7)], 0)  # 64-bit float
                            Write-Host "$value"
                            $dataOffset += 8
                        }
                        elseif ($serialType -eq 8) {
                            Write-Host "0"  # Integer 0
                        }
                        elseif ($serialType % 2 -eq 0 -and $serialType -ge 12) {
                            # BLOB
                            $length = ($serialType - 12) / 2
                            $blob = $bytes[$dataOffset..($dataOffset + $length - 1)]
                            Write-Host "BLOB[$length]: $($blob -join ' ')"
                            $dataOffset += $length
                        }
                        else {
                            Write-Host "<type $serialType not implemented>"
                        }
                        $columnIndex++
                    }
                    Write-Host "--"
                }
            } else {
                Write-Host "This is an INDEX object - we are not interested in this"
            }
            Write-Host "---------------------------------"
        }
        Write-Host "---------------------------------"
        Write-Host ""
    }
    catch {
        Write-Host "Error: $_"
        Write-Host $_.ScriptStackTrace
    }
}

# sqlite sqlite.db

```