```
function mqtt {
    param (
        [string]$serverString,
        [Parameter(Mandatory=$false)]
        [string]$user,
        [Parameter(Mandatory=$false)]
        [string]$pass
    )

    # Parse server and port from input string
    $server = "test.mosquitto.org"  # Default server
    $port = 1883                    # Default port

    if ($serverString) {
        if ($serverString -match '^([^:]+):(\d+)$') {
            $server = $matches[1]
            $port = [int]$matches[2]
        }
        elseif ($serverString -match '^([^:]+)$') {
            $server = $matches[1]
        }
    }

    # Track connection state
    $script:isConnected = $false

    # Track ignored topics
    $script:ignoredTopics = [System.Collections.ArrayList]@()

    # Store subscribed topics
    $script:subscribedTopics = @($topic)
    $script:currentTopic = $topic
    $script:ctrlCPressed = $false

    # Initialize the window buffer and tracking variables
    $script:windowBuffer = New-Object System.Collections.ArrayList
    $script:windowHeight = [Console]::WindowHeight
    $script:screenBufferLock = [System.Object]::new()
    $script:currentInput = ""
    $script:prevWindowWidth = [Console]::WindowWidth
    $script:prevWindowHeight = [Console]::WindowHeight

    function Show-Help {
        $helpMessage = "# MQTT Client Help"
        $helpMessage += "`n-----------------"
        $helpMessage += "`nCommands:"
        $helpMessage += "`n  sub <topic>    Subscribe to a topic"
        $helpMessage += "`n  unsub <topic>  Unsubscribe from a topic"
        $helpMessage += "`n  use <topic>    Switch to a different subscribed topic for sending messages"
        $helpMessage += "`n  help           Show this help message"
        $helpMessage += "`n"
        $helpMessage += "`nKeys:"
        $helpMessage += "`n  [Tab]          Cycle through subscribed topics"
        $helpMessage += "`n  [Enter]        Send message or execute command"
        $helpMessage += "`n  [Ctrl+C]       Exit the client"
        $helpMessage += "`n"
        $helpMessage += "`nExamples:"
        $helpMessage += "`n  sub test       Subscribe to 'test' topic"
        $helpMessage += "`n  use test       Switch to 'test' topic for sending"
        $helpMessage += "`n  unsub test     Unsubscribe from 'test' topic"
        $helpMessage += "`n"
        $helpMessage += "`nNote: Any other input will be sent as a message to the current topic."
        $helpMessage += "`n"
        $helpMessage += "`n-----------------"
        $helpMessage += "`n"
        
        foreach ($line in $helpMessage -split "`n") {
            Add-Message $line.TrimEnd()
        }
        Add-Message ""
    }

    function Render-Screen {
        $windowHeight = [Console]::WindowHeight
        $windowWidth = [Console]::WindowWidth
        if ($windowWidth -le 0 -or $windowHeight -le 0) { return }
        
        $promptPrefix = "mqtt ($script:currentTopic)> "
        
        try {
            # Instead of Clear-Host, we'll use more targeted clearing
            $clearLine = " " * $windowWidth
            
            # Draw messages
            $currentLine = $windowHeight - 3  # Changed from -2 to -3 to make room for separator
            foreach ($msg in $script:windowBuffer) {
                if ($currentLine -ge 0) {
                    [Console]::SetCursorPosition(0, $currentLine)
                    [Console]::Write($clearLine)  # Clear the line first
                    [Console]::SetCursorPosition(0, $currentLine)
                    if ($msg.Length -gt $windowWidth) {
                        [Console]::Write($msg.Substring(0, $windowWidth))
                    } else {
                        [Console]::Write($msg)
                    }
                    $currentLine--
                }
                if ($currentLine -lt 0) { break }
            }
            
            # Draw separator line
            [Console]::SetCursorPosition(0, $windowHeight - 2)
            [Console]::Write("-" * $windowWidth)
            
            # Draw prompt with truncated input
            $maxInputWidth = $windowWidth - $promptPrefix.Length - 1
            $displayInput = if ($script:currentInput.Length -gt $maxInputWidth) {
                "..." + $script:currentInput.Substring($script:currentInput.Length - ($maxInputWidth - 3))
            } else {
                $script:currentInput
            }
            
            # Clear and redraw prompt line
            [Console]::SetCursorPosition(0, $windowHeight - 1)
            [Console]::Write($clearLine)
            [Console]::SetCursorPosition(0, $windowHeight - 1)
            [Console]::Write($promptPrefix + $displayInput)
            
            # Position cursor correctly
            $cursorX = [Math]::Min($promptPrefix.Length + $displayInput.Length, $windowWidth - 1)
            [Console]::SetCursorPosition($cursorX, $windowHeight - 1)
        }
        catch {
            # Ignore render errors during resize
        }
    }

    function Add-Message {
        param([string]$message)
        
        [System.Threading.Monitor]::Enter($script:screenBufferLock)
        try {
            $windowWidth = [Console]::WindowWidth
            if ($windowWidth -le 0) { return }
            
            $lines = @()
    
            # Check if this is a system message
            if ($message.StartsWith("Connecting to") -or 
                $message.StartsWith("Using Client ID")) {
                $lines += $message
            }
            elseif ($message.StartsWith("Received CONNACK")) {
                $lines += "Server accepted our connection request"
            }
            elseif ($message.StartsWith("Received SUBACK")) {
                $lines += "Server accepted our subscription request"
            }
            elseif ($message.StartsWith("Sent (")) {
                $lines += $message
            }
            elseif ($message -match '^Sent \((.*?)\): (.*)$') {
                # For sent messages, use the actual topic from the message
                $topic = $matches[1]
                $content = $matches[2]
                $lines += "Sent ($topic): $content"
            }
            elseif ($message -match '^.*\((.*?)\):\s*(.*)$') {
                # Regular MQTT message - extract the original topic and content
                $originalTopic = $matches[1]
                $fullContent = $matches[2].TrimStart()
                
                # Only add the message if the topic is not in the ignored list
                if (-not $script:ignoredTopics.Contains($originalTopic)) {
                    $lines += "($originalTopic): $fullContent"
                }
            }
            else {
                # Regular message
                $lines += $message
            }
            
            # Add lines to buffer, filtering out empty lines
            foreach ($line in $lines) {
                if (-not [string]::IsNullOrWhiteSpace($line)) {
                    [void]$script:windowBuffer.Insert(0, $line.TrimEnd())
                }
            }
            
            # Keep only last windowHeight-1 messages
            while ($script:windowBuffer.Count -gt $script:windowHeight - 2) {
                $script:windowBuffer.RemoveAt($script:windowBuffer.Count - 1)
            }
            
            # Throttle screen updates
            $script:lastRender = if ($null -eq $script:lastRender) { [DateTime]::Now } else { $script:lastRender }
            if (([DateTime]::Now - $script:lastRender).TotalMilliseconds -ge 50) {  # Max 20 updates per second
                Render-Screen
                $script:lastRender = [DateTime]::Now
            }
        }
        finally {
            [System.Threading.Monitor]::Exit($script:screenBufferLock)
        }
    }

    # Set up CTRL-C trap
    trap [System.Management.Automation.PipelineStoppedException] {
        Write-Host "`nDisconnecting from MQTT...`n"
        Send-DisconnectPacket
        $script:ctrlCPressed = $true
        break
    }

    # Clear screen and show help
    Clear-Host
    Show-Help

    # Modify the connection logic
    if (-not $script:isConnected) {
        Add-Message "Connecting to $server`:$port..."
        $tcpClient = New-Object System.Net.Sockets.TcpClient
        $tcpClient.Connect($server, $port)
        $stream = $tcpClient.GetStream()
        $script:isConnected = $true
    }

    function Process-MqttPacket {
        param (
            [System.IO.Stream]$stream
        )
    
        try {
            # Verify stream is still valid
            if (-not $stream -or -not $stream.CanRead) {
                Add-Message "Stream is no longer valid"
                return $false > $null
            }
    
            # Use larger buffer for high-volume scenarios
            $buffer = New-Object byte[] 65535
            
            # Add timeout protection with longer interval
            $script:lastPacketTime = if ($null -eq $script:lastPacketTime) { [DateTime]::Now } else { $script:lastPacketTime }
            $timeSinceLastPacket = [DateTime]::Now.Subtract($script:lastPacketTime).TotalSeconds
            
            # Send PINGREQ every 45 seconds if no other packets
            if ($timeSinceLastPacket -gt 45 -and -not $script:ctrlCPressed) {
                try {
                    [byte[]]$pingPacket = @(0xC0, 0x00)  # PINGREQ packet
                    [void]$stream.Write($pingPacket, 0, $pingPacket.Length)
                    [void]$stream.Flush()
                    $script:lastPacketTime = [DateTime]::Now
                }
                catch {
                    # Ignore ping errors
                }
            }
            
            # Only consider connection stale after 90 seconds of no packets
            if ($timeSinceLastPacket -gt 90) {
                Add-Message "Connection lost, attempting to reconnect..."
                return $false > $null
            }
            
            # Rate limiting setup
            $script:messageCount = if ($null -eq $script:messageCount) { 0 } else { $script:messageCount }
            $script:lastRateReset = if ($null -eq $script:lastRateReset) { [DateTime]::Now } else { $script:lastRateReset }
            
            # Reset message counter every second
            $timeSinceReset = [DateTime]::Now.Subtract($script:lastRateReset).TotalSeconds
            if ($timeSinceReset -ge 1) {
                $script:messageCount = 0
                $script:lastRateReset = [DateTime]::Now
            }
    
            try {
                if ($stream.DataAvailable) {
                    $bytesRead = $stream.Read($buffer, 0, $buffer.Length)
                    if ($bytesRead -gt 0) {
                        $script:lastPacketTime = [DateTime]::Now
                        $packetType = $buffer[0] -band 0xF0
                        
                        # Get remaining length with improved validation
                        $pos = 1
                        $multiplier = 1
                        $remainingLength = 0
                        $maxLengthBytes = 4
                        
                        do {
                            if ($pos -ge $bytesRead -or $pos -ge $maxLengthBytes) { 
                                # Instead of showing error, just skip this packet
                                return $true > $null 
                            }
                            $encodedByte = $buffer[$pos]
                            $remainingLength += ($encodedByte -band 127) * $multiplier
                            if ($remainingLength -gt $buffer.Length) {
                                # Skip oversized packets silently
                                return $true > $null
                            }
                            $multiplier *= 128
                            $pos++
                        } while (($encodedByte -band 128) -ne 0)
                        
                        # Validate packet size silently
                        $totalLength = $pos + $remainingLength
                        if ($totalLength -gt $buffer.Length -or $totalLength -gt $bytesRead) {
                            return $true > $null
                        }
                        
                        if ($totalLength -le $bytesRead) {
                            switch ($packetType) {
                                0xE0 { # DISCONNECT
                                    if ($remainingLength -eq 0 -and -not $script:ctrlCPressed) {
                                        Add-Message "Server closed connection"
                                        $script:ctrlCPressed = $true
                                        $script:isConnected = $false
                                        return $false > $null
                                    }
                                    break
                                }
                                0x90 { # SUBACK
                                    Add-Message "Server accepted our subscription request"
                                }
                                0xB0 { # UNSUBACK
                                    Add-Message "Server accepted our unsubscription request"
                                }
                                0x20 { # CONNACK
                                    if (-not $script:isConnected) {
                                        Add-Message "Server accepted our connection request"
                                        $script:isConnected = $true
                                    }
                                }
                                0x40 { # PUBACK
                                    # Silently handle publish acknowledgments
                                }
                                0xD0 { # PINGRESP
                                    # Silently handle ping responses
                                }
                                0x30 { # PUBLISH
                                    try {
                                        # Rate limiting: max 100 messages per second
                                        if ($script:messageCount -gt 100) {
                                            continue
                                        }
                                        
                                        if ($bytesRead -lt 4) { continue }
                                        
                                        $topicLen = ($buffer[$pos] -shl 8) + $buffer[$pos+1]
                                        $pos += 2
                                        
                                        # Improved bounds checking
                                        if ($topicLen -le 0 -or $topicLen -gt 65535 -or 
                                            ($pos + $topicLen) -gt $bytesRead -or 
                                            ($pos + $topicLen + 1) -gt $totalLength) { 
                                            continue 
                                        }
                                        
                                        $receivedTopic = [System.Text.Encoding]::UTF8.GetString($buffer[$pos..($pos+$topicLen-1)])
                                        
                                        if ($receivedTopic -match '^[A-Za-z0-9/._#+-]+$') {
                                            $isWildcardMatch = $script:subscribedTopics -contains "#" -and 
                                                             -not $script:ignoredTopics.Contains($receivedTopic)
                                            
                                            if (-not $script:ignoredTopics.Contains($receivedTopic) -and
                                                ($isWildcardMatch -or $script:subscribedTopics -contains $receivedTopic)) {
                                                
                                                $pos += $topicLen
                                                if ($pos -lt $bytesRead -and $pos -lt $totalLength) {
                                                    $message = [System.Text.Encoding]::UTF8.GetString($buffer[$pos..($totalLength-1)])
                                                    if ($message -match '[\x20-\x7E\s]+') {
                                                        $script:messageCount++
                                                        Add-Message "($receivedTopic): $message"
                                                        
                                                        if ($script:messageCount -eq 100) {
                                                            Add-Message "Warning: High message volume - some messages will be skipped"
                                                        }
                                                    }
                                                }
                                            }
                                        }
                                    }
                                    catch {
                                        # Log the error but continue processing
                                        Add-Message "Error processing PUBLISH: $($_.Exception.Message)"
                                        continue
                                    }
                                }
                                default {
                                    continue
                                }
                            }
                        }
                        
                        # Ensure stream is still valid before flushing
                        if ($stream.CanWrite) {
                            try {
                                [void]$stream.Flush()
                            }
                            catch {
                                # Ignore flush errors
                            }
                        }
                    }
                }
                else {
                    Start-Sleep -Milliseconds 10
                    return $true > $null
                }
            }
            catch [System.IO.IOException] {
                Start-Sleep -Milliseconds 100
                return $true > $null
            }
            
            return $true > $null
        }
        catch {
            if (-not $script:ctrlCPressed) {
                Start-Sleep -Milliseconds 100
            }
            return $true > $null
        }
    }

    try {
        # Generate client ID
        $clientId = "MrsThePointALot_" + (Get-Random -Minimum 100 -Maximum 999)
        Add-Message "Using Client ID: $clientId"
        $clientIdBytes = [System.Text.Encoding]::UTF8.GetBytes($clientId)
    
        # Calculate CONNECT packet
        $connectFlags = 0x02  # Clean Session
    
        # Add authentication if provided
        $userBytes = $null
        $passBytes = $null
        if ($user) {
            $connectFlags = $connectFlags -bor 0x80  # Username flag
            $userBytes = [System.Text.Encoding]::UTF8.GetBytes($user)
            if ($pass) {
                $connectFlags = $connectFlags -bor 0x40  # Password flag
                $passBytes = [System.Text.Encoding]::UTF8.GetBytes($pass)
            }
        }
    
        # Build CONNECT packet
        [byte[]]$connectPacket = @(
            0x10,                    # Connect
            0x00,                    # Remaining Length (placeholder)
            0x00, 0x04,             # Protocol Name Length
            0x4D, 0x51, 0x54, 0x54, # "MQTT"
            0x05,                    # Protocol Level (5)
            [byte]$connectFlags,     # Connect Flags
            0x00, 0x3C,             # Keep Alive (60s)
            0x05,                    # Properties Length
            0x11, 0x00, 0x00, 0x00, 0x00,  # Session Expiry
            [byte](($clientIdBytes.Length -shr 8) -band 0xFF),  # Client ID Length MSB
            [byte]($clientIdBytes.Length -band 0xFF)            # Client ID Length LSB
        )
        $connectPacket += $clientIdBytes
    
        # Add username if provided
        if ($userBytes) {
            $connectPacket += [byte](($userBytes.Length -shr 8) -band 0xFF)  # Username Length MSB
            $connectPacket += [byte]($userBytes.Length -band 0xFF)           # Username Length LSB
            $connectPacket += $userBytes
    
            # Add password if provided
            if ($passBytes) {
                $connectPacket += [byte](($passBytes.Length -shr 8) -band 0xFF)  # Password Length MSB
                $connectPacket += [byte]($passBytes.Length -band 0xFF)           # Password Length LSB
                $connectPacket += $passBytes
            }
        }
    
        # Calculate and set remaining length
        $remainingLength = $connectPacket.Length - 2  # Subtract fixed header
        $connectPacket[1] = [byte]$remainingLength
    
        Add-Message "Connecting to $server`:$port..."
        if ($user) {
            Add-Message "Using username: $user"
        }
    
        $tcpClient = New-Object System.Net.Sockets.TcpClient
        $tcpClient.Connect($server, $port)
        $stream = $tcpClient.GetStream()
    
        # Send CONNECT packet
        $stream.Write($connectPacket, 0, $connectPacket.Length)
        $stream.Flush()

        # Wait for CONNACK
        $buffer = New-Object byte[] 1024
        $bytesRead = $stream.Read($buffer, 0, $buffer.Length)
        Add-Message "Received CONNACK: " + [System.Text.Encoding]::ASCII.GetString($buffer[0..($bytesRead-1)])
        
        Add-Message "Please subscribe to a topic using 'sub <topic>'"
        Add-Message "For example: sub test"
        Add-Message ""
        
        Start-Sleep -Milliseconds 100

        # Function to send SUBSCRIBE packet
        function Send-SubscribePacket {
            param([string]$topic)
            
            $topicBytes = [System.Text.Encoding]::UTF8.GetBytes($topic)
            $topicLength = $topicBytes.Length
            $remainingLength = 2 + 2 + 1 + $topicLength + 1  # Packet ID + Topic Length + Properties Length + Topic + QoS
            
            [byte[]]$subscribePacket = @(
                0x82,                          # Subscribe
                [byte]$remainingLength,        # Remaining Length
                0xF7, 0x78,                    # Packet ID (exact match)
                0x00,                          # Properties Length
                [byte](($topicLength -shr 8) -band 0xFF),  # Topic Length MSB
                [byte]($topicLength -band 0xFF)            # Topic Length LSB
            )
            
            $subscribePacket += $topicBytes    # Topic
            $subscribePacket += 0x00           # QoS 0
            
            $stream.Write($subscribePacket, 0, $subscribePacket.Length)
            $stream.Flush()
        }

        function Send-UnsubscribePacket {
            param([string]$topic)
            
            try {
                $topicBytes = [System.Text.Encoding]::UTF8.GetBytes($topic)
                $topicLength = $topicBytes.Length
                
                # Build UNSUBSCRIBE packet
                [byte[]]$unsubscribePacket = @(
                    0xA2,                    # Unsubscribe with QoS 1
                    0x00,                    # Remaining Length (placeholder)
                    0x00, 0x01,             # Packet ID
                    0x00,                    # Properties Length
                    [byte](($topicLength -shr 8) -band 0xFF),  # Topic Length MSB
                    [byte]($topicLength -band 0xFF)            # Topic Length LSB
                )
                
                $unsubscribePacket += $topicBytes
                
                # Calculate and set remaining length
                $remainingLength = $unsubscribePacket.Length - 2  # Subtract fixed header
                $unsubscribePacket[1] = [byte]$remainingLength
                
                # Clear any pending messages
                while ($stream.DataAvailable) {
                    $tempBuffer = New-Object byte[] 1024
                    $stream.Read($tempBuffer, 0, $tempBuffer.Length) | Out-Null
                }
                
                # Send the packet with error handling
                if ($stream.CanWrite) {
                    $stream.Write($unsubscribePacket, 0, $unsubscribePacket.Length)
                    $stream.Flush()
                    Start-Sleep -Milliseconds 100  # Give time for the server to process
                    return $true
                }
                return $false
            }
            catch {
                Add-Message "Error sending unsubscribe packet: $_"
                return $false
            }
        }
        
        function Send-DisconnectPacket {
            try {
                if (-not $script:ctrlCPressed) {
                    [byte[]]$disconnectPacket = @(
                        0xE0,  # DISCONNECT
                        0x00   # Remaining Length
                    )
                    
                    [void]$stream.Write($disconnectPacket, 0, $disconnectPacket.Length)
                    [void]$stream.Flush()
                    
                    Start-Sleep -Milliseconds 50
                }
                return $true > $null
            }
            catch {
                return $false > $null
            }
        }

        # Function to send PUBLISH packet
        function Send-PublishPacket {
            param([string]$topic, [string]$message)
            
            $topicBytes = [System.Text.Encoding]::UTF8.GetBytes($topic)
            $messageBytes = [System.Text.Encoding]::UTF8.GetBytes($message)
            $topicLength = $topicBytes.Length
            $remainingLength = 2 + $topicLength + 1 + $messageBytes.Length  # Topic Length + Topic + Separator + Message
            
            # Fixed header: PUBLISH (0x30) with QoS 0 and no flags
            [byte[]]$publishPacket = @(0x30)

            # Encode remaining length using MQTT variable length encoding
            while ($true) {
                $encodedByte = $remainingLength -band 0x7F
                $remainingLength = $remainingLength -shr 7
                if ($remainingLength -gt 0) {
                    $encodedByte = $encodedByte -bor 0x80
                }
                $publishPacket += [byte]$encodedByte
                if ($remainingLength -eq 0) {
                    break
                }
            }
            
            # Variable header: Topic Name
            $publishPacket += [byte](($topicLength -shr 8) -band 0xFF)  # Topic Length MSB
            $publishPacket += [byte]($topicLength -band 0xFF)           # Topic Length LSB
            $publishPacket += $topicBytes                               # Topic
            
            # Add separator
            $publishPacket += 0x00
            
            # Payload: Message
            $publishPacket += $messageBytes                             # Message
            
            # Send the packet
            $stream.Write($publishPacket, 0, $publishPacket.Length)
            $stream.Flush()
        }

        # Main loop
        while (-not $script:ctrlCPressed) {
            if ($stream.DataAvailable) {
                [void](Process-MqttPacket $stream)
            }

            if ([Console]::KeyAvailable) {
                $key = [Console]::ReadKey($true)
                
                [System.Threading.Monitor]::Enter($script:screenBufferLock)
                try {
                    switch ($key.Key) {
                        "Enter" {
                            $input = $script:currentInput.Trim()
                                
                            # Check if input is a command or a message
                            if ($input -match '^(sub|unsub|use|help)\s*') {
                                switch -Regex ($input) {
                                    '^help$' {
                                        Show-Help
                                    }
                                    '^sub\s+(.+)$' {
                                        $newTopic = $matches[1]
                                        if ($null -eq $script:subscribedTopics) {
                                            $script:subscribedTopics = @()  # Reinitialize if null
                                        }
                                        if (-not $script:subscribedTopics.Contains($newTopic)) {
                                            $script:subscribedTopics += $newTopic
                                            $script:currentTopic = $newTopic  # Set current topic to the newly subscribed one
                                            Add-Message "Subscribing to: $newTopic"
                                            Send-SubscribePacket $newTopic
                                        }
                                        else {
                                            Add-Message "Already subscribed to $newTopic"
                                        }
                                    }
                                    '^unsub\s+(.+)$' {
                                        $topic = $matches[1]
                                        if ($script:subscribedTopics.Contains($topic)) {
                                            try {
                                                # IMMEDIATELY add to ignored list
                                                [void]$script:ignoredTopics.Add($topic)
                                                Add-Message "Unsubscribing from $topic..."
                                                
                                                # Give a small delay for the ignored list to take effect
                                                Start-Sleep -Milliseconds 50
                                                
                                                if (Send-UnsubscribePacket $topic) {
                                                    # Create a new array without the unsubscribed topic
                                                    $script:subscribedTopics = @($script:subscribedTopics | Where-Object { $_ -ne $topic })
                                                    Add-Message "Unsubscribed from $topic"
                                                    
                                                    # Handle current topic change
                                                    if ($script:currentTopic -eq $topic) {
                                                        if ($script:subscribedTopics.Count -gt 0) {
                                                            # Set current topic to the first remaining subscribed topic
                                                            $script:currentTopic = $script:subscribedTopics[0]
                                                            Add-Message "Switched to topic: $script:currentTopic"
                                                        } else {
                                                            $script:currentTopic = ""
                                                            Add-Message "No topics subscribed. Please subscribe to a topic using 'sub <topic>'"
                                                        }
                                                    }
                                                }
                                                else {
                                                    Add-Message "Failed to unsubscribe from $topic"
                                                    # Remove from ignored list if unsubscribe failed
                                                    $script:ignoredTopics.Remove($topic)
                                                }
                                            }
                                            catch {
                                                Add-Message "Error during unsubscribe: $_"
                                                # Remove from ignored list if there was an error
                                                $script:ignoredTopics.Remove($topic)
                                            }
                                        }
                                        else {
                                            Add-Message "Not subscribed to $topic"
                                        }
                                    }
                                    '^use\s+(.+)$' {
                                        $topic = $matches[1]
                                        if ($script:subscribedTopics.Contains($topic)) {
                                            $script:currentTopic = $topic
                                            Add-Message "Now using $topic for sending messages"
                                        }
                                        else {
                                            Add-Message "Not subscribed to $topic"
                                        }
                                    }
                                }
                            } else {
                                if ($input -notmatch '^(sub|unsub|use|help)\s*') {
                                    # Treat as a message to send
                                    if ([string]::IsNullOrEmpty($script:currentTopic)) {
                                        Add-Message "No topic selected. Please subscribe to a topic first."
                                    } else {
                                        Send-PublishPacket -topic $script:currentTopic -message $input
                                        Add-Message "Sent ($script:currentTopic): $input"
                                    }
                                }
                            }
                            $script:currentInput = ""
                            Render-Screen
                        }
                        "Backspace" {
                            if ($script:currentInput.Length -gt 0) {
                                $script:currentInput = $script:currentInput.Substring(0, $script:currentInput.Length - 1)
                                Render-Screen
                            }
                        }
                        "Tab" {
                            # Only cycle through non-empty topics
                            $validTopics = @($script:subscribedTopics | Where-Object { -not [string]::IsNullOrEmpty($_) })
                            if ($validTopics.Count -gt 0) {
                                $currentIndex = $validTopics.IndexOf($script:currentTopic)
                                $nextIndex = ($currentIndex + 1) % $validTopics.Count
                                $script:currentTopic = $validTopics[$nextIndex]
                                Render-Screen
                            }
                        }
                        default {
                            if (-not [char]::IsControl($key.KeyChar)) {
                                $script:currentInput += $key.KeyChar
                                Render-Screen
                            }
                        }
                    }
                }
                finally {
                    [System.Threading.Monitor]::Exit($script:screenBufferLock)
                }
                continue  # Skip to next iteration after handling input
            }

            # Second priority: Check for window resize
            $currentWidth = [Console]::WindowWidth
            $currentHeight = [Console]::WindowHeight
            
            if ($currentWidth -ne $script:prevWindowWidth -or $currentHeight -ne $script:prevWindowHeight) {
                [System.Threading.Monitor]::Enter($script:screenBufferLock)
                try {
                    $script:windowHeight = $currentHeight
                    Clear-Host  # Clear the screen to prevent artifacts
                    
                    # Store original messages
                    $originalMessages = @()
                    foreach ($message in $script:windowBuffer) {
                        $originalMessages += $message
                    }
                    
                    # Clear and rebuild buffer
                    $script:windowBuffer.Clear()
                    foreach ($message in $originalMessages) {
                        Add-Message $message
                    }
                    
                    $script:prevWindowWidth = $currentWidth
                    $script:prevWindowHeight = $currentHeight
                }
                finally {
                    [System.Threading.Monitor]::Exit($script:screenBufferLock)
                }
                continue  # Skip to next iteration after handling resize
            }

            
            if ($stream.DataAvailable) {
                Process-MqttPacket $stream
            }
            
            # Small sleep if no data to prevent CPU spinning
            if (-not [Console]::KeyAvailable -and -not $stream.DataAvailable) {
                Start-Sleep -Milliseconds 1
            }
        }
    }
    finally {
        if ($stream) {
            Send-DisconnectPacket
            $stream.Close()
        }
        if ($tcpClient) {
            $tcpClient.Close()
        }
        Write-Host "`nDisconnected from MQTT"
    }
}

# mqtt test.mosquitto.org:1884 -u rw -p readwrite

```