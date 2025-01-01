```
function portscan {
    param(
        [Parameter(Position=0, Mandatory=$true)]
        [string]$ip,
        [Parameter(Position=1, Mandatory=$false)]
        [string]$range = "1-65535"  # Default to full port range
    )

    if ($range -match '^(\d+)-(\d+)$') {
        $start = [int]$matches[1]
        $end = [int]$matches[2]
    } else {
        Write-Host "Invalid port range format. Use start-end, e.g., 1-65535" -ForegroundColor Red
        return
    }

    # Validate port range
    if ($start -lt 1 -or $end -gt 65535 -or $start -gt $end) {
        Write-Host "Invalid port range. Ports must be between 1 and 65535, and start must be less than end." -ForegroundColor Red
        return
    }

    $totalPorts = $end - $start + 1
    $maxThreads = 500
    $openPorts = @()

    # Create RunspacePool
    $runspacePool = [runspacefactory]::CreateRunspacePool(1, $maxThreads)
    $runspacePool.Open()

    # Enhanced port scanning scriptblock
    $scriptBlock = {
        param ($ip, $port)
        $timeout = 750
        
        try {
            $tcpClient = New-Object System.Net.Sockets.TcpClient
            $tcpClient.ReceiveTimeout = $timeout
            $tcpClient.SendTimeout = $timeout
            
            $attempts = 2
            while ($attempts -gt 0) {
                try {
                    $connect = $tcpClient.BeginConnect($ip, $port, $null, $null)
                    if ($connect.AsyncWaitHandle.WaitOne($timeout, $false)) {
                        try {
                            $tcpClient.EndConnect($connect)
                            if ($tcpClient.Connected) {
                                $tcpClient.Close()
                                return $port
                            }
                        } catch {}
                    }
                    $attempts--
                    if ($attempts -gt 0) {
                        Start-Sleep -Milliseconds 100
                    }
                }
                catch {
                    $attempts--
                    if ($attempts -gt 0) {
                        Start-Sleep -Milliseconds 100
                    }
                }
            }
            
            if ($tcpClient.Connected) { $tcpClient.Close() }
        }
        catch {}
        finally {
            if ($tcpClient) {
                $tcpClient.Close()
                $tcpClient.Dispose()
            }
        }
        return $null
    }

    Clear-Host
    Write-Host "Scanning $ip ports $start-$end`n--"
    
    # Initialize progress line position
    $progressLine = [Console]::CursorTop
    Write-Host "Progress: 0%`n`n"

    $jobs = New-Object System.Collections.ArrayList
    $completed = 0
    $lastProgress = 0
    $resultsLine = [Console]::CursorTop - 1

    # Create all jobs first (0-40% progress)
    foreach ($port in $start..$end) {
        $ps = [powershell]::Create().AddScript($scriptBlock).AddArgument($ip).AddArgument($port)
        $ps.RunspacePool = $runspacePool

        [void]$jobs.Add([PSCustomObject]@{
            PowerShell = $ps
            Handle = $ps.BeginInvoke()
            Port = $port
            Completed = $false
        })

        # Update progress during job creation (0-40%)
        $progress = if ($totalPorts -eq 0) {
            0
        } else {
            [math]::Floor(($port - $start + 1) / [math]::Max(1, $totalPorts) * 40)
        }
        
        if ($progress -ne $lastProgress) {
            [Console]::SetCursorPosition(0, $progressLine)
            [Console]::Write("`r" + " ".PadRight([Console]::WindowWidth - 1))
            [Console]::SetCursorPosition(0, $progressLine)
            [Console]::Write("Progress: $progress%`n`n")
            $lastProgress = $progress
        }
    }    

    # Process results
    do {
        $jobsToProcess = $jobs | Where-Object { $_.Handle.IsCompleted -and -not $_.Completed }
        $completed = ($jobs | Where-Object { $_.Completed }).Count
        
        foreach ($job in $jobsToProcess) {
            try {
                $result = $job.PowerShell.EndInvoke($job.Handle)
                if ($result) {
                    [Console]::SetCursorPosition(0, $resultsLine)
                    [Console]::Write("`r" + " ".PadRight([Console]::WindowWidth - 1))
                    [Console]::SetCursorPosition(0, $resultsLine)
                    Write-Host "Port " -NoNewline
                    Write-Host "$([char]27)[38;2;144;238;144mopen$([char]27)[0m" -NoNewline
                    Write-Host ": $result"
                    [Console]::Out.Flush()
                    $resultsLine++
                }
            }
            catch {
                # Ignore EndInvoke errors
            }
            finally {
                $job.PowerShell.Dispose()
                $job.Completed = $true
            }
        }

        # Calculate progress for completion phase
        $remainingJobs = $jobs.Where({-not $_.Completed}).Count
        if ($remainingJobs -eq 0) {
            $progress = 100
        } else {
            $completed = ($jobs | Where-Object { $_.Completed }).Count
            $progress = 40 + [math]::Floor(($completed / [math]::Max(1, $totalPorts)) * 60)
        }

        if ($progress -ne $lastProgress) {
            [Console]::SetCursorPosition(0, $progressLine)
            [Console]::Write("`r" + " ".PadRight([Console]::WindowWidth - 1))
            [Console]::SetCursorPosition(0, $progressLine)
            [Console]::Write("Progress: $progress%")
            $lastProgress = $progress
        }
        
        Start-Sleep -Milliseconds 50
    } while ($jobs.Where({-not $_.Completed}).Count -gt 0)

    # Move to a new line after all output
    [Console]::SetCursorPosition(0, $resultsLine + 1)
    Write-Host "Scan complete`n--"

    # Clean up
    $runspacePool.Close()
    $runspacePool.Dispose()
}

# portscan 127.0.0.1 1-100

```