```
function start_install  {
$iso="$HOME\Downloads\ubuntu-24.04.1-live-server-amd64.iso"
$vmname="ubuntu2404"
$vmip="192.168.1.242/24"
$vmgateway="192.168.1.1"
$vmdns="[192.168.1.1]"
$user="test"
$pass="test"
$folder="E:\Virtualbox"
$packages="['cowsay','rolldice']"

# Install ShaCrypt which we use to generate a sha512-crypt ($6$) password
Install-PackageProvider NuGet -Force -ea 0
Set-PSRepository PSGallery -InstallationPolicy Trusted -ea 0
Install-Module -Name ShaCrypt -Repository PSGallery -ea 0
Set-PSRepository PSGallery -InstallationPolicy Untrusted -ea 0

# Generate the password hash
$sha512crypthash = New-ShaPassword -Password $pass -Sha512 -Salt 05g7td3MAd.KP
$passbytes = [IO.MemoryStream]::new([byte[]][char[]]$pass)
$passhash = (Get-FileHash -InputStream $passbytes -Algorithm SHA512).Hash.ToLower()

# VirtualBox settings
$hostip = (Get-NetIPConfiguration | Where-Object {$_.IPv4DefaultGateway -ne $null -and $_.NetAdapter.Status -ne "Disconnected"}).IPv4Address.IPAddress
$adapter=(get-netadapter Ethernet).InterfaceDescription
new-alias vbm 'C:\Program Files\Oracle\VirtualBox\VBoxManage.exe' -ea 0
vbm controlvm $vmname poweroff *>$null
while($LastExitCode -eq 0){vbm list runningvms|findstr $vmname|Out-Null;echo 'Waiting for VM to shut down';sleep 2};echo 'VM shut down complete';
vbm unregistervm --delete $vmname *>$null
rm -force -rec $folder\$vmname -ea 0
#vbm list ostypes | Select-String -Pattern "Ubuntu"
vbm createvm --basefolder $folder\$vmname --name $vmname --ostype Ubuntu_64 --register
vbm modifyvm $vmname --cpus 1 --memory 8192 --vram 128 --usbxhci on --graphicscontroller vboxsvga --nic1 bridged --mouse usbtablet --audio none --pae off --firmware bios --bridgeadapter1 $adapter --clipboard-mode=bidirectional
vbm storagectl $vmname --name "SATA" --add sata --controller IntelAHCI --portcount 3 --bootable on --hostiocache off
vbm createhd --filename $folder\$vmname\$vmname.vdi --size 99999 --variant Standard
vbm storageattach $vmname --storagectl "SATA" --port 0 --device 0 --type hdd --medium $folder\$vmname\$vmname.vdi
vbm storageattach $vmname --storagectl "SATA" --port 2  --device 0 --type dvddrive --medium $iso

# Start the VM unattended installation
vbm unattended install $vmname --iso=$iso --extra-install-kernel-parameters="autoinstall cloud-config-url=/dev/null ds='nocloud-net;s=http://$hostip`:8080/'" --start-vm=gui

# Create user-data file which contains the unattended configuration
$filename = 'user-data';
$content = @"
#cloud-config
autoinstall:
  version: 1
  network:
    network:
      version: 2
      ethernets:
        enp0s3:
          addresses:
            - $vmip
          gateway4: $vmgateway
          nameservers: 
            addresses: $vmdns
  identity:
    hostname: $vmname
    username: $user
    password: $sha512crypthash
  source:
    id: ubuntu-server-minimal
  packages:
    $packages
  ssh:
    install-server: true
    allow-pw: false
  late-commands:
    - echo "PubKeyAuthentication yes" >> /target/etc/ssh/sshd_config
    - echo "PubkeyAcceptedAlgorithms +ssh-rsa" >> /target/etc/ssh/sshd_config
    - systemctl restart ssh
    - curl -m 1 -X BOOYAH http://$hostip`:8080
  user-data:
    users:
      - name: mrsthepointalot
        plain_text_passwd: StillBetterThanTo_MissThePointAllTheTime
        sudo: ALL=(ALL) NOPASSWD:ALL
        shell: /bin/bash
        lock_passwd: true
        ssh_authorized_keys:
          - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCFmefihjqWF5ub8KqdNHzzYuiTeiNb25d65mY0glTd2P64gVds4Qk9rl5W88cFCrcgBaL6MKmgaAagHbXc4CR03f5aJrTv62gf5axQ8K38K8aVVv3y3iTtYygY8p20T0pq38WlCLlTA9chFs4vYBB0L9QtrPq6xgOCHasSDz/Mngbsg9WKGGBfMRYMey82lQ7MOcQQi10rU4yoNK9FGyvNxYCzw8AeUTE1lxWBbjNq7pW8q/V61EehHeeZv0I6n8LIyCysFTQdnzTbIrbKrFAwFfT2AlMNsrPZBoQqw5CPF44i0DPhhv8DdUlHZRMHyDXwwChCT0nsuXR3nefrfutN rsa-key-20241221-MrsThePointALot
"@;
# We use IO.File because we want the file to be UTF8 and without BOM
[IO.File]::WriteAllLines($filename, $content);

# Create empty meta-data and vendor-data file
rm meta-data -ea 0; ni meta-data -ea 0
rm vendor-data -ea 0; ni vendor-data -ea 0

# Allow inbound connections on Windows machine:
Get-NetConnectionProfile -NetworkCategory Public -ea 0|%{Set-NetConnectionProfile -NetworkCategory Private -ea 0}

# Allow 8080 through firewall
New-NetFirewallRule -DisplayName 'ubuntu_unattended_install' -LocalPort 8080 -Action Allow -Protocol TCP -Direction Inbound

# Start a tiny HTTP server to send the user-data configuration file to Ubuntu
$http = [System.Net.HttpListener]::new()
$http.Prefixes.Add("http://+:8080/")
$http.Start()
Add-Type -AssemblyName System.Web

# auto-install does 3 GET, followed by a BOOYAH request when installation finishes
$counter = 4;
$endpoints = @('vendor-data','meta-data','user-data')
while($counter -gt 0) {
  $context = $http.GetContext();
  $request = $context.Request;
  $get_request = $request.HttpMethod -eq 'GET';
  $booyah_request = $request.HttpMethod -eq 'BOOYAH';
  $valid_endpoint = $null -ne ($endpoints|?{$request.Url -match $_});
  if ($get_request -and $valid_endpoint) {
    $counter--;
    $URL = $context.Request.Url.LocalPath.Trim('/')
    echo "Remote request for $URL - sending $URL";
    $ContentStream = [System.IO.File]::OpenRead("$PWD\$URL");
    $Context.Response.ContentType = [System.Web.MimeMapping]::GetMimeMapping("$PWD\$URL");
    $ContentStream.CopyTo( $Context.Response.OutputStream );
    $Context.Response.Close();
    $ContentStream.Close();
  }
  if ($booyah_request) {
    $counter--;
    echo "Guest indicates it finished installation and is about to reboot"
    echo "Shutting down HTTP server";
    $Context.Response.Close();
  }
};
$http.Close()
echo "HTTP server closed"

# Change back 'Private' to 'Public' type network
Get-NetConnectionProfile -NetworkCategory Private|%{Set-NetConnectionProfile -NetworkCategory Public}

# Remove 8080 from firewall
Remove-NetFirewallRule -DisplayName "ubuntu_unattended_install"

echo 'Installation complete'
echo "VM IP: $vmip"
}

$currentPrincipal = New-Object Security.Principal.WindowsPrincipal([Security.Principal.WindowsIdentity]::GetCurrent())
if ($currentPrincipal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {start_install} else { Echo "Please run this script with Administrative privileges." }

```