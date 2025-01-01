
```
function websocket {
    param (
        [Parameter()]
        [string]$c,

        [Parameter(DontShow)]
        [array]$filterCategories = @()
    )
    
    # Parse the filter categories if provided
    if ($c) {
        $filterCategories = $c.Split(',') | ForEach-Object { $_.Trim() }
        Write-Host "Filtering messages with categories: $($filterCategories -join ', ')" -ForegroundColor Cyan
    }

    $script:wsTypes = {
        [System.Net.WebSockets.ClientWebSocket]
        [System.Threading.CancellationTokenSource]
        [System.Threading.Tasks.Task]
        [System.Collections.Concurrent.ConcurrentQueue[string]]
        [System.Management.Automation.PsBreakException]
    }.GetNewClosure()

    $prevWindowHeight = [Console]::WindowHeight
    $prevWindowWidth = [Console]::WindowWidth

    # Initialize message arrays
    $script:Window0Messages = [System.Collections.ArrayList]::new()
    $script:Window1Messages = [System.Collections.ArrayList]::new()

    # Configuration
    $script:ws_url = "wss://2024.holidayhackchallenge.com/ws"
    $script:login_user = [System.Environment]::GetEnvironmentVariable('HHC_USER','User')
    $script:login_pass = [System.Environment]::GetEnvironmentVariable('HHC_PASS','User')

    if ([string]::IsNullOrEmpty($login_user) -or [string]::IsNullOrEmpty($login_pass)) {
        Write-Host " "
        Write-Host "User and/or Pass not found in environment." -ForegroundColor Yellow
        Write-Host "Please set your user and pass using the following script:" -ForegroundColor Yellow
        Write-Host " "
        Write-Host "function u(`$a, `$b) {[System.Environment]::SetEnvironmentVariable(`$a,`$b,'User')}" -ForegroundColor Cyan
        Write-Host "u 'HHC_USER' 'your_username'" -ForegroundColor Cyan
        Write-Host "u 'HHC_PASS' 'your_password'" -ForegroundColor Cyan
        Write-Host " "
        Write-Host " "
        Write-Host "websocket -c 'category1,category2,category3' to filter messages by category" -ForegroundColor Cyan
        Write-Host " "
        return
    }

    # Initialize screen buffer
    $script:screenBuffer = @(
        @{  # Top window
            Messages = [System.Collections.ArrayList]::new()
            Input = ""
        },
        @{  # Bottom window
            Messages = [System.Collections.ArrayList]::new()
            Input = ""
        }
    )

    # Locks
    $script:screenBufferLock = [System.Object]::new()
    $script:renderLock = [System.Object]::new()
    $script:switchLock = [System.Object]::new()
    $script:messageLock = [System.Object]::new()

    $script:messageQueue = [System.Collections.Concurrent.ConcurrentQueue[object]]::new()
    $script:maxMessages = 100  # Maximum messages to keep in history

    # Session management
    $script:sessions = @{
        Active = 0  # 0 for top window, 1 for bottom
        Windows = @(
            @{
                WS = $null
                Sync = $null
                Input = ""
                RunspaceHandle = $null
                ReceivePS = $null
                CancelSource = $null
                URI = ""
                History = [System.Collections.ArrayList]::new()
                HistoryIndex = -1
            },
            @{
                WS = $null
                Sync = $null
                Input = ""
                RunspaceHandle = $null
                ReceivePS = $null
                CancelSource = $null
                URI = ""
                History = [System.Collections.ArrayList]::new()
                HistoryIndex = -1
            }
        )
    }

    # Register CTRL+C handler
    $null = [Console]::TreatControlCAsInput = $false
    $global:ctrlCPressed = $false

    trap {
        if ($_.Exception.GetType().FullName -eq 'System.Management.Automation.PsBreakException') {
            Write-Host "`n[Info] CTRL-C detected, cleaning up..." -ForegroundColor Cyan
            $global:ctrlCPressed = $true
            try {
                foreach ($window in $script:sessions.Windows) {
                    if ($window.WS) { $window.WS.Abort() }
                    if ($window.CancelSource) { $window.CancelSource.Cancel() }
                }
                $timeoutTask = [Task]::Delay(500)
                $timeoutTask.Wait()
            } catch { }
            continue
        }
    }

    function Update-ScreenBuffer {
        param(
            [int]$WindowIndex,
            [string]$Message
        )
        
        # Lock the screen buffer operations
        [System.Threading.Monitor]::Enter($script:screenBufferLock)
        try {
            $windowHeight = [Console]::WindowHeight
            $windowWidth = [Console]::WindowWidth
            $midPoint = [math]::Floor($windowHeight / 2)
            
            # Initialize buffer if needed
            if ($script:screenBuffer.Count -eq 0) {
                $script:screenBuffer = @(
                    @{  # Top window
                        Messages = [System.Collections.ArrayList]::new()
                        Input = ""
                    },
                    @{  # Bottom window
                        Messages = [System.Collections.ArrayList]::new()
                        Input = ""
                    }
                )
            }
            
            # Add message to appropriate window buffer
            if ($Message) {
                # Split message into lines if it's longer than window width
                $lines = @()
                $words = $Message -split '(?<=[\s,{}[\]"])'
                $currentLine = ""
                
                foreach ($word in $words) {
                    if (($currentLine.Length + $word.Length) -ge $windowWidth) {
                        if ($currentLine) {
                            $lines += $currentLine.TrimEnd()
                            $currentLine = ""
                        }
                        while ($word.Length -gt $windowWidth) {
                            $lines += $word.Substring(0, $windowWidth)
                            $word = $word.Substring($windowWidth)
                        }
                    }
                    $currentLine += $word
                }
                if ($currentLine) {
                    $lines += $currentLine.TrimEnd()
                }
                
                # Add each line as a separate message (in reverse to maintain order)
                [array]::Reverse($lines)
                foreach ($line in $lines) {
                    [void]$script:screenBuffer[$WindowIndex].Messages.Insert(0, $line)
                }
                
                # Trim message history if needed
                $maxLines = if ($WindowIndex -eq 0) { $midPoint - 2 } else { $windowHeight - $midPoint - 2 }
                while ($script:screenBuffer[$WindowIndex].Messages.Count -gt $maxLines) {
                    $script:screenBuffer[$WindowIndex].Messages.RemoveAt($script:screenBuffer[$WindowIndex].Messages.Count - 1)
                }
            }
            
            # Update inputs
            $script:screenBuffer[0].Input = $script:sessions.Windows[0].Input
            $script:screenBuffer[1].Input = $script:sessions.Windows[1].Input
            
            # Render entire screen
            Render-Screen
        }
        finally {
            [System.Threading.Monitor]::Exit($script:screenBufferLock)
        }
    }

    function Connect-WebSocket {
        param(
            [int]$WindowIndex,
            [string]$Uri,
            [switch]$IsInitial
        )

        try {
            Write-WindowMessage -WindowIndex $WindowIndex -Message "Connecting to WebSocket..."
            
            # Create WebSocket
            $ws = New-Object System.Net.WebSockets.ClientWebSocket
            $cancelSource = New-Object System.Threading.CancellationTokenSource
            
            # Connect
            $connectTask = $ws.ConnectAsync($Uri, $cancelSource.Token)
            $connectTask.Wait()
            
            if ($ws.State -ne [System.Net.WebSockets.WebSocketState]::Open) {
                throw "WebSocket connection failed"
            }
            
            Write-WindowMessage -WindowIndex $WindowIndex -Message "Connected successfully"

            if ($WindowIndex -eq 1) {
                Write-WindowMessage -WindowIndex $WindowIndex -Message "Valid commands: TTL | RESPAWN | TEARDOWN"
            }
            
            # Store connection
            $script:sessions.Windows[$WindowIndex].WS = $ws
            $script:sessions.Windows[$WindowIndex].CancelSource = $cancelSource
            $script:sessions.Windows[$WindowIndex].URI = $Uri
            
            # Start receive runspace
            $sync = [hashtable]::Synchronized(@{
                Host = $Host
                Running = $true
                MyUserId = $null
                WS = $ws
                CurrentInput = ""
                ForceExit = $false
                WindowIndex = $WindowIndex
                WriteMessage = ${function:Write-WindowMessage}
                UpdateScreenBuffer = ${function:Update-ScreenBuffer}
                Input = ""
                FilterCategories = $filterCategories
            })
            
            $script:sessions.Windows[$WindowIndex].Sync = $sync
            
            $runspace = [runspacefactory]::CreateRunspace()
            $runspace.Open()
            $runspace.SessionStateProxy.SetVariable('sync', $sync)
            $runspace.SessionStateProxy.SetVariable('cancelSource', $cancelSource)
            $runspace.SessionStateProxy.SetVariable('login_user', $login_user)
              
            $receivePS = [powershell]::Create().AddScript({
              $buffer = [byte[]]::new(8192)
              $segment = [ArraySegment[byte]]::new($buffer)
              $messageBuffer = ""
              
              function Write-WindowMessage {
                  param($WindowIndex, $Message, $ForegroundColor = 'White')
                  
                  # Clean up JSON messages at the source
                  if ($Message -match '^\{.*\}$') {
                      $Message = [regex]::Replace($Message, '\\"', '"')
                  }
                  $sync.UpdateScreenBuffer.Invoke($WindowIndex, $Message)
              }
              
              while ($sync.Running) {
                  try {
                      if ($sync.WS.State -eq [System.Net.WebSockets.WebSocketState]::Open) {
                          try {
                              $result = $sync.WS.ReceiveAsync($segment, $cancelSource.Token).GetAwaiter().GetResult()
                              if ($result.Count -gt 0) {
                                  $message = [System.Text.Encoding]::UTF8.GetString($buffer, 0, $result.Count)
                                    
                                    if ($sync.WindowIndex -eq 0) {
                                        # Top window - handle JSON messages
                                        $messageBuffer += $message
                                        while ($messageBuffer -match '(\{(?:[^{}]|(?<c>\{)|(?<-c>\}))+(?(c)(?!))\})') {
                                            $jsonMatch = $matches[1]
                                            try {
                                                $jsonMessage = $jsonMatch | ConvertFrom-Json
                                                # Check if message contains any of the filtered categories
                                                $shouldFilter = $false
                                                foreach ($category in $sync.FilterCategories) {
                                                    if ($jsonMessage.PSObject.Properties.Name.Contains($category)) {
                                                        $shouldFilter = $true
                                                        break
                                                    }
                                                }
                                                
                                                if (-not $shouldFilter) {
                                                    Write-WindowMessage -WindowIndex $sync.WindowIndex -Message "[Message] $jsonMatch"
                                                }
                                                $messageBuffer = $messageBuffer.Substring($messageBuffer.IndexOf($jsonMatch) + $jsonMatch.Length)
                                            } catch {
                                                $nextBrace = $messageBuffer.IndexOf('{', 1)
                                                if ($nextBrace -gt 0) {
                                                    $messageBuffer = $messageBuffer.Substring($nextBrace)
                                                } else {
                                                    $messageBuffer = ""
                                                }
                                            }
                                        }
                                    } else {
                                        # Bottom window - display raw messages
                                        Write-WindowMessage -WindowIndex $sync.WindowIndex -Message $message
                                    }
                                }
                            } catch {
                                if (-not $cancelSource.Token.IsCancellationRequested -and 
                                    $sync.WS.State -ne [System.Net.WebSockets.WebSocketState]::Aborted) {
                                    Write-WindowMessage -WindowIndex $sync.WindowIndex -Message "[Error] Receive error: $_"
                                }
                            }
                        } else {
                            Write-WindowMessage -WindowIndex $sync.WindowIndex -Message "[Error] WebSocket disconnected"
                            break
                        }
                    } catch {
                        if (-not $cancelSource.Token.IsCancellationRequested -and 
                            $sync.WS.State -ne [System.Net.WebSockets.WebSocketState]::Aborted) {
                            Write-WindowMessage -WindowIndex $sync.WindowIndex -Message "[Error] WebSocket error: $_"
                        }
                    }
                    Start-Sleep -Milliseconds 10
                }
            })



            $receivePS.Runspace = $runspace
            $script:sessions.Windows[$WindowIndex].RunspaceHandle = $runspace
            $script:sessions.Windows[$WindowIndex].ReceivePS = $receivePS
            
            # Start receiving
            $null = $receivePS.BeginInvoke()
            
            # Send initial messages if needed
            if ($IsInitial) {
                $chimney = get-chimney
                if ($chimney) {
                    $msg = @{
                        type = "WS_CHIMNEY"
                        chimney = $chimney
                    } | ConvertTo-Json
                    Send-WebSocketMessage -WindowIndex $WindowIndex -Message $msg
                }
            }
            
            return $true
        }
        catch {
            Write-Host "[Error] Connection failed: $_" -ForegroundColor Red
            return $false
        }
    }

    function get-chimney {
        # First request - login
        $session = New-Object Microsoft.PowerShell.Commands.WebRequestSession
        $session.UserAgent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36"

        try {
            $ProgressPreference = 'SilentlyContinue'
            $response = Invoke-WebRequest -UseBasicParsing `
                -Uri "https://account.counterhack.com/login" `
                -Method "POST" `
                -WebSession $session `
                -ContentType "application/x-www-form-urlencoded" `
                -Body "email=$login_user&password=$login_pass"

            $response2 = Invoke-WebRequest -UseBasicParsing `
                -Uri "https://account.counterhack.com/hhc24magic" `
                -Method "GET" `
                -WebSession $session `
                -MaximumRedirection 0 `
                -ErrorAction SilentlyContinue

            return [regex]::Match($($response2.Headers.Location), "[\w\-]{36}").Value
        }
        catch [System.Net.WebException] {
            if ($_.Exception.Response.StatusCode -eq 302) {
                $location = $_.Exception.Response.Headers['Location']
                Write-Host "302 Redirect Location: $location"
            }
            else {
                Write-Host "Error: $($_.Exception.Message)"
            }
        }
    }

    function Update-Display {
        [System.Threading.Monitor]::Enter($script:screenBufferLock)
        try {
            Render-Screen
        }
        finally {
            [System.Threading.Monitor]::Exit($script:screenBufferLock)
        }
    }

    function Switch-Window {
        [System.Threading.Monitor]::Enter($script:switchLock)
        try {
            $script:sessions.Active = ($script:sessions.Active + 1) % 2
            # Only update the display if we can get the render lock
            if ([System.Threading.Monitor]::TryEnter($script:renderLock, 100)) {
                try {
                    Render-Screen
                }
                finally {
                    [System.Threading.Monitor]::Exit($script:renderLock)
                }
            }
        }
        finally {
            [System.Threading.Monitor]::Exit($script:switchLock)
        }
    }

    function Write-WindowMessage {
        param(
            [int]$WindowIndex,
            [string]$Message,
            [System.ConsoleColor]$ForegroundColor = 'White'
        )
        
        # Clean up JSON messages using explicit regex replace
        if ($Message -match '^\{.*\}$') {
            $Message = [regex]::Replace($Message, '\\(?!")', '\')  # Keep non-quote escapes
            $Message = [regex]::Replace($Message, '\\"', '"')      # Replace escaped quotes
        }
        
        [System.Threading.Monitor]::Enter($script:messageLock)
        try {
            [void]$script:screenBuffer[$WindowIndex].Messages.Insert(0, $Message)
            
            # Trim message history if needed
            $windowHeight = [Console]::WindowHeight
            $midPoint = [math]::Floor($windowHeight / 2)
            $maxLines = if ($WindowIndex -eq 0) { $midPoint - 2 } else { $windowHeight - $midPoint - 2 }
            while ($script:screenBuffer[$WindowIndex].Messages.Count -gt $maxLines) {
                $script:screenBuffer[$WindowIndex].Messages.RemoveAt($script:screenBuffer[$WindowIndex].Messages.Count - 1)
            }
            
            # Only update the display if we can get the render lock
            if ([System.Threading.Monitor]::TryEnter($script:renderLock, 100)) {
                try {
                    Render-Screen
                }
                finally {
                    [System.Threading.Monitor]::Exit($script:renderLock)
                }
            }
        }
        finally {
            [System.Threading.Monitor]::Exit($script:messageLock)
        }
    }

    function Render-Screen {
        # Already under renderLock when called
        $windowHeight = [Console]::WindowHeight
        $windowWidth = [Console]::WindowWidth
        $bufferWidth = [Console]::BufferWidth
        $midPoint = [math]::Floor($windowHeight / 2)
        $promptPrefix = "> "
        
        # Ensure buffer width matches window width
        if ($bufferWidth -ne $windowWidth) {
            try {
                [Console]::BufferWidth = $windowWidth
            } catch {
                # If we can't set the buffer width, use the smaller of the two
                $windowWidth = [Math]::Min($windowWidth, $bufferWidth)
            }
        }
        
        # Create a copy of the messages to render
        [System.Threading.Monitor]::Enter($script:messageLock)
        try {
            $topMessages = @($script:screenBuffer[0].Messages.ToArray())
            $bottomMessages = @($script:screenBuffer[1].Messages.ToArray())
        }
        finally {
            [System.Threading.Monitor]::Exit($script:messageLock)
        }
        
        # Clear screen safely
        try {
            Clear-Host
        } catch {
            # Fallback clear if Clear-Host fails
            for ($i = 0; $i -lt $windowHeight; $i++) {
                [Console]::SetCursorPosition(0, $i)
                [Console]::Write(" " * $windowWidth)
            }
        }
        
        # Safe write function
        function Write-SafeLine {
            param($text, $line)
            if ($line -ge 0 -and $line -lt $windowHeight) {
                try {
                    [Console]::SetCursorPosition(0, $line)
                    if ($text.Length -gt $windowWidth) {
                        [Console]::Write($text.Substring(0, $windowWidth))
                    } else {
                        [Console]::Write($text.PadRight($windowWidth))
                    }
                } catch {
                    # Ignore write errors
                }
            }
        }
        
        # Draw top window messages
        $currentLine = $midPoint - 2
        foreach ($msg in $topMessages) {
            if ($currentLine -ge 0) {
                Write-SafeLine -text $msg -line $currentLine
                $currentLine--
            }
            if ($currentLine -lt 0) { break }
        }
        
        # Draw separator
        Write-SafeLine -text ("-" * $windowWidth) -line $midPoint
        
        # Draw bottom window messages
        $currentLine = $windowHeight - 2
        foreach ($msg in $bottomMessages) {
            if ($currentLine -gt $midPoint) {
                Write-SafeLine -text $msg -line $currentLine
                $currentLine--
            }
            if ($currentLine -le $midPoint) { break }
        }
        
        # Draw prompts with truncated input
        $maxInputWidth = $windowWidth - $promptPrefix.Length - 1
        
        # Top prompt
        $topInput = $script:sessions.Windows[0].Input
        $topDisplayInput = if ($topInput.Length -gt $maxInputWidth) {
            "..." + $topInput.Substring($topInput.Length - ($maxInputWidth - 3))
        } else {
            $topInput
        }
        Write-SafeLine -text ($promptPrefix + $topDisplayInput) -line ($midPoint - 1)
        
        # Bottom prompt
        $bottomInput = $script:sessions.Windows[1].Input
        $bottomDisplayInput = if ($bottomInput.Length -gt $maxInputWidth) {
            "..." + $bottomInput.Substring($bottomInput.Length - ($maxInputWidth - 3))
        } else {
            $bottomInput
        }
        Write-SafeLine -text ($promptPrefix + $bottomDisplayInput) -line ($windowHeight - 1)
        
        # Position cursor safely
        try {
            $promptLine = if ($script:sessions.Active -eq 0) { $midPoint - 1 } else { $windowHeight - 1 }
            $currentInput = if ($script:sessions.Active -eq 0) { $topDisplayInput } else { $bottomDisplayInput }
            $cursorX = [Math]::Min($promptPrefix.Length + $currentInput.Length, $windowWidth - 1)
            
            # If we're at the end of a line that exactly fills the width,
            # move to the start of the next line
            if ($cursorX -eq ($windowWidth - 1) -and 
                ($promptPrefix.Length + $currentInput.Length) -ge $windowWidth) {
                [Console]::SetCursorPosition(0, [Math]::Min($promptLine + 1, $windowHeight - 1))
            } else {
                [Console]::SetCursorPosition($cursorX, $promptLine)
            }
        } catch {
            # If cursor positioning fails, try to set it to a safe position
            try {
                [Console]::SetCursorPosition(0, $windowHeight - 1)
            } catch {
                # Ignore if even safe positioning fails
            }
        }
    }

    function Send-WebSocketMessage {
        param (
            [int]$WindowIndex,
            [string]$Message
        )
        
        try {
            $ws = $script:sessions.Windows[$WindowIndex].WS
            if ($ws -and $ws.State -eq [System.Net.WebSockets.WebSocketState]::Open) {
                Write-WindowMessage -WindowIndex $WindowIndex -Message "[Sent] $Message"
                $bytes = [System.Text.Encoding]::UTF8.GetBytes($Message)
                $segment = [ArraySegment[byte]]::new($bytes)
                $ws.SendAsync($segment, [System.Net.WebSockets.WebSocketMessageType]::Text, $true, $script:sessions.Windows[$WindowIndex].CancelSource.Token).Wait()
            }
        }
        catch {
            Write-WindowMessage -WindowIndex $WindowIndex -Message "[Error] Send failed: $_"
        }
    }

    # Main loop
    try {
        # Initialize display
        Clear-Host
        
        # Connect first WebSocket
        $null = Connect-WebSocket -WindowIndex 0 -Uri $ws_url -IsInitial
        
        # Set initial commands
        $script:sessions.Windows[0].Input = '{"type":"HELLO_ENTITY","entityType":"terminal","id":"termSantaVision"}'
        $script:sessions.Windows[1].Input = "connect wss://hhc24-gatexor.holidayhackchallenge.com/ws?username=if_my_grandmother_had_wheels_she_would_be_a_bicycle&page=santavision&env=prod&id="

        # Main input loop
        Write-Host "`nType your messages (press Enter to send, 'quit' to exit, TAB to switch windows):"
        Update-Display
        
        $exitRequested = $false
        
        while (-not $exitRequested) {
            try {
                # Check for window resize
                $currentHeight = [Console]::WindowHeight
                $currentWidth = [Console]::WindowWidth
                if ($currentHeight -ne $prevWindowHeight -or $currentWidth -ne $prevWindowWidth) {
                    Clear-Host
                    Update-Display
                    $prevWindowHeight = $currentHeight
                    $prevWindowWidth = $currentWidth
                }

                $activeWindow = $script:sessions.Active
                $activeSession = $script:sessions.Windows[$activeWindow]
                
                if ($activeSession.Sync.ForceExit) { 
                    $exitRequested = $true
                    break 
                }
                
                if ([Console]::KeyAvailable) {
                    $key = [Console]::ReadKey($true)
                    
                    # Lock the input processing
                    [System.Threading.Monitor]::Enter($script:screenBufferLock)
                    try {
                        switch ($key.Key) {
                            "Tab" { 
                                Switch-Window
                            }
                            "UpArrow" {
                                $history = $activeSession.History
                                if ($history.Count -gt 0) {
                                    if ($activeSession.HistoryIndex -lt ($history.Count - 1)) {
                                        $activeSession.HistoryIndex++
                                        $activeSession.Input = $history[$activeSession.HistoryIndex]
                                        Update-Display
                                    }
                                }
                            }
                            "DownArrow" {
                                if ($activeSession.HistoryIndex -gt -1) {
                                    $activeSession.HistoryIndex--
                                    $activeSession.Input = if ($activeSession.HistoryIndex -eq -1) {
                                        ""
                                    } else {
                                        $activeSession.History[$activeSession.HistoryIndex]
                                    }
                                    Update-Display
                                }
                            }
                            "Enter" {
                                $input = $activeSession.Input
                                if ($input -eq "quit") { 
                                    $exitRequested = $true
                                    break 
                                }
                                
                                # Add to history if not empty and not duplicate of last command
                                if ($input -and ($activeSession.History.Count -eq 0 -or $activeSession.History[0] -ne $input)) {
                                    $activeSession.History.Insert(0, $input)
                                    # Keep only last 20 commands
                                    while ($activeSession.History.Count -gt 20) {
                                        $activeSession.History.RemoveAt(20)
                                    }
                                }
                                $activeSession.HistoryIndex = -1
                                
                                # Get current window dimensions
                                $windowHeight = [Console]::WindowHeight
                                $midPoint = [math]::Floor($windowHeight / 2)
                                
                                # Clear the current line before sending (with bounds checking)
                                $promptLine = if ($activeWindow -eq 0) { 
                                    [Math]::Max(0, $midPoint - 1) 
                                } else { 
                                    [Math]::Min($windowHeight - 1, $windowHeight - 1) 
                                }
                                
                                [Console]::SetCursorPosition(0, $promptLine)
                                [Console]::Write(" " * ([Console]::WindowWidth))
                                
                                if ($input -match '^connect\s+(.+)$') {
                                    $uri = $matches[1]
                                    $null = Connect-WebSocket -WindowIndex $activeWindow -Uri $uri
                                }
                                elseif ($input) {
                                    if ($activeWindow -eq 0) {
                                        # Top window - use JSON
                                        try {
                                            $null = $input | ConvertFrom-Json
                                            Send-WebSocketMessage -WindowIndex $activeWindow -Message $input
                                        } catch {
                                            $msg = @{
                                                type = "message"
                                                content = $input
                                            } | ConvertTo-Json
                                            Send-WebSocketMessage -WindowIndex $activeWindow -Message $msg
                                        }
                                    } else {
                                        # Bottom window - send raw message
                                        Send-WebSocketMessage -WindowIndex $activeWindow -Message $input
                                    }
                                }
                                $activeSession.Input = ""
                                Update-Display
                            }
                            "Backspace" {
                                if ($activeSession.Input.Length -gt 0) {
                                    # Get current window dimensions
                                    $windowHeight = [Console]::WindowHeight
                                    $midPoint = [math]::Floor($windowHeight / 2)
                                    $windowWidth = [Console]::WindowWidth
                                    
                                    # Calculate prompt line with bounds checking
                                    $promptLine = if ($activeWindow -eq 0) { 
                                        [Math]::Max(0, $midPoint - 1) 
                                    } else { 
                                        [Math]::Min($windowHeight - 1, $windowHeight - 1) 
                                    }
                                    
                                    # Clear the current line
                                    [Console]::SetCursorPosition(0, $promptLine)
                                    [Console]::Write(" " * $windowWidth)
                                    
                                    # Update input
                                    $activeSession.Input = $activeSession.Input.Substring(0, $activeSession.Input.Length - 1)
                                    
                                    # Calculate visible portion of input
                                    $promptPrefix = "> "
                                    $maxWidth = $windowWidth - $promptPrefix.Length - 1
                                    $displayInput = if ($activeSession.Input.Length -gt $maxWidth) {
                                        $activeSession.Input.Substring($activeSession.Input.Length - $maxWidth)
                                    } else {
                                        $activeSession.Input
                                    }
                                    
                                    # Redraw prompt and input
                                    [Console]::SetCursorPosition(0, $promptLine)
                                    Write-Host "> $displayInput" -NoNewline
                                    
                                    # Position cursor after input
                                    [Console]::SetCursorPosition($promptPrefix.Length + $displayInput.Length, $promptLine)
                                }
                            }
                            default {
                                if (-not [char]::IsControl($key.KeyChar)) {
                                    $activeSession.Input += $key.KeyChar
                                    Update-Display
                                }
                            }
                        }
                    }
                    finally {
                        [System.Threading.Monitor]::Exit($script:screenBufferLock)
                    }
                }
                Start-Sleep -Milliseconds 1
            }
            catch {
                Write-WindowMessage -WindowIndex $activeWindow -Message "[Error] Input processing error: $_"
                Start-Sleep -Milliseconds 100  # Add small delay on error
            }
        }
    }
    catch {
        Write-Host "`n[Error] Fatal error: $_" -ForegroundColor Red
    }
    finally {
        clear  # exit cleanly

        if (-not $exitRequested) {
            Write-Host "`n[Status] Unexpected exit detected, cleaning up..." -ForegroundColor Yellow
        } else {
            Write-Host "`n[Status] Cleaning up..." -ForegroundColor Cyan
        }
        try {
            foreach ($window in $script:sessions.Windows) {
                if ($window.WS) { $window.WS.Abort() }
                if ($window.CancelSource) { $window.CancelSource.Cancel() }
                if ($window.ReceivePS) { $window.ReceivePS.Stop() }
            }
            $timeoutTask = [Task]::Delay(500)
            $timeoutTask.Wait()
        } catch { }
        Write-Host "[Status] Done" -ForegroundColor Green
    }
}

# websocket -c 'style,SET_LOCATIONS'

```




