---
layout: post
title: Realtime output from child processes with Powershell
tags: [powershell]
---
```
function Invoke-Call {
    $exe = $args[0]
    $args = @($args[1..$args.Length])

    Write-Host 'Command: ', $exe, $args

    # Disable ErrorActionPreference temporarily https://stackoverflow.com/questions/10666101/lastexitcode-0-but-false-in-powershell-redirecting-stderr-to-stdout-gives
    $SaveErrorActionPreference = $ErrorActionPreference
    $ErrorActionPreference = 'Continue'

    # Refer to https://stackoverflow.com/questions/8097354/how-do-i-capture-the-output-into-a-variable-from-an-external-process-in-powershe
    $Output = & $exe $args 2>&1 | ForEach-Object {
        if($_ -Is [System.Management.Automation.ErrorRecord])
        {
            $_.Exception.Message | Write-Host
            $_.Exception.Message
        }
        else
        {
            $_ | Write-Host
            $_
        }
    }

    $ExitCode = $LASTEXITCODE

    # Reset ErrorActionPreference
    $ErrorActionPreference = $SaveErrorActionPreference

    return $ExitCode, $Output
}

function Check-Call {
    $ExitCode, $Output = Invoke-Call @args
    $Success = ($ExitCode -eq 0)
    return $Success, $Output
}

function Verify-Call {
    $ExitCode, $Output = Invoke-Call @args
    $Success = ($ExitCode -eq 0)
    if (-Not $Success)
    {
        Write-Error "'$Args' failed with exit code $ExitCode"
        exit $ExitCode
    }
}

# Specific function for returning the results so the output to the host is not duplicated
function Verify-CallWithOutput {
    $ExitCode, $Output = Invoke-Call @args
    $Success = ($ExitCode -eq 0)
    if (-Not $Success)
    {
        Write-Error "'$Args' failed with exit code $ExitCode"
        exit $ExitCode
    }

    return $Output
}

function Try-Call {
    param (
        [Parameter(Position=0, Mandatory)]
        [ScriptBlock]
        $ScriptBlock,

        [Parameter(Mandatory)]
        [int]
        $RetryCount,

        [Parameter(Mandatory)]
        [int]
        $SecondsBetweenAttempts
    )

    $counter = 0

    $result = & $ScriptBlock
    while($result -ne $true)
    {
        $counter += 1
        if($counter -eq $RetryCount)
        {
            Write-Error "Try-Call failed"
            exit 1
        }

        if($SecondsBetweenAttempts -gt 0)
        {
            Start-Sleep -Seconds $SecondsBetweenAttempts
        }

        $result = & $ScriptBlock
    }
}
```
