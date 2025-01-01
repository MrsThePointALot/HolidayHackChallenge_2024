```
function browse {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true, Position=0)]
        [string]$Url
    )

    # Add https:// if no protocol specified
    if (-not $Url.StartsWith("http://") -and -not $Url.StartsWith("https://")) {
        $Url = "https://$Url"
    }

    try {
        # Download the webpage content
        $ProgressPreference = 'SilentlyContinue'
        $response = Invoke-WebRequest -Uri $Url -UseBasicParsing
        
        # Display the raw content
        return $response.Content
    }
    catch {
        Write-Host "Failed to download $Url. Error: $_" -ForegroundColor Red
    }
}

# browse http://www.example.com/

```