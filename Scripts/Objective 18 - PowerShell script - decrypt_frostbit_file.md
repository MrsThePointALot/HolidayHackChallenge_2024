```
function decrypt_frostbit_file {
    param (
        [string]$path,
        [string]$key
    )

    $encryptedData = [System.IO.File]::ReadAllBytes($path)
    $iv = $encryptedData[0..15]
    $key_bytes = @()
    foreach ($char in $key.ToCharArray()) {
        $key_bytes += [byte][char]$char
    }
    $aes = [System.Security.Cryptography.Aes]::Create()
    $aes.KeySize = 256
    $aes.BlockSize = 128
    $aes.Mode = [System.Security.Cryptography.CipherMode]::CBC
    $aes.Padding = [System.Security.Cryptography.PaddingMode]::PKCS7
    $aes.Key = $key_bytes
    $aes.IV = $iv
    $decryptor = $aes.CreateDecryptor()

    try {
        $decryptedData = $decryptor.TransformFinalBlock($encryptedData[16..($encryptedData.Length - 1)], 0, $encryptedData.Length - 16)
        # Output the decrypted data (you can also save it to a file)
        [System.IO.File]::WriteAllBytes("$path.decrypted", $decryptedData)
        Write-Host "Decrypted file saved as '$path.decrypted'."
    } catch {
        Write-Host "Decryption failed: $_"
    }
}

decrypt_frostbit_file -path "$HOME\Downloads\frostbitartifacts\naughty_nice_list.csv.frostbit" -key 'f822b94a4ffbe1c32f7655a0858944ce'

```