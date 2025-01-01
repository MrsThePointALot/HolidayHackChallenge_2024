```
function deactivate ($uuid) {
  [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
  [System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
  $client = New-Object System.Net.WebClient
  $client.Headers.Add("x-api-key", "abe7a6ad-715e-4e6a-901b-c9279a964f91")
  try {      $client.DownloadString("https://34.173.241.47/api/v1/frostbitadmin/bot/$uuid/deactivate?debug=1")
  } catch [System.Net.WebException] {
      $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream())
      Write-Host $reader.ReadToEnd()
  }
}

# deactivate c013b20d-c25c-4f58-97b5-2d114f8476fb

```