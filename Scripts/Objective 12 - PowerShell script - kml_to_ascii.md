```
function kml_to_ascii {
    param(
        [string]$kml_file
    )

    # Required console dimensions
    $requiredWidth = 150
    $requiredHeight = 30

    # Get current console dimensions
    $currentWidth = $Host.UI.RawUI.WindowSize.Width
    $currentHeight = $Host.UI.RawUI.WindowSize.Height

    # Check if console is large enough
    if ($currentWidth -lt $requiredWidth -or $currentHeight -lt $requiredHeight) {
        Write-Host "Error: Console window is too small." -ForegroundColor Red
        Write-Host "Current size: ${currentWidth}x${currentHeight}" -ForegroundColor Yellow
        Write-Host "Required size: ${requiredWidth}x${requiredHeight}" -ForegroundColor Yellow
        Write-Host "`nPlease resize your terminal window and try again." -ForegroundColor Yellow
        return
    }

    $width = $requiredWidth
    $height = $requiredHeight

    function Convert-ToAzimuthalEquidistant {
        param (
            [double]$lon,
            [double]$lat
        )
        $lonRad = $lon * [Math]::PI / 180
        $latRad = $lat * [Math]::PI / 180
        $rho = [Math]::PI/2 + $latRad
        
        return @{
            X = $rho * [Math]::Sin($lonRad)
            Y = -$rho * [Math]::Cos($lonRad)
        }
    }

    # Create empty canvas
    $canvas = New-Object 'char[,]' $height, $width
    for ($y = 0; $y -lt $height; $y++) {
        for ($x = 0; $x -lt $width; $x++) {
            $canvas[$y, $x] = ' '
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
        Write-Host "Error reading KML file: $_" -ForegroundColor Red
        return
    }

    # Convert coordinates
    $points = $coordinates -split ' ' | Where-Object { $_ -ne '' } | ForEach-Object {
        $coords = $_ -split ','
        $projected = Convert-ToAzimuthalEquidistant -lon $coords[0] -lat $coords[1]
        [PSCustomObject]@{
            X = $projected.X
            Y = $projected.Y
        }
    }

    # Find bounds
    $minX = ($points | Measure-Object -Property X -Minimum).Minimum
    $maxX = ($points | Measure-Object -Property X -Maximum).Maximum
    $minY = ($points | Measure-Object -Property Y -Minimum).Minimum
    $maxY = ($points | Measure-Object -Property Y -Maximum).Maximum

    # Calculate scale with zoom out
    $rangeX = $maxX - $minX
    $rangeY = $maxY - $minY
    $scale = [Math]::Min($width / $rangeX, ($height * 2) / $rangeY)

    # Center point
    $centerX = ($minX + $maxX) / 2
    $centerY = ($minY + $maxY) / 2

    # Draw connected path
    for ($i = 0; $i -lt $points.Count - 1; $i++) {
        $x1 = [Math]::Floor($width/2 + ($points[$i].X - $centerX) * $scale)
        $y1 = [Math]::Floor($height/2 + ($points[$i].Y - $centerY) * $scale/2)
        $x2 = [Math]::Floor($width/2 + ($points[$i+1].X - $centerX) * $scale)
        $y2 = [Math]::Floor($height/2 + ($points[$i+1].Y - $centerY) * $scale/2)
        
        # Draw line between points
        $steps = [Math]::Max([Math]::Abs($x2 - $x1), [Math]::Abs($y2 - $y1))
        if ($steps -eq 0) { $steps = 1 }
        
        for ($step = 0; $step -le $steps; $step++) {
            $x = [Math]::Floor($x1 + ($x2 - $x1) * $step / $steps)
            $y = [Math]::Floor($y1 + ($y2 - $y1) * $step / $steps)
            
            if ($x -ge 0 -and $x -lt $width -and $y -ge 0 -and $y -lt $height) {
                $canvas[$y, $x] = '.'
            }
        }
    }

    # Output the path
    Write-Host "`nDisplaying path from $kml_file `n" -ForegroundColor Cyan
    for ($y = 0; $y -lt $height; $y++) {
        $line = ""
        for ($x = 0; $x -lt $width; $x++) {
            $line += $canvas[$y, $x]
        }
        Write-Host $line
    }
}

# kml_to_ascii "fritjolf-Path.kml"
```