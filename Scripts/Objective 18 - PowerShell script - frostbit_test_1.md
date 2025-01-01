```
function frostbit_test_1 {
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
        (Invoke-WebRequest "https://34.173.241.47/view/$path/$uuid/status?digest=$digest&debug=1").Content
        
    } catch {
        Write-Host "Error:"
        Write-Host $_.ErrorDetails.Message
    }
}

# frostbit_test_1 -uuid 'c013b20d-c25c-4f58-97b5-2d114f8476fb' -path 'ZJAs0rHpbZUbWeK4jTmQ' -digest '00c080500580c8081080a55007004008'

```