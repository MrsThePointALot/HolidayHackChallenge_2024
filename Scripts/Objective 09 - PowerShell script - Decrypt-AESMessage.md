```
function Decrypt-AESMessage {
    param(
        [Parameter(Mandatory=$true)]
        [string]$EncryptedBase64
    )
    
    function Convert-FromBase64 {
        param([string]$base64)
        return [Convert]::FromBase64String($base64)
    }

    function Decrypt-AES-CTR {
        param(
            [byte[]]$key,
            [byte[]]$iv,
            [byte[]]$data
        )
        
        $aes = [System.Security.Cryptography.Aes]::Create()
        $aes.Mode = [System.Security.Cryptography.CipherMode]::ECB
        $aes.Padding = [System.Security.Cryptography.PaddingMode]::None
        $aes.Key = $key
        
        $counter = $iv.Clone()
        $encryptor = $aes.CreateEncryptor()
        $plaintext = New-Object byte[] $data.Length
        
        for ($i = 0; $i -lt $data.Length; $i += 16) {
            $keystream = $encryptor.TransformFinalBlock($counter, 0, 16)
            $blockSize = [Math]::Min(16, $data.Length - $i)
            
            for ($j = 0; $j -lt $blockSize; $j++) {
                $plaintext[$i + $j] = $data[$i + $j] -bxor $keystream[$j]
            }
            
            # Increment counter
            for ($j = 15; $j -ge 0; $j--) {
                $counter[$j]++
                if ($counter[$j] -ne 0) { break }
            }
        }
        
        $encryptor.Dispose()
        $aes.Dispose()
        
        return $plaintext
    }

    try {
        # Known values from the original script
        $key = Convert-FromBase64 "rmDJ1wJ7ZtKy3lkLs6X9bZ2Jvpt6jL6YWiDsXtgjkXw="
        $iv = [byte[]]@(0x43,0x68,0x65,0x63,0x6B,0x4D,0x61,0x74,0x65,0x72,0x69,0x78,0x00,0x00,0x00,0x02)
        
        # Decode input ciphertext
        $ciphertext = Convert-FromBase64 $EncryptedBase64
        
        # Remove last 16 bytes (auth tag)
        $ciphertext = $ciphertext[0..($ciphertext.Length-17)]
        
        # Decrypt
        $plaintext = Decrypt-AES-CTR -key $key -iv $iv -data $ciphertext
        $result = [System.Text.Encoding]::UTF8.GetString($plaintext)
        
        return $result
    }
    catch {
        Write-Error "Decryption failed: $_"
        return $null
    }
}

$encrypted = "IVrt+9Zct4oUePZeQqFwyhBix8cSCIxtsa+lJZkMNpNFBgoHeJlwp73l2oyEh1Y6AfqnfH7gcU9Yfov6u70cUA2/OwcxVt7Ubdn0UD2kImNsclEQ9M8PpnevBX3mXlW2QnH8+Q+SC7JaMUc9CIvxB2HYQG2JujQf6skpVaPAKGxfLqDj+2UyTAVLoeUlQjc18swZVtTQO7Zwe6sTCYlrw7GpFXCAuI6Ex29gfeVIeB7pK7M4kZGy3OIaFxfTdevCoTMwkoPvJuRupA6ybp36vmLLMXaAWsrDHRUbKfE6UKvGoC9d5vqmKeIO9elASuagxjBJ"

$decrypted = Decrypt-AESMessage -EncryptedBase64 $encrypted
Write-Host $decrypted

```