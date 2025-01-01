```
$ip = "34.72.26.140"
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
$helper_user = $r.Headers.BrkrUser
$helper_pass = $r.Headers.BrkrPswd

# Activate Gold images in feed 'northpolefeeds':
$connected = iwr "http://$ip`:8000/mqtt?clientConnect=santashelper2024" -WebSession $s -Headers $h

# If everything goes well, you'll see:
# {"connect":"Connected"}
$connected.Content

```