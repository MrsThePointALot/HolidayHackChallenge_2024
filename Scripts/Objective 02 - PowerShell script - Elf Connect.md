```
# Define word sets as hashtables
$wordSets = @{
    1 = @("Tinsel", "Sleigh", "Belafonte", "Bag", "Comet", "Garland", "Jingle Bells", "Mittens", "Vixen", "Gifts", "Star", "Crosby", "White Christmas", "Prancer", "Lights", "Blitzen")
    2 = @("Nmap", "burp", "Frida", "OWASP Zap", "Metasploit", "netcat", "Cycript", "Nikto", "Cobalt Strike", "wfuzz", "Wireshark", "AppMon", "apktool", "HAVOC", "Nessus", "Empire") 
    3 = @("AES", "WEP", "Symmetric", "WPA2", "Caesar", "RSA", "Asymmetric", "TKIP", "One-time Pad", "LEAP", "Blowfish", "hash", "hybrid", "Ottendorf", "3DES", "Scytale")
    4 = @("IGMP", "TLS", "Ethernet", "SSL", "HTTP", "IPX", "PPP", "IPSec", "FTP", "SSH", "IP", "IEEE 802.11", "ARP", "SMTP", "ICMP", "DNS")
}

# Define correct sets as array of arrays
$correctSets = @(
    @(0, 5, 10, 14),  # Set 1
    @(1, 3, 7, 9),    # Set 2 
    @(2, 6, 11, 12),  # Set 3
    @(4, 8, 13, 15)   # Set 4
)

function Display-AllSets {
    # Loop through each word set
    for($setNum = 1; $setNum -le 4; $setNum++) {
        Write-Host "`nWord Set $($setNum):"
        
        # Get all sets that use words from this wordSet
        $setGroups = @()
        for($i = 0; $i -lt 4; $i++) {
            $indices = $correctSets[$i]
            $words = $indices | ForEach-Object { $wordSets[$setNum][$_] }
            $setGroups += ,$words
        }
        
        # Display each group
        for($i = 0; $i -lt $setGroups.Count; $i++) {
            $group = $setGroups[$i]
            Write-Host "Group $($i + 1): $($group -join ", ")"
        }
    }
}

# Call the function
Display-AllSets
```