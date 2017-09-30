---
layout: post
title: "Invocation Operators, States and Scopes"
description: "Way more information than anyone needs about scope."
category: PowerShell
tags: ["Operators", "Invocation", "Scopes", "SessionState", "ExecutionContext", "StateTree"]
---
{% include JB/setup %}

I'm going to start off with invocation operators, but don't worry it'll come around full circle. If
you're here just for scopes, you can skip to the PSModuleInfo section.

## Common Knowledge

In general you can think of the invocation operators `&` and `.` as a way of telling PowerShell "run
whatever is after this symbol". There are a few different ways you can use theses symbols.

### Name or Path

```powershell
& .\Path\To\Command.ps1
```

Invoke whatever is at this path. The path can be a script, an executable, or anything else that resolves as a command.

```powershell
& Get-ChildItem
```

Invoke a command by name. You don't typically need an operator to invoke a command, but what you
might not know is you can also use the `.` operator here to invoke a command without creating a new
scope.

For example:

```powershell
function MyPrivateCommand {
    $var = 10
}

. MyPrivateCommand
$var
# 10
```

### Invoke a ScriptBlock

```powershell
& { Get-ChildItem }
```

There's a ton of ways to use this and it could probably be it's own blog post. There is one use though,
that I don't see a lot and want to talk about.

Let's say you have a bunch of commands that output to the pipeline, but you really don't care about
their output.  You aren't going to assign them to a variable, so you would typically use `Out-Null`
or `$null =` on each statement to ensure your pipeline remains clean. Instead, you can group those
commands in a single scriptblock and null them all at once.

Lets take `StringBuilder` for instance. This class returns an instance of itself with every method.
It does this so you can "chain" the methods together, like this.

```powershell
$sb = [System.Text.StringBuilder]::new()
$sb.Append("This").AppendLine(" is a string").Append("Cool right?").ToString()
# This is a string
# Cool right?
```

But let's say you can't chain all the way through, you need to stop and check some things after you
append to see what comes next in the string.  If you do that often enough, you end up with a lot of
lines you need to null.

Alternatively, just group them together and don't worry about it.

```powershell
$sb = [System.Text.StringBuilder]::new()
$isString = $false
$null = & {
    $sb.Append("This is a")
    if ($isString) {
        $sb.Append(" string!")
    } else {
        $sb.Append("... I'm not sure actually")
    }
}

$sb.ToString()
# This is a... I'm not sure actually
```

## Less Common Knowledge

### CommandInfo objects

`CommandInfo` is the object type returned by the `Get-Command` cmdlet. I tend to use this when I need
to look a few different places for a command, or if I need to be very specific.

Lets say you're looking for a executable. If they have it installed and in their `$env:PATH`, you want
to use that.  If they don't, you want to install it to the current folder.

```powershell
function Get-DotNet {
    $dotnet = Get-Command dotnet -ErrorAction Ignore
    if (-not $dotnet) {
        # Some code that saves the dotnet cli to ./dotnet
        $dotnet = Get-Command ./dotnet/dotnet.exe -ErrorAction Ignore
    }

    if (-not $dotnet -or $dotnet.Version -lt '2.0.0') {
        throw "Couldn't find dotnet!"
    }

    return $dotnet
}

& (Get-DotNet) build
```

At the end of `Get-DotNet` I don't care where I got it from, or even what type of command it is.  I
can call it all the same. You can use `Get-Command` to narrow down a command to a specific version,
from a specific module, in a specific path, of a specific command type and more.

### PSModuleInfo

Here's where things get cool. I'm going to be honest though, this is not widely applicable. If you
read this and think "I can't see myself ever using this" don't worry, that's normal.  But a few of
you might be screaming with excitement.

```powershell
$module = Get-Module Pester

& $module { $ExecutionContext.SessionState.Module }
# ModuleType Version    Name                                ExportedCommands
# ---------- -------    ----                                ----------------
# Script     4.0.6      pester                              {Add-AssertionOperator,
```

This one works a little differently. It isn't *invoking* the module, instead it's acting as a
modifier.  It basically tells PowerShell to run whatever comes after the module as if it was ran
from within the module.

More specifically, it replaces the `SessionStateInternal` of the command with the `SessionStateInternal`
of the modules `SessionState`.

#### Non-Exported Commands

Most modules have private functions, classes, variables, etc. that it uses internally, but aren't meant
to be used from outside the module. Using this method, you can do just that.

Let me be clear here though. If you do use them, **you're on your own**. They are private for a reason.
Maybe they are dangerous, maybe they are buggy in less than perfect scenarios, or maybe they are just
obtuse to use. Whatever the reason, they are **not supported**. If it's your module though, go nuts.
It's great for troubleshooting and testing.

Anyway, lets look at Pester again. If you've ever glanced at Pester's source, you've probably seen
they make a table of "safe" commands that aren't mocked. Lets say you wanted to make sure a command
was resolved correctly in that table.

```powershell
& (gmo Pester) { $SafeCommands['New-Object'] }

# CommandType     Name                                               Version    Source
# -----------     ----                                               -------    ------
# Cmdlet          New-Object                                         3.1.0.0    Microsoft.PowerShell.Utility
```

You can also call functions, change the value of script scope variables, and even return classes
without having to use the `using module` syntax.

#### Vessels for State

Here's a fun fact, the `SessionState` property on `PSModuleInfo` is *writeable*. This means you can
throw any old `SessionState` into the module and you now have a portable, invokeable link to that state.
Even if the state isn't from a module.

```powershell
$globalState = [PSModuleInfo]::new($false)
$globalState.SessionState = $ExecutionContext.SessionState

$myGlobalVar = 'Hello from global!'
$myGlobalVar
# Hello from global!

$null = [PSModuleInfo]::new({
    $myGlobalVar = 'Hello from a module!'
})
$myGlobalVar
# Hello from global!

$null = [PSModuleInfo]::new({
    . $globalState { $myGlobalVar = 'Hello from a module!' }
})
$myGlobalVar
# Hello from a module!
```

In the first example, the variable changed in the module, but it didn't change the original because
of the different `SessionState`.

#### Enter a Modules Scope

Have you ever been troubleshooting a module and wished you could just enter the scope of the module,
and run commands from the prompt like you were in the `psm1`?

```powershell
. (gmo Pester) { $Host.EnterNestedPrompt() }
```

There you go, you're now in the `psm1`. Run `exit` when you want to return to the global scope. Do you
instead wish that it worked more like you were in a function? Switch the `.` with `&`.

Some of you may have read that last sentence and thought "Wait what? That's not how I thought dot
sourcing worked..." Keep reading, because that's why I want to talk about what I call the
"PowerShell State Tree".

## The PowerShell State Tree

Alright so I just made up the name "State Tree", I don't think there is an actual term that refers
to the whole thing.  The rest *are* real terms though :)

```text
PowerShell Process
   |------ ExecutionContext
   |      |------ SessionStateInternal
   |      |      |------ SessionStateScope
   |      |             |------ SessionStateScope
   |      |             |      |------ SessionStateScope
   |      |             |------ SessionStateScope
   |      |
   |      |------ SessionStateInternal
   |             |------ SessionStateScope
   |
   |------ ExecutionContext
          |------ SessionStateInternal
                 |------ SessionStateScope

```

### ExecutionContext

For every runspace in a process, there is an `ExecutionContext`. This holds instances of a lot of
high level features you don't ever need to worry about, like the help system, the parser, and most
importantly, all of our `SessionState` objects.

The public facing version of this object is `EngineIntrinsics` which is confusingly returned from the
variable `$ExecutionContext`.

### SessionStateInternal

For every module in a runspace, there is a `SessionStateInternal`.  Plus one "Top Level" or "Global"
`SessionStateInternal`.  These internally hold references to several `SessionStateScope` objects.

- **Global** - Typically this scope is the link back to the top level state.
- **Script** - Typically the psm1 file in a module or the first child scope otherwise. If no child
  scopes are created (like when running commands from the console) this scope is the same as global.
- **Current** - The newest child scope, can be the same as script or global.

The public facing version of this object is `SessionState`. It is the object returned from
`$ExecutionContext.SessionState`, `PSModuleInfo.SessionState` and pretty much anything else that
calls itself "SessionState".

### SessionStateScope

I could probably write a whole post on scopes alone. Scopes are like a moment in time during the
invocation of a script. Many scopes are created during a single script execution.

At the very least, any time a `scriptblock` is invoked, a new scope is created by default. Keep in
mind, almost everything is boiled down or brought up to a `scriptblock` at some point in the
invocation cycle. Functions, PowerShell class methods, modules, scripts, even the text you type at
the prompt is or will be a `scriptblock` at some point. Cmdlets (compiled commands like `Get-ChildItem`)
do **not** create scopes.

When a scope is created, it is added as a child to the `Current` scope of the **session state** the command
**belongs to**.  This small detail causes a lot of common misconceptions about how scopes work. When you
first hear of scopes, you initially picture something more like the call stack. You assume that if
you call a function from a module then the scope that gets created will be a child of the scope you
are calling it in.  But instead, it's a part of it's own separate tree of scopes entirely.

There is no public facing version of this object.

### What is Dot Sourcing Really

Towards the start of this post, when describing the dot source operator I described it as "a way to
invoke a command without creating a new scope".  The phrasing used was very specific and purposeful.

When you dot source something it runs in the scope that is marked as the current scope in the
session state the command is from.  That part is very important. It means that you aren't creating
a new scope, **but** you may still **change** scopes. Thankfully, invoking modules provides a very
easy way to demonstrate this.

```powershell
$pester = Get-Module Pester
# Creates a new scope in the Pester module.  This is the same context
# a function from the module would run in.
& $pester { $test; $test = 'test' }
& $pester { $test; $test = 'test' }
```

Both of the above calls return nothing because test was defined in a child scope that stops
existing as soon as the command finishes.

```powershell
$pester = Get-Module Pester
# What happens if we add another in the middle that is dot sourced?
& $pester { $test; $test = 'test' } # Nothing
. $pester { $test; $test = 'test' } # Nothing
& $pester { $test; $test = 'test' } # test
$test # Nothing
```

The first two still return nothing, but the third returns the value we set.  This is because we dot
sourced the second, so no new scope was created, therefore `$test` was defined in the current scope
of the session state. This is just like if the code in the `scriptblock` was placed in the `psm1` file
directly.

The last line doesn't return anything because it runs in the top level session state.  Remember,
dot sourcing runs the command in what is marked as the current scope of the session state for the
command, it does *not* run the *literal* current scope.

### Show-PSStateTree

One of the most challenging things when you are trying to learn about scopes is how difficult it is
to pin down exactly when one is being created.  First, there is no public facing interface for scopes,
so you are limited to using a ton of reflection to even get to a representation of them. Second, almost
all of the things you do in PowerShell create a new scope, so how do you really know how many scopes
you created just trying to figure out if a scope was created?

I mentioned cmdlets don't create a scope, so I made a small cmdlet to help track scopes. It will return
the hash code of several parts of the state tree.  This won't tell you much, but you can compare them
to see when they change, which ones stick around, etc.

Below is the same example as above, with this cmdlet added.

```powershell
$pester = Get-Module Pester

& $pester { Show-PSStateTree; $test; $test = 'test' }

# ExecutionContext     : 56919981
# TopLevelSessionState : 3678064
# EngineSessionState   : 49566835
# GlobalScope          : 62978787
# ModuleScope          : 22936020
# ScriptScope          : 22936020
# ParentScope          : 22936020
# CurrentScope         : 23035345

. $pester { Show-PSStateTree; $test; $test = 'test' }

# ExecutionContext     : 56919981
# TopLevelSessionState : 3678064
# EngineSessionState   : 49566835
# GlobalScope          : 62978787
# ModuleScope          : 22936020
# ScriptScope          : 22936020
# ParentScope          : 62978787
# CurrentScope         : 22936020

& $pester { Show-PSStateTree; $test; $test = 'test' }

# ExecutionContext     : 56919981
# TopLevelSessionState : 3678064
# EngineSessionState   : 49566835
# GlobalScope          : 62978787
# ModuleScope          : 22936020
# ScriptScope          : 22936020
# ParentScope          : 22936020
# CurrentScope         : 42591724
# test

Show-PSStateTree; $test

# ExecutionContext     : 56919981
# TopLevelSessionState : 3678064
# EngineSessionState   : 3678064
# GlobalScope          : 62978787
# ModuleScope          : 62978787
# ScriptScope          : 62978787
# ParentScope          : 0
# CurrentScope         : 62978787
```

[Here's the Gist][1] if you want to play around with it. Copy and paste it into the console, save it
or save it as a file and run it, then test away.

[1]: https://gist.github.com/SeeminglyScience/fc64a6228c115768926b13617c5f56d8

```powershell
$typeDefinition = @'
using System;
using System.Management.Automation;
using System.Reflection;

namespace PSStateTree.Commands
{
    [OutputType(typeof(StateTreeInfo))]
    [Cmdlet(VerbsCommon.Show, "PSStateTree")]
    public class ShowPSStateTreeCommand : PSCmdlet
    {
        private struct StateTreeInfo
        {
            public int ExecutionContext;
            public int TopLevelSessionState;
            public int EngineSessionState;
            public int GlobalScope;
            public int ModuleScope;
            public int ScriptScope;
            public int ParentScope;
            public int CurrentScope;
        }

        private const BindingFlags BINDING_FLAGS = BindingFlags.Instance | BindingFlags.NonPublic;

        protected override void ProcessRecord()
        {
            StateTreeInfo result;
            EngineIntrinsics engine = GetVariableValue("ExecutionContext") as EngineIntrinsics;
            var context = engine
                .GetType()
                .GetTypeInfo()
                .GetField("_context", BINDING_FLAGS)
                .GetValue(engine);

            var stateInternal = GetProperty("Internal", SessionState);
            var currentScope = GetProperty("CurrentScope", stateInternal);
            var parent = GetProperty("Parent", currentScope);
            
            result.ParentScope = 0;
            if (parent != null)
            {
                result.ParentScope = parent.GetHashCode();
            }

            result.ExecutionContext = context.GetHashCode();
            result.EngineSessionState = GetProperty("EngineSessionState", context).GetHashCode();
            result.TopLevelSessionState = GetProperty("TopLevelSessionState", context).GetHashCode();
            result.CurrentScope = currentScope.GetHashCode();
            result.ScriptScope = GetProperty("ScriptScope", stateInternal).GetHashCode();
            result.ModuleScope = GetProperty("ModuleScope", stateInternal).GetHashCode();
            result.GlobalScope = GetProperty("GlobalScope", stateInternal).GetHashCode();

            WriteObject(result);
        }

        private object GetProperty(string propertyName, object source)
        {
            if (string.IsNullOrEmpty(propertyName))
            {
                throw new ArgumentNullException("propertyName");
            }

            if (source == null)
            {
                throw new ArgumentNullException("source");
            }

            return source
                .GetType()
                .GetTypeInfo()
                .GetProperty(propertyName, BINDING_FLAGS)
                .GetValue(source);
        }
    }
}
'@

if (-not ('PSStateTree.Commands.ShowPSStateTreeCommand' -as [type])) {
    $module = New-Module -Name PSStateTree -ScriptBlock {
        $types = Add-Type -TypeDefinition $typeDefinition -PassThru
        Import-Module -Assembly $types[0].Assembly
        Export-ModuleMember -Cmdlet Show-PSStateTree
    }
    Import-Module $module
}
```