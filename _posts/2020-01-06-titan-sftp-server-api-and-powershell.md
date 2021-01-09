---
layout: post
title: Titan FTP server API and PowerShell
tags: [powershell]
---

So, your company uses Titan FTP and would like to automate things a bit? Or maybe you are fed up with clicking through the GUI? I bet that you googled your way through the Internet, and you know by now that the server has an API and documentation which describes it and that does not mention PowerShell at all?

I've been there. I tried to work with `srxCfg.exe` but, let's be honest, once you were spoiled by working with objects, you do not want to deal with text anymore. Nor do you want to call some external tool, like an animal.

The only option left is `srxCom`, a COM interface that lets you connect to the server, issue some commands and receive results. Thing is, the sample code does not include PowerShell. Hello, we're living in 2021, right?

There's this example in the docs, which looks kind of powershell-y:

```vb
Dim srxcom
Set srxcom = CreateObject("srxCom.SRXCornerstone") ' instantiate the object
srxcom.SRX_Connect("localhost",31000,"Administrator","MyPassword")
srxcom.SVR_Create("fred", "1.2.3.4", 12, "C:\fred", 0)
srxcom.SRX_Disconnect
```

Sigh, it's Visual Basic 6. Let's try to figure out how to translate this code to PS.

Creating an object is easy in Powershell:

```powershell
$object = New-Object -ComObject SrxCom.SRXCornerstone
New-Object : Retrieving the COM class factory for component with CLSID {00000000-0000-0000-0000-000000000000} failed due to the following error: 80040154 Class not registered (Exception from HRESULT: 0x80040154 (REGDB_E
_CLASSNOTREG)).
At line:1 char:11
+ $object = New-Object -ComObject SrxCom.SRXCornerstone
+      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  + CategoryInfo     : ResourceUnavailable: (:) [New-Object], COMException
  + FullyQualifiedErrorId : NoCOMClassIdentified,Microsoft.PowerShell.Commands.NewObjectCommand 

```

Wait, what? What does the documentation say again?

> srxCOM is a COM/Scripting interface that can be used to configure Servers, Groups, and Users. The scripting engine is installed with Titan FTP.
>
> Interface Name: srxCom.SRXTitan
>
> Implementation DLL: srxCom.dll

So, there's a bug in the docs, the name of the interface is <em>srxCom.SRXTitan</em> and not SrxCom.SRXCornerstone. Okaaaaay...

```powershell
$object = New-Object -ComObject SrxCom.SRXTitan 
```

The object is ready[^1] Let’s connect to the server!

```powershell
[ref]$errorCode = 999 
$object.SRX_Connect2('127.0.0.1','12345','administrator','my_not_so_secret_password', $errorCode)
```

Before I’ll start explaining what have just happened here, let me show you something first:

```powershell
$errorCode

Value
-----
    0 
```

If you’re a bit confused, do not worry. It just means that you have never worked with [reference variables](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_ref?view=powershell-7.1) before.

Generally speaking, Powershell is a [dynamically typed programming language](https://eli.thegreenplace.net/2006/11/25/a-taxonomy-of-typing-systems/). It means that you can do:

```powershell
$Variable = 'test'
$Variable.GetType()

IsPublic IsSerial Name                  BaseType
-------- -------- ----                  --------
Yes      Yes      String                System.Object
```
You just declared a variable of type string without even thinking about it. PowerShell figured it out for you, in the background[^2].

But wait! PowerShell can be also a statically typed language:

```powershell
[string]$Variable = 'test'
```

One of the types of objects available in Powershell is `System.Management.Automation.PSReference ` or `[ref]`, for short. This is a special kind of object which, when passed to a function, can be modified by it <em>regardless of the type of data passed</em>[^3]. You can think of COM interfaces as a type of function that behaves a  bit strange. You can’t for example do this:

```powershell
$Variable = $object.SRX_Connect('127.0.0.1', '12345', 'administrator', 'my_not_so_secret_password_wrong_password')
```

You’d expect that variable `$Variable` will store a result of the connection attempt. Nope.

```powershell
$Variable.GetType()
You cannot call a method on a null-valued expression.
At line:1 char:1
+ $Variable.GetType()
+ ~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) [], RuntimeException
    + FullyQualifiedErrorId : InvokeMethodOnNull
```
  
As you can see, the value of your variable is `NULL` which means that the interface did not return any value. That’s why we have a variant of `SRX_Connect`: `SRX_Connect2`. The only difference is that this variant has an additional parameter: you can specify a variable which will be used to store exit code of the operation.

OK, what happens when you try to pass a regular variable to it? You can even guess that the code should be `[int]` and statically set the type:

```powershell
[int]$errorCode = 999 
$object.SRX_Connect2('127.0.0.1', '12345', 'administrator', 'my_not_so_secret_password', $errorCode)
```

What’s the result?

```powershell
$errorCode
999
```

You see? The value of the variable was not modified but this operation returned an exit code, you just did not get it.

That’s exactly why we have to go with this version:

```powershell
[ref]$errorCode = 999 
$object.SRX_Connect2('127.0.0.1', '12345', 'administrator', 'my_not_so_secret_password', $errorCode)
```

Drumroll, please.

```powershell
$errorCode

Value
---- -
0
```

The value returned by the interface became a `value` property of the object passed to COM. Hence, our object was modified![^4]

Well, that’s really everything there’s to it. The hardest part is to understand how to work with the interface using PowerShell. Once you grasp that, it’s just a matter of using all the other available methods.

Actually, there’s one more thing. The results are returned either as an integer or as a string. If multiple string values are returned, they are concatenated using „\|” character. Like this:

```powershell
$listOfservers

Value                 
-----                 
|test|test1|test2|
```

Thankfully, we have a `split` method available for strings.

```powershell
$listOfservers.Value.Split('|') 

test
test1
test2
```


[^1]: If you see an error that the COM class is not registered, you need to register the DLL with `regsrv C:\Windows\System32\srxCom.dll`
[^2]: Such „automation” has its costs of course and it leads sometimes to funny results.
[^3]: More on this [here](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_ref?view=powershell-7.1)
[^4]: I am sorry for taking you from New York to Newark via Tokio (metaphorically speaking) but sometimes the longer route has a higher ROI than the regular one (metaphorically speaking again).