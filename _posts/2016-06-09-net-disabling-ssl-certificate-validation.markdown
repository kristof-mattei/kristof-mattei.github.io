---
author: Kristof
date: 2016-06-09 14:17:39+00:00
layout: post
title: '.NET: Disabling SSL Certificate validation'
categories:
- .NET
- F#
- Programming
---

Yesterday I wanted to download some content off a website with F#, however unfortunately the certificate of the website was expired.

```
let result = 
    try
        let request = 
            "https://somewebsite/with/expired/ssl/certificate/data.json?paramx=1&paramy=2"
            |> WebRequest.Create

        let response = 
            request.GetResponse ()

        // parse data
        let parsed = "..." 

        Ok parsed
    with
    | ex ->      
        Error ex
```



If we execute this, then result would be of Error with the following exception:

{% include image.html name="remote_certificate_invalid.png" caption="SSL certificate validation exception" %}

```
ex.Message
"The underlying connection was closed: Could not establish trust relationship for the SSL/TLS secure channel."
ex.InnerException.Message
"The remote certificate is invalid according to the validation procedure."
```

So how do we fix this?

The solution is to set the following code at startup of the application (or at least before the first call):

    ServicePointManager.ServerCertificateValidationCallback <- 
        new RemoteCertificateValidationCallback(fun _ _ _ _ -> true)


Notice that you should not do this, because this does not validate the certificate at ALL!
Also, this is for ALL calls, if you want to do it on a specific call you need to do make some changes.

First of all, it doesn't work with [`WebRequest.Create`](https://msdn.microsoft.com/en-us/library/bw00b1dc(v=vs.110).aspx), you need to use [`WebRequest.CreateHttp`](https://msdn.microsoft.com/en-us/library/ff382788(v=vs.110).aspx), or cast the [`WebRequest`](https://msdn.microsoft.com/en-us/library/system.net.webrequest(v=vs.110).aspx) to [`HttpWebRequest`](https://msdn.microsoft.com/en-us/library/system.net.httpwebrequest(v=vs.110).aspx), as the property we need, [`ServerCertificateValidationCallback`](https://msdn.microsoft.com/en-us/library/system.net.httpwebrequest.servercertificatevalidationcallback(v=vs.110).aspx) is not available on [`WebRequest`](https://msdn.microsoft.com/en-us/library/system.net.webrequest(v=vs.110).aspx), only on [`HttpWebRequest`](https://msdn.microsoft.com/en-us/library/system.net.httpwebrequest(v=vs.110).aspx). The resulting code looks like this:

```
let request = 
    "https://somewebsite/with/expired/ssl/certificate/data.json?paramx=1&paramy=2"
    |> WebRequest.CreateHttp

request.ServerCertificateValidationCallback <- new RemoteCertificateValidationCallback(fun _ _ _ _ -> true)

let response = 
    request.GetResponse ()
```
    
Again, don't do this in production!

If need be, do it on a single [`HttpWebRequest`](https://msdn.microsoft.com/en-us/library/system.net.httpwebrequest(v=vs.110).aspx), like the last example, and write some code so that you ignore the expiration part, but leave in place the validation part.

Code on [Github](https://github.com/kristof-mattei/dont-validate-ssl)!
