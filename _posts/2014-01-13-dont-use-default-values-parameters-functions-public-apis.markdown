---
author: Kristof
comments: true
date: 2014-01-13 20:39:02+00:00
layout: post
slug: dont-use-default-values-parameters-functions-public-apis
title: Default values and overloads are not the same!
categories:
- .NET
- C#
- Programming
---

Consider the following class of the Awesome(r) library, using default parameters.


```csharp
public class Foo
{
    public void DoCall(int timeout = 10)
    {
        /* awesome implementation goes here */
    }
}
```

You get the dll and that class & function in your code, like this:


```csharp
Foo foo = new Foo();

foo.DoCall();
```


Can't get much easier than this right?

Then the Awesome(r) library gets updated:

```csharp
public class Foo
{
    public void DoCall(int timeout = 20)
    {
        /* awesome implementation goes here */
    }
}
```


Notice that the default value has changed. You assume that when you just overwrite the dll in production, you will adopt the new behavior.

Nop. You need to recompile. Let me show you: the problem with default values is that the developer of Awesome(r) library is no longer in control of it.

Let's take a look at an excerpt of the IL where we create a new `Foo` and **call** `DoCall` without specifying `timeout`:

```csharp
IL_0000:  newobj     instance void AwesomeLibrary.Foo::.ctor()
IL_0005:  stloc.0
IL_0006:  ldloc.0
IL_0007:  ldc.i4.s   10
IL_0009:  callvirt   instance void AwesomeLibrary.Foo::DoCall(int32)
IL_000e:  ret
```

This is a release build.

Notice how on line 4 value 10 gets pushed on the the stack, and the next line calls the `DoCall`. 

This is a big danger in public APIs, and this is why the developer of Awesome(r) library should have used an overload instead of a default parameter:

    
```csharp
public class Foo
{
    public void DoCall()
    { 
        this.DoCall(20); 
    }

    public void DoCall(int timeout)
    {
        /* awesome implementation goes here */
    }
}
```

This ensures that when a new version of Awesome(r) library is released AND that if that release is backwards API compatible, it can just be dropped in, without you having to recompile your whole codebase (but you should still test it :P )
