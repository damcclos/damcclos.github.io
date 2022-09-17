---
layout: post
title: Realtime output from child processes with Powershell
tags: [powershell]
---
Powershell buffers output from child processes waiting until the child process has completed before outputting any text. This is quite frustrating as the lack of feedback for long running processes can be an issue. There are several ways to start a child process with Powershell each with it's own interesting tidbits of why you would use it. That is beyond the scope of this post as for my work I was concerned with collecting the collated output from the child processes stdout and stderr streams from the child process to a variable while also having it printed to the console.

I tried several approaches before landing on this solution such as using the redirection through Start-Process and creating the process with System.Diagnostics.Process. None of the approaches worked quite like I wanted.

First, I found using the call operator with redirection, as would be done with Linux shells, was the most straight forward for collating stdout and stderr. This also afforded me the ability to easily collect the output to a variable.

Next, came exceptions that would dump Powershell exception information for error messages from the child processes. While useful in some situations, completely unnecessary for me. Piping the child process output to a `ForEach-Object` block and extracting the actual child process message from the exception did the trick for getting rid of the Powershell exception information.

Last, simply writing the message to the host from the `ForEach-Object` block was enough to both have the output collected to a variable and written to the screen in realtime without waiting for the process to finish.

While this may not be a perfect solution for everyone, this is the simplest solution I have been able to find or develop. Below is the entire script I use for calling executables. I have not done extensive testing on this code so use at your own risk but as it uses the call operator, anything you can do with the call operator you should be able to do here.

```powershell
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

    return $Output
}
```

Simple Examples

```powershell
$Output = Verify-Call git checkout master
```

```powershell
$Success, $Output = Check-Call git checkout master
if($Success)
    ...
```

```powershell
if(Check-Call git checkout master)
    ...
```
