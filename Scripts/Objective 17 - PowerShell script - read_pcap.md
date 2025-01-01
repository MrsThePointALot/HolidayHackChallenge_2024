```
function read_pcap ($pcap_file) {
	$fs = [System.IO.File]::Open($pcap_file, 'Open', 'Read')
	$globalHeader = New-Object byte[] 24
	$fs.Read($globalHeader, 0, 24)
	$buffer = New-Object byte[] 65536
    $fs.Read($buffer, 0, $buffer.Length)
	$ipHeader = $buffer[30..49]
	$srcIp = [string]::Format("{0}.{1}.{2}.{3}", $ipHeader[12], $ipHeader[13], $ipHeader[14], $ipHeader[15])
	$dstIp = [string]::Format("{0}.{1}.{2}.{3}", $ipHeader[16], $ipHeader[17], $ipHeader[18], $ipHeader[19])
	Write-Host "Source IP: $srcIp, Destination IP: $dstIp"
			
	$fs.Close()
}

# read_pcap "$HOME\Downloads\frostbitartifacts\ransomware_traffic.pcap"

```