```
function csv_to_maptiles ($csv_filename) {
	# Required for image manipulation
	Add-Type -AssemblyName System.Drawing

	# Read the CSV file
	$csvData = Import-Csv -Path $csv_filename

	# Create output directory if it doesn't exist
	$outputDir = Join-Path $PWD "map_squares"
	if (-not (Test-Path $outputDir)) {
		New-Item -ItemType Directory -Path $outputDir -Force
	}

	# Parameters for the map
	$zoom = 16          # Zoom level for detail
	$width = 256        # Standard tile size
	$height = 256       # Standard tile size
	$radius = 400       # Circle radius in meters
	$gridSize = 5       # Number of tiles in each direction (e.g., 5 means 5x5 grid)
	$halfGrid = [Math]::Floor($gridSize / 2)  # Used for centering

	function Get-TileNumber($lat, $lon, $zoom) {
		$n = [Math]::Pow(2, $zoom)
		$x = [Math]::Floor(($lon + 180.0) / 360.0 * $n)
		$y = [Math]::Floor((1.0 - [Math]::Log([Math]::Tan($lat * [Math]::PI / 180.0) + 1.0 / [Math]::Cos($lat * [Math]::PI / 180.0)) / [Math]::PI) / 2.0 * $n)
		return @($x, $y)
	}

	function Get-PixelCoordinate($lat, $lon, $centerTileX, $centerTileY, $zoom) {
		$n = [Math]::Pow(2, $zoom)
		
		# Calculate global pixel coordinates
		$globalX = (($lon + 180) / 360) * $n * $width
		$latRad = $lat * [Math]::PI / 180
		$globalY = (1 - ([Math]::Log([Math]::Tan($latRad) + 1 / [Math]::Cos($latRad))) / [Math]::PI) / 2 * $n * $height
		
		# Calculate the pixel coordinates relative to the center tile
		$centerTilePixelX = $centerTileX * $width
		$centerTilePixelY = $centerTileY * $height
		
		# Calculate local coordinates within our grid
		$localX = $globalX - ($centerTileX - $halfGrid) * $width
		$localY = $globalY - ($centerTileY - $halfGrid) * $height
		
		return @([int]$localX, [int]$localY)
	}

	function Get-MetersPerPixel($lat, $zoom) {
		return (156543.03392 * [Math]::Cos($lat * [Math]::PI / 180) / [Math]::Pow(2, $zoom))
	}

	# Process each coordinate
	$index = 0
	foreach ($point in $csvData) {
		try {
			# Convert coordinates to double and ensure they're valid
			$lat = [double]($point.'OSD.latitude' -replace '[^0-9.-]', '')
			$lon = [double]($point.'OSD.longitude' -replace '[^0-9.-]', '')
			
			Write-Host "Processing point $($index + 1) - Lat: $lat, Lon: $lon"
			
			# Get center tile coordinates
			$tileCoords = Get-TileNumber $lat $lon $zoom
			$centerX = $tileCoords[0]
			$centerY = $tileCoords[1]
			
			# Create a bitmap to store the combined image
			$combinedBitmap = New-Object System.Drawing.Bitmap -ArgumentList ($gridSize * $width), ($gridSize * $height)
			$graphics = [System.Drawing.Graphics]::FromImage($combinedBitmap)
			
			# Download and draw tiles
			for ($offsetY = -$halfGrid; $offsetY -lt ($gridSize - $halfGrid); $offsetY++) {
				for ($offsetX = -$halfGrid; $offsetX -lt ($gridSize - $halfGrid); $offsetX++) {
					$x = $centerX + $offsetX
					$y = $centerY + $offsetY
					
					$url = "https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/$zoom/$y/$x"
					$tempFile = [System.IO.Path]::GetTempFileName()
					
					$headers = @{
						"User-Agent" = "PowerShell/7.0 (Windows NT 10.0; Win64; x64) Custom/1.0"
						"Referer" = "https://www.openstreetmap.org/"
					}
					
					try {
						$ProgressPreference = 'SilentlyContinue'
						Invoke-WebRequest -Uri $url -OutFile $tempFile -Headers $headers
						
						$tile = [System.Drawing.Image]::FromFile($tempFile)
						$graphics.DrawImage($tile, 
							($offsetX + $halfGrid) * $width, 
							($offsetY + $halfGrid) * $height, 
							$width, $height)
						$tile.Dispose()
						Remove-Item $tempFile -Force
					}
					catch {
						Write-Host "Error downloading/processing tile: $_"
					}
					
					Start-Sleep -Milliseconds 500
				}
			}
			
			# Calculate pixel coordinates for the center point
			$pixelCoords = Get-PixelCoordinate $lat $lon $centerX $centerY $zoom
			$centerPixelX = $pixelCoords[0]
			$centerPixelY = $pixelCoords[1]
			
			# Create pen for circle (2 pixels wide, red)
			$pen = New-Object System.Drawing.Pen ([System.Drawing.Color]::Red), 2
			
			# Calculate circle radius in pixels
			$metersPerPixel = Get-MetersPerPixel $lat $zoom
			$radiusPixels = [int]($radius / $metersPerPixel)
			
			# Draw the circle
			$graphics.DrawEllipse($pen, 
				$centerPixelX - $radiusPixels, 
				$centerPixelY - $radiusPixels, 
				2 * $radiusPixels, 
				2 * $radiusPixels)
			
			# Save combined image using a memory stream
			$outputFile = Join-Path $outputDir "map_point_$($index.ToString('000')).jpg"
			Write-Host "Saving to: $outputFile"
			
			$memoryStream = New-Object System.IO.MemoryStream
			try {
				$combinedBitmap.Save($memoryStream, [System.Drawing.Imaging.ImageFormat]::Jpeg)
				[System.IO.File]::WriteAllBytes($outputFile, $memoryStream.ToArray())
			}
			finally {
				$memoryStream.Dispose()
			}
			
			Write-Host "Successfully saved point $($index + 1)"
		}
		catch {
			Write-Host "Error processing point $($index + 1): $_"
		}
		finally {
			if ($pen) { $pen.Dispose() }
			if ($graphics) { $graphics.Dispose() }
			if ($combinedBitmap) { $combinedBitmap.Dispose() }
		}
		
		$index++
	}

	Write-Host "Completed downloading $index map images to $outputDir"
}

# csv_to_maptiles "Preparations-drone-name.csv"

```