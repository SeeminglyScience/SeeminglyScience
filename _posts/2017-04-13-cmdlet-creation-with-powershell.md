---
layout: post
title: "Cmdlet Creation with PowerShell"
description: "How to create a PowerShell cmdlet without C#"
category: "PowerShell"
tags: ["PowerShell", "Cmdlet", "Classes"]
---
{% include JB/setup %}

I was looking through the [PowerShell-Tests](https://github.com/PowerShell/PowerShell-Tests/tree/master) repo and I saw they were creating Cmdlets inline for some tests. They were using C# and Add-Type which got me wondering.

Can you create one using pure PowerShell? Yeah, you can. With a little extra handling.

## Building the Cmdlet

```powershell
using namespace System.Management.Automation
using namespace System.Reflection

# Basic do nothing Cmdlet.
[Cmdlet([VerbsDiagnostic]::Test, 'Cmdlet')]
class TestCmdletCommand : PSCmdlet {
    [Parameter(ValueFromPipeline)]
    [object]
    $InputObject;

    [void] ProcessRecord () {
        $this.WriteObject($this.InputObject)
    }
}
```

Just a basic cmdlet that passes anything it gets from the pipeline.  But right now, it's just a class.
Now we need to tell PowerShell it's a cmdlet. This is where we have to stretch a little.

<!--more-->

First, we need to create a session state entry.

```powershell
$cmdletEntry = [Runspaces.SessionStateCmdletEntry]::new(
    <# name:             #> 'Test-Cmdlet',
    <# implementingType: #> [TestCmdletCommand],
    <# helpFileName:     #> $null
)
```

Then we need to get the internal session state.  This is the "private" version of the normally accessible
`$ExecutionContext.SessionState`.

```powershell
$internal = $ExecutionContext.SessionState.GetType().
    GetProperty('Internal', [BindingFlags]'Instance, NonPublic').
    GetValue($ExecutionContext.SessionState)
```

Now we can load the session state entry into our current session.

```powershell
$internal.GetType().InvokeMember(
    <# name:       #> 'AddSessionStateEntry',
    <# invokeAttr: #> [BindingFlags]'InvokeMethod, Instance, NonPublic',
    <# binder:     #> $null,
    <# target:     #> $internal,
    <# args:       #> @(
        <# entry: #> $cmdletEntry
        <# local: #> $true
    )
)
```

That still won't actually load the cmdlet.  But if we wrap all that in a module (dump it in a .psm1 file)
and import it, we have ourselves a working cmdlet.

## PSClassCmdlet.psm1

<script src="https://gist.github.com/SeeminglyScience/533140625f67f2f46535e652c8cd6da2.js"></script>

## Trying it out

```powershell
PS> Import-Module .\PSClassCmdlet.psm1
PS> Get-Module PSClassCmdlet.psm1

ModuleType Version    Name                   ExportedCommands
---------- -------    ----                   ----------------
Script     0.0        PSClassCmdlet          Test-Cmdlet

PS> Get-Command Test-Cmdlet

CommandType     Name                         Version    Source
-----------     ----                         -------    ------
Cmdlet          Test-Cmdlet                  0.0        PSClassCmdlet

PS> 0..3 | Test-Cmdlet
0
1
2
3
```

## Why use this over functions

That's a good question.  I haven't actually tried using it past this example, but here are some thoughts.

### Using the class structure

There doesn't seem to be a whole lot of adoption of classes just yet. But if you're a fan this could
be a nice alternative to traditional PowerShell functions.

### Inheritance

Inheritance is the main value I see. Lets say you are working on a module where every command shares
some parameters.  You could set up a base cmdlet and every child cmdlet will share those parameters.
You could also inherit existing cmdlets.  If that's something you're interested in you can run this
snippet to show you what existing cmdlets can be inherited.

```powershell
$assemblies = [AppDomain]::CurrentDomain.GetAssemblies()
$assemblies.GetTypes().Where{
    [System.Management.Automation.Cmdlet].IsAssignableFrom($_) -and
    -not $_.IsSealed
}
```

## Word of warning

Unless there is a much easier way to load these than I found, it's pretty clear this isn't exactly
*supported*.  If you decide to give this a go, keep in mind that some things may act unpredicably.
If I happen to find any gotchas I'll update this post.