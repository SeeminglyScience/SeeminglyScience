---
layout: post
title: "Formatting Objects without XML"
description: "Generate ps1xml files from PSControl objects."
category: "PowerShell"
tags: ["PowerShell", "Formatting", "PSControl"]
---
{% include JB/setup %}

Custom formatting in PowerShell has always seemed like one of the most under utilized features of PowerShell to me.
And I understand why. It feels kind of bizarre to spend all this time writing PowerShell, creating a cool custom
object and then jumping into...XML.

The bad news is, what I'm about to tell you is still going to involve XML.  The good news, you don't
have to write it.

## PSControl Classes

One of the things I love the most about PowerShell is the pure discoverability it provides.  You can
usually figure out what you need to do with just about any command, object, etc.  That all gets a little
abstracted when you're working through XML.

The PSControl object (and more importantly it's child classes) give that discoverability back.  Let's
pipe `TableControl` to `Get-Member` so you can see what I mean.

```raw
PS C:\> [System.Management.Automation.TableControl] | Get-Member -Static

   TypeName: System.Management.Automation.TableControl

Name            MemberType Definition
----            ---------- ----------
Create          Method     static System.Management.Automation.Table...
Equals          Method     static bool Equals(System.Object objA, Sy...
new             Method     System.Management.Automation.TableControl...
ReferenceEquals Method     static bool ReferenceEquals(System.Object...
```

Let's take a closer look at the `Create` static method.
<!--more-->
```raw
PS C:\> [System.Management.Automation.TableControl]::Create

OverloadDefinitions
-------------------
static System.Management.Automation.TableControlBuilder Create(bool outOfBand, bool autoSize, bool hideTableHeaders)
```

Now, I know I just went on about discoverability, but what this is leaving out is all of these parameters are
optional. So all we actually need to do is run `Create()`. Also, you may have noticed that returns a
different type, so let's see what options it has.

```raw
PS C:\> [System.Management.Automation.TableControl]::Create() | Get-Member

   TypeName: System.Management.Automation.TableControlBuilder

Name               MemberType Definition
----               ---------- ----------
AddHeader          Method     System.Management.Automation.TableCont...
EndTable           Method     System.Management.Automation.TableCont...
GroupByProperty    Method     System.Management.Automation.TableCont...
GroupByScriptBlock Method     System.Management.Automation.TableCont...
StartRowDefinition Method     System.Management.Automation.TableRowD...
```

So if you were to expand the definitions you would see a few of the methods return a copy of themselves,
meaning they are meant to be chained together.  We also have a new builder, `TableRowDefinitionBuilder`.

I'm sure you get the idea by now, so I'm going to skip ahead a little and show you an example I made
for an existing type.

```powershell
# For System.Reflection.RuntimeParameterInfo
[System.Management.Automation.TableControl]::Create().
    GroupByProperty('Member', $null, 'Definition').
    AddHeader('Left',   3,  '#').
    AddHeader('Left',   30, 'Type').
    AddHeader('Left',   20, 'Name').
    AddHeader('Center', 2,  'In').
    AddHeader('Center', 3,  'Out').
    AddHeader('Center', 3,  'Opt').
    StartRowDefinition($false).
        AddPropertyColumn('Position').
        AddScriptBlockColumn('$_.ParameterType.Name').
        AddPropertyColumn('Name').
        AddScriptBlockColumn('if ($_.IsIn) { ''X'' }').
        AddScriptBlockColumn('if ($_.IsOut) { ''X'' }').
        AddScriptBlockColumn('if ($_.IsOptional) { ''X'' }').
    EndRowDefinition().
EndTable()
```

If you've ever dug into reflection you know most of it is completely unformatted.  For example, here
is what the `TableControl.Create()` method looks like.

```powershell
# Before (this is just *one* parameter)
PS C:\> [System.Management.Automation.TableControl].GetMethod('Create').GetParameters()


ParameterType    : System.Boolean
Name             : outOfBand
HasDefaultValue  : True
DefaultValue     : False
RawDefaultValue  : False
MetadataToken    : 134234830
Position         : 0
Attributes       : Optional, HasDefault
Member           : System.Management.Automation.TableControlBuilder Create(Boolean, Boolean, Boolean)
IsIn             : False
IsOut            : False
IsLcid           : False
IsRetval         : False
IsOptional       : True
CustomAttributes : {[System.Runtime.InteropServices.OptionalAttribute()]}

# After
PS C:\> [System.Management.Automation.TableControl].GetMethod('Create').GetParameters()

   Definition: System.Management.Automation.TableControlBuilder Create(Boolean, Boolean, Boolean)

#   Type                           Name                 In Out Opt
-   ----                           ----                 -- --- ---
0   Boolean                        outOfBand                    X
1   Boolean                        autoSize                     X
2   Boolean                        hideTableHeaders             X
```

## Actually loading it

So you ran my example and all it did was return an object.  Now what?

Well, we need to wrap it in an object that tells the formatter what types to target and what to name
our view. Combine this with the `Export-FormatData` and `Update-FormatData` cmdlets to load it into
the session.

```powershell
using namespace System.Management.Automation

[ExtendedTypeDefinition]::new(
    'System.Reflection.ParameterInfo',
    [FormatViewDefinition]::new(
        'MyParameterView',
        [TableControl]::Create().
            GroupByProperty('Member', $null, 'Definition').
            AddHeader('Left',   3,  '#').
            AddHeader('Left',   30, 'Type').
            AddHeader('Left',   20, 'Name').
            AddHeader('Center', 2,  'In').
            AddHeader('Center', 3,  'Out').
            AddHeader('Center', 3,  'Opt').
            StartRowDefinition($false).
                AddPropertyColumn('Position').
                AddScriptBlockColumn('$_.ParameterType.Name').
                AddPropertyColumn('Name').
                AddScriptBlockColumn('if ($_.IsIn) { ''X'' }').
                AddScriptBlockColumn('if ($_.IsOut) { ''X'' }').
                AddScriptBlockColumn('if ($_.IsOptional) { ''X'' }').
            EndRowDefinition().
        EndTable()
    ) -as [List[FormatViewDefinition]]
) | ForEach-Object {

    Export-FormatData -Path        ".\$($PSItem.TypeName).ps1xml" `
                      -InputObject $PSItem `
                      -IncludeScriptBlock `
                      -Force
    # Use -PrependPath for existing types, -AppendPath for custom ones.
    Update-FormatData -PrependPath ".\$($PSItem.TypeName).ps1xml"
}
```

I highly recommend exploring the objects with `Get-Member` or diving into the [MSDN documentation](https://msdn.microsoft.com/en-us/library/system.management.automation.pscontrol(v=vs.85).aspx) for each class.

## Getting Complex

So that was a small example of some pretty basic formatting. There are classes for all of the formatting types:
`ListControl`, `WideControl`, `CustomControl` and of course `TableControl`.  The one you'll probably
use the most for general formatting is `TableControl`

But if you need *really* precise control over your output, you want `CustomControl`.

For example, I've been looking for a way to build flexible string expressions for generating code in editor commands.
I've been playing with the idea of using formatting for this because it's really easy to build dynamic
statements, and you can easily customize it by adding your own view.

Here is a really early draft of some controls that take a `[type]` object and "implements" any abstract
or interface methods the type has.

```powershell

using namespace System.Management.Automation
using namespace System.Collections.Generic

$parameterControl = [CustomControl]::
    Create().
        StartEntry().
            AddScriptBlockExpressionBinding('", "', 0, 0, '$_.Position -ne 0').
            AddText('[').
            AddPropertyExpressionBinding('ParameterType').
            AddText('] $').
            AddPropertyExpressionBinding('Name').
        EndEntry().
    EndControl()

$methodControl = [CustomControl]::
    Create().
        StartEntry().
            AddNewline().
            AddText('[').
            AddPropertyExpressionBinding('ReturnType').
            AddText('] ').
            AddPropertyExpressionBinding('Name').
            AddText(' (').
            AddScriptBlockExpressionBinding(
                <# scriptBlock:         #> '$_.GetParameters()',
                <# enumerateCollection: #> 1,
                <# selectedByType:      #> 0,
                <# selectedByScript:    #> '$_.GetParameters().Count',
                <# customControl:       #> $parameterControl
            ).
            AddText(') {').
            AddNewline().
            StartFrame(4).
                AddText('throw [NotImplementedException]::new()').
                AddNewline().
            EndFrame().
            AddText('}').
            AddNewline().
        EndEntry().
    EndControl()

$classControl = [CustomControl]::
    Create().
        StartEntry().
            AddText('class MyClass : ').
            AddScriptBlockExpressionBinding('$_').
            AddText(' {').
            AddNewline().
            StartFrame(4).
                AddScriptBlockExpressionBinding(
                    <# scriptBlock:         #> '
                        if ($_.IsAbstract) {

                            $return = $_.DeclaredMethods.
                                Where{ $_.IsAbstract }

                        } elseif ($_.IsInterface) {
                            $return = $_.DeclaredMethods
                        }
                        $return
                    ',
                    <# enumerateCollection: #> $true,
                    <# selectedByType:      #> $null,
                    <# selectedByScript:    #> '
                        $_.IsInterface -or
                        ($_.IsAbstract -and
                        $_.DeclaredMethods.Where{ $_.IsAbstract })
                    ',
                    <# customControl:       #> $methodControl
                ).
            EndFrame().
            AddText('}').
        EndEntry().
    EndControl()


$formats = @(
    [ExtendedTypeDefinition]::new(
        'System.Reflection.RuntimeParameterInfo',
        [FormatViewDefinition]::new(
            'ParameterView',
            $parameterControl
        ) -as [List[FormatViewDefinition]]
    )
    [ExtendedTypeDefinition]::new(
        'System.Reflection.RuntimeMethodInfo',
        [FormatViewDefinition]::new(
            'MethodView',
            $methodControl
        ) -as [List[FormatViewDefinition]]
    )
    [ExtendedTypeDefinition]::new(
        'System.RuntimeType',
        [FormatViewDefinition]::new(
            'TypeView',
            $classControl
        ) -as [List[FormatViewDefinition]]
    )
)
```

And here it is in action.

```powershell
PS C:\> [System.Collections.IDictionary]

# This won't actually load because it missed a method, but you get the idea.
class MyClass : System.Collections.IDictionary {

    [System.Object] get_Item ([System.Object] $key) {
        throw [NotImplementedException]::new()
    }

    [System.Void] set_Item ([System.Object] $key, [System.Object]
    $value) {
        throw [NotImplementedException]::new()
    }

    [System.Collections.ICollection] get_Keys () {
        throw [NotImplementedException]::new()
    }

    [System.Collections.ICollection] get_Values () {
        throw [NotImplementedException]::new()
    }

    [System.Boolean] Contains ([System.Object] $key) {
        throw [NotImplementedException]::new()
    }

    [System.Void] Add ([System.Object] $key, [System.Object] $value) {
        throw [NotImplementedException]::new()
    }

    [System.Void] Clear () {
        throw [NotImplementedException]::new()
    }

    [System.Boolean] get_IsReadOnly () {
        throw [NotImplementedException]::new()
    }

    [System.Boolean] get_IsFixedSize () {
        throw [NotImplementedException]::new()
    }

    [System.Collections.IDictionaryEnumerator] GetEnumerator () {
        throw [NotImplementedException]::new()
    }

    [System.Void] Remove ([System.Object] $key) {
        throw [NotImplementedException]::new()
    }
}
```

## Final thoughts

There **is** a way to load this directly without writing it to XML, but it requires a *huge* amount
of reflection and isn't really consistant.  If anyone is looking for a project, a domain specific
language that does all this for you would be *really* cool.

Also, if you're looking for more examples, check out the [DefaultFormatters](https://github.com/PowerShell/PowerShell/tree/master/src/System.Management.Automation/commands/utility/FormatAndOutput/common/DefaultFormatters) folder in the PowerShell repo.
It's all in C#, but it should be pretty easy to translate.