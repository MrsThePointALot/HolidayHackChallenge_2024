```
# create a working directory
$dir = "$HOME\Desktop\HHC2024\"; mkdir $dir -ea 0; cd $dir

# download shreds.zip
wget https://holidayhackchallenge.com/2024/shreds.zip -Outfile shreds.zip

# unzip shreds.zip
Expand-Archive -ea 0 shreds.zip

# magic time
Add-Type -AssemblyName System.Drawing

function Get-ExifUserComment {
    param ([string]$imagePath)
    try {
        $image = [System.Drawing.Image]::FromFile($imagePath)
        foreach ($prop in $image.PropertyItems) {
            if ($prop.Id -eq 37510) {
                return [System.Text.Encoding]::Unicode.GetString($prop.Value).Trim([char]0)
            }
        }
    } finally {
        if ($image) { $image.Dispose() }
    }
}

function Decode-MetadataComment {
    param ([string]$base64Comment)
    try {
        $decoded = [Text.Encoding]::Utf8.GetString([Convert]::FromBase64String($base64Comment))
        if ($decoded -match '^\["([^"]+)",\s*"([^"]+)"\]$') {
            $orderPartReversed = -join $matches[1][$matches[1].Length..0]
            $orderNumber = [Text.Encoding]::Utf8.GetString([Convert]::FromBase64String($orderPartReversed))
            return @{
                Order = [int]$orderNumber
                Text = $matches[2]
            }
        }
    } catch {
        Write-Host "Error decoding metadata: $_"
    }
}

function Convert-UnicodeEscapes {
    param ([string]$text)
    
    if ($text -match '\\u[0-9a-fA-F]{4}') {
        $decoded = [Regex]::Replace(
            $text,
            '\\u([0-9a-fA-F]{4})',
            { param($m) [char][int]('0x' + $m.Groups[1].Value) }
        )
        return $decoded
    }
    return $text
}

$imageData = @()
Get-ChildItem -Path ".\shreds\slices\" -Filter "*.jpg" | ForEach-Object {
    $comment = Get-ExifUserComment -imagePath $_.FullName
    if ($comment) {
        $metadata = Decode-MetadataComment -base64Comment $comment
        if ($metadata) {
            $imageData += @{
                Path = $_.FullName
                Order = $metadata.Order
                Text = $metadata.Text
            }
        }
    }
}

if ($imageData.Count -gt 0) {
    $sortedImages = $imageData | Sort-Object { [int]$_.Order }
    $images = @()
    $finalImage = $null
    $graphics = $null
    
    try {
        $totalWidth = 0
        $height = 0
        
        foreach ($imgData in $sortedImages) {
            $img = [System.Drawing.Image]::FromFile($imgData.Path)
            $images += $img
            $totalWidth += $img.Width
            $height = if ($height -eq 0) { $img.Height } else { $height }
        }
        
        $finalImage = New-Object System.Drawing.Bitmap([int]$totalWidth, [int]$height)
        $graphics = [System.Drawing.Graphics]::FromImage($finalImage)
        $graphics.InterpolationMode = [System.Drawing.Drawing2D.InterpolationMode]::HighQualityBicubic
        
        $currentX = 0
        foreach ($img in $images) {
            $graphics.DrawImage($img, $currentX, 0, $img.Width, $img.Height)
            $currentX += $img.Width
        }
        
        $outputPath = Join-Path $PWD.Path "combined.jpg"
        $jpegCodec = [System.Drawing.Imaging.ImageCodecInfo]::GetImageEncoders() | 
            Where-Object { $_.MimeType -eq 'image/jpeg' }
        $encoderParams = New-Object System.Drawing.Imaging.EncoderParameters(1)
        $encoderParams.Param[0] = New-Object System.Drawing.Imaging.EncoderParameter(
            [System.Drawing.Imaging.Encoder]::Quality, 100L)
        
        $finalImage.Save($outputPath, $jpegCodec, $encoderParams)
        Write-Host "Combined image saved to: $outputPath"
        
        $finalText = ($sortedImages | ForEach-Object { $_.Text }) -join ''
        $finalText = Convert-UnicodeEscapes $finalText
        Write-Host "`nDecoded text in order:"
        Write-Host $finalText
        
    } finally {
        if ($graphics) { $graphics.Dispose() }
        if ($finalImage) { $finalImage.Dispose() }
        foreach ($img in $images) { if ($img) { $img.Dispose() } }
    }
}

```