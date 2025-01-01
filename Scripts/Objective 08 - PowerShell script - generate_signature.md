```
$access = "1"
$card_uuid = "c06018b6-5e80-4395-ab71-ae5124560189"
$hmac_secret = "9ed1515819dec61fd361d5fdabb57f41ecce1a5fe1fe263b98c0d6943b9b232e"

function Convert-HexToBytes {
    param(
        [string]$HexString
    )
    $Bytes = New-Object byte[] ($HexString.Length / 2)
    for($i=0; $i -lt $HexString.Length; $i+=2){
        $Bytes[$i/2] = [Convert]::ToByte($HexString.Substring($i, 2), 16)
    }
    return $Bytes
}

# Create the key as raw bytes (not hex-decoded)
$secret_key = [System.Text.Encoding]::ASCII.GetBytes($hmac_secret)

# Create message bytes
$access = [System.Text.Encoding]::ASCII.GetBytes($access)
$uuid = [System.Text.Encoding]::ASCII.GetBytes($card_uuid)

# Combine bytes
$hmac_message = New-Object byte[] ($access.Length + $uuid.Length)
[System.Buffer]::BlockCopy($access, 0, $hmac_message, 0, $access.Length)
[System.Buffer]::BlockCopy($uuid, 0, $hmac_message, $access.Length, $uuid.Length)

# Create HMAC
$hmac = New-Object System.Security.Cryptography.HMACSHA256
$hmac.Key = $secret_key
$signature = [System.BitConverter]::ToString($hmac.ComputeHash($hmac_message)).Replace("-", "").ToLower()

# Output
Write-Host "signature = $signature"

```