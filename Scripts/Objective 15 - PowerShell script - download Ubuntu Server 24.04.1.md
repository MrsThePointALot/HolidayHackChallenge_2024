```
$f = "ubuntu-24.04.1-live-server-amd64.iso"
$URL="https://releases.ubuntu.com/24.04.1/$f"
$Path="$HOME\Downloads\$f"

$ProgressPreference = 0  # greatly improves download speed
Invoke-Webrequest $URL -Outfile $Path

```