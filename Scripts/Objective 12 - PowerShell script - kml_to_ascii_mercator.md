```
function kml_to_ascii_mercator {
    param(
        [Parameter(Mandatory=$true, Position=0)]
        [string]$kml_file,
        
        [Parameter(Mandatory=$false)]
        [int]$width = 150  # Default width if not specified
    )

    # Get current console dimensions
    $currentWidth = $Host.UI.RawUI.WindowSize.Width
    $currentHeight = $Host.UI.RawUI.WindowSize.Height

    # Check if requested width is larger than console width
    if ($width -gt $currentWidth) {
        Write-Host "Warning: Requested width ($width) is larger than console width ($currentWidth)" -ForegroundColor Yellow
        Write-Host "Output may be truncated" -ForegroundColor Yellow
    }

    function Convert-ToMercator {
        param (
            [double]$lon,
            [double]$lat
        )
        # Limit latitude to avoid infinite values
        $lat = [Math]::Max([Math]::Min($lat, 85), -85)
        
        $x = $lon
        $y = [Math]::Log([Math]::Tan(($lat + 90) * [Math]::PI / 360)) * 180 / [Math]::PI
        
        return @{
            X = $x
            Y = $y
        }
    }

    # Check if file exists
    if (-not (Test-Path $kml_file)) {
        Write-Host "Error: KML file not found: $kml_file" -ForegroundColor Red
        return
    }

    # Read and parse KML
    try {
        $kmlContent = Get-Content $kml_file -Raw
        $coordinates = [regex]::Match($kmlContent, '<coordinates>(.*?)</coordinates>', 'Singleline').Groups[1].Value.Trim()

        if ([string]::IsNullOrEmpty($coordinates)) {
            Write-Host "Error: No coordinates found in KML file" -ForegroundColor Red
            return
        }
    }
    catch {
        Write-Host ("Error reading KML file: " + $_) -ForegroundColor Red
        return
    }

    # Convert coordinates
    $points = $coordinates -split ' ' | Where-Object { $_ -ne '' } | ForEach-Object {
        $coords = $_ -split ','
        $projected = Convert-ToMercator -lon $coords[0] -lat $coords[1]
        [PSCustomObject]@{
            X = $projected.X
            Y = $projected.Y
        }
    }

    # Handle longitude wrapping
    $previousX = $points[0].X
    for ($i = 1; $i -lt $points.Count; $i++) {
        $diff = $points[$i].X - $previousX
        if ([Math]::Abs($diff) -gt 180) {
            if ($diff -gt 0) {
                $points[$i].X -= 360
            }
            else {
                $points[$i].X += 360
            }
        }
        $previousX = $points[$i].X
    }

    # Find bounds
    $minX = ($points | Measure-Object -Property X -Minimum).Minimum
    $maxX = ($points | Measure-Object -Property X -Maximum).Maximum
    $minY = ($points | Measure-Object -Property Y -Minimum).Minimum
    $maxY = ($points | Measure-Object -Property Y -Maximum).Maximum

    # Calculate ranges
    $rangeX = $maxX - $minX
    $rangeY = $maxY - $minY

    # Calculate height based on aspect ratio
    $aspectRatio = $rangeY / $rangeX
    $height = [Math]::Max([Math]::Floor($width * $aspectRatio * 0.5), 10)

    # Check if calculated height exceeds console
    if ($height -gt $currentHeight) {
        Write-Host "Warning: Calculated height ($height) exceeds console height ($currentHeight)" -ForegroundColor Yellow
        Write-Host "Output may be truncated" -ForegroundColor Yellow
    }

    # Calculate scale with zoom out using provided width
    $scale = [Math]::Min($width / $rangeX, $height / $rangeY) * 0.7

    # Center point
    $centerX = ($minX + $maxX) / 2
    $centerY = ($minY + $maxY) / 2

    # Calculate actual width needed
    $leftmostX = $width
    $rightmostX = 0
    for ($i = 0; $i -lt $points.Count - 1; $i++) {
        $x1 = [Math]::Floor($width/2 + ($points[$i].X - $centerX) * $scale)
        $x2 = [Math]::Floor($width/2 + ($points[$i+1].X - $centerX) * $scale)
        $leftmostX = [Math]::Min($leftmostX, [Math]::Min($x1, $x2))
        $rightmostX = [Math]::Max($rightmostX, [Math]::Max($x1, $x2))
    }

    # Calculate actual required width with padding
    $requiredWidth = $rightmostX - $leftmostX + 4  # Add 4 for padding

    # Create empty canvas
    $canvas = New-Object 'char[,]' $height, $requiredWidth
    for ($y = 0; $y -lt $height; $y++) {
        for ($x = 0; $x -lt $requiredWidth; $x++) {
            $canvas[$y, $x] = ' '
        }
    }

    # Adjust X coordinates to remove empty space
    $xOffset = $leftmostX - 2

    # Draw connected path
    for ($i = 0; $i -lt $points.Count - 1; $i++) {
        $x1 = [Math]::Floor($width/2 + ($points[$i].X - $centerX) * $scale) - $xOffset
        $y1 = [Math]::Floor($height/2 + ($points[$i].Y - $centerY) * $scale * 0.9)
        $x2 = [Math]::Floor($width/2 + ($points[$i+1].X - $centerX) * $scale) - $xOffset
        $y2 = [Math]::Floor($height/2 + ($points[$i+1].Y - $centerY) * $scale * 0.9)
        
        # Draw line between points using ASCII characters
        $steps = [Math]::Max([Math]::Abs($x2 - $x1), [Math]::Abs($y2 - $y1))
        if ($steps -eq 0) { $steps = 1 }
        
        for ($step = 0; $step -le $steps; $step++) {
            $x = [Math]::Floor($x1 + ($x2 - $x1) * $step / $steps)
            $y = [Math]::Floor($y1 + ($y2 - $y1) * $step / $steps)
            
            if ($x -ge 0 -and $x -lt $requiredWidth -and $y -ge 0 -and $y -lt $height) {
                # Determine line character based on direction
                $dx = $x2 - $x1
                $dy = $y2 - $y1
                
                $char = if ([Math]::Abs($dx) -lt 0.0001) {
                    '|'  # Vertical line
                }
                elseif ([Math]::Abs($dy) -lt 0.0001) {
                    '-'  # Horizontal line
                }
                elseif (($dx -gt 0 -and $dy -gt 0) -or ($dx -lt 0 -and $dy -lt 0)) {
                    '/'  # Diagonal up
                }
                else {
                    '\'  # Diagonal down
                }
                
                $canvas[$y, $x] = $char
            }
        }
    }

    # Mark start and end points with special characters
    $startX = [Math]::Floor($width/2 + ($points[0].X - $centerX) * $scale) - $xOffset
    $startY = [Math]::Floor($height/2 + ($points[0].Y - $centerY) * $scale * 0.9)
    $endX = [Math]::Floor($width/2 + ($points[-1].X - $centerX) * $scale) - $xOffset
    $endY = [Math]::Floor($height/2 + ($points[-1].Y - $centerY) * $scale * 0.9)
    
    if ($startX -ge 0 -and $startX -lt $requiredWidth -and $startY -ge 0 -and $startY -lt $height) {
        $canvas[$startY, $startX] = 'S'
    }
    if ($endX -ge 0 -and $endX -lt $requiredWidth -and $endY -ge 0 -and $endY -lt $height) {
        $canvas[$endY, $endX] = 'E'
    }

    # Output the path (flipped vertically)
    Write-Host ("`nDisplaying path from " + $kml_file + "`n") -ForegroundColor Cyan
    for ($y = $height - 1; $y -ge 0; $y--) {
        $line = ""
        for ($x = 0; $x -lt $requiredWidth; $x++) {
            $line += $canvas[$y, $x]
        }
        Write-Host $line
    }
}

kml_to_ascii_mercator "$HOME\Downloads\ELF-HAWK-dump.kml" -width 1000

```