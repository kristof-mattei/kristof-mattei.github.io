---
author: Kristof
comments: true
date: 2014-03-04 20:52:00+00:00
layout: post
slug: dictionary-removing-adding-items-order
title: About a dictionary, removing and adding items, and their order.
wordpress_id: 2105
categories:
- .NET
- C#
- Programming
---

I had a weird problem today using a [Dictionary](http://msdn.microsoft.com/en-us/library/xfhwa508(v=vs.110).aspx). The process involved removing and adding data, and then printing the data. I assumed that it was ordered. I was wrong! Let me show you:

```csharp
var dictionary = new Dictionary<int, string>();

dictionary.Add(5, "The");
dictionary.Add(7, "quick");
dictionary.Add(31, "brown");
dictionary.Add(145, "fox");

dictionary.Remove(7); // remove the "quick" entry
```

After a while I added another line to the dictionary:

```csharp
dictionary.Add(423, "jumps");
```

While printing this data I discovered an oddity.

```csharp
dictionary
    .ToList()
    .ForEach(e => Console.WriteLine("{0} => {1}", e.Key, e.Value));
```

What do you expect the output of this to be?

```csharp
5 => The
31 => brown
145 => fox
423 => jumps
```

However the actual result was this:

```csharp
5 => The
423 => jumps
31 => brown
145 => fox
```

The [documentation](http://msdn.microsoft.com/en-us/library/xfhwa508(v=vs.110).aspx) tells us the following:


<blockquote>For purposes of enumeration, each item in the dictionary is treated as a KeyValuePair<TKey, TValue> structure representing a value and its key. The order in which the items are returned is undefined.</blockquote>


Interested in the actual behavior I looked at the source code of Dictionary [here](http://referencesource-beta.microsoft.com/#mscorlib/system/collections/generic/dictionary.cs#d3599058f8d79be0).

If you look closely, first at [`Remove`](http://referencesource-beta.microsoft.com/#mscorlib/system/collections/generic/dictionary.cs#a6db5ffdec557169) and then to [`Add`](http://referencesource-beta.microsoft.com/#mscorlib/system/collections/generic/dictionary.cs#a7861da7aaa500fe) (and subsequently [`Insert`](http://referencesource-beta.microsoft.com/#mscorlib/system/collections/generic/dictionary.cs#fd1acf96113fbda9)) you can see that when you remove an item it holds a reference (in [`freelist`](http://referencesource-beta.microsoft.com/#mscorlib/system/collections/generic/dictionary.cs#998e5f475d87f454)) to the free 'entry'.

What's more weird is the behavior when you delete 2 entries, and then add 2 others:

```csharp
var dictionary = new Dictionary<int, string>();

dictionary.Add(5, "The");
dictionary.Add(7, "quick");
dictionary.Add(31, "brown");
dictionary.Add(145, "fox");

dictionary.Remove(7); // remove the "quick" entry
dictionary.Remove(31); // also remove the "brown" entry

dictionary.Add(423, "jumps");
dictionary.Add(534, "high");

dictionary
    .ToList()
    .ForEach(e => Console.WriteLine("{0} => {1}", e.Key, e.Value));
```

Which yields:

```csharp
5 => The
534 => high
423 => jumps
145 => fox
```

But for that you'll need to look at [line 340](http://referencesource-beta.microsoft.com/#mscorlib/system/collections/generic/dictionary.cs#340) and further!

So what have we learned? It's not ordered until MSDN tells you!

Have a good one!
