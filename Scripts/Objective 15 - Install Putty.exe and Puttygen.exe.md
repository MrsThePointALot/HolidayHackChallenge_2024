```
$url = "https://the.earth.li/~sgtatham/putty/latest/w64/"
$ProgressPreference = 0  # greatly improves download speed
Invoke-Webrequest "$($url)putty.exe" -Outfile $HOME\Downloads\putty.exe
Invoke-Webrequest "$($url)puttygen.exe" -Outfile $HOME\Downloads\puttygen.exe

```