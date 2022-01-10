---
author: Kristof
comments: true
date: 2015-01-15 15:10:27+00:00
layout: post
slug: topshelf-install-powershell-get-credentials
title: Topshelf install, PowerShell and Get-Credentials
categories:
- Build automation
- Powershell
---

In the project I'm currently working at we use PowerShell script for configuration and build execution.

This means that if you get a new laptop, or a new member joins the team, or even when you need to change your Windows password, you just need to run the script again and it will set up everything in the correct locations & with the correct credentials.

The credentials were a problem though.

When installing a Topshelf service with the `--interactive` parameter (we need to install under the current user, not `System`) it will prompt you for your credentials for each service you want to install. For one, it's fine, for 2, it's already boring, for 3, ... You get the point.

We initially used the following command line to install the services:

    
```powershell
. $pathToServiceExe --install --interactive --autostart
```

To fix this we will give the `$pathToServiceExe` the username and password ourselves with the `-username` and `-password`. We should also omit the `--interactive`.

First gotcha here: When reading the [documentation](http://topshelf.readthedocs.org/en/latest/overview/commandline.html), it says one must specify the commands in this format:

```powershell    
. $pathToServiceExe --install --autostart -username:username -password:password
```

However, this is not the case. You mustn't separate the command line argument and the value with a `:`.

Now, we don't want to hardcode the username & password file in the setup script.

So let's get the credentials of the current user:

   
```powershell
$credentialsOfCurrentUser = Get-Credential -Message "Please enter your username & password for the service installs"
```

Next up we should extract the username & password of the `$credentialsOfCurrentUser` variable, as we need it in clear-text (potential security risk!).

One can do this in 2 ways, either by getting the `NetworkCredential` from the `PSCredential` with `GetNetworkCredential()`:

```powershell
$networkCredentials = $credentialsOfCurrentUser.GetNetworkCredential();
$username = ("{0}\{1}") -f $networkCredentials.Domain, $networkCredentials.UserName # change this if you want the user@domain syntax, it will then have an empty Domain and everything will be in UserName. 
$password = $networkCredentials.Password
```    


Notice the `$username` caveat.

Or, by not converting it to a `NetworkCredential`:

    
```powershell
# notice the UserName contains the Domain AND the UserName, no need to extract it separately
$username = $credentialsOfCurrentUser.UserName

# little more for the password
$BSTR = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($credentialsOfCurrentUser.Password)
$password = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($BSTR)
```    

Notice the extra code to retrieve the `$password` in plain-text.

I would recommend combining both, using the `NetworkCredential` for the `$password`, but the regular `PSCredential` for the `$username` as then you're not dependent on how your user enters his username.

So the best version is:

```powershell
$credentialsOfCurrentUser = Get-Credential -Message "Please enter your username & password for the service installs" 
$networkCredentials = $credentialsOfCurrentUser.GetNetworkCredential();
$username = $credentialsOfCurrentUser.UserName
$password = $networkCredentials.Password
```    


Now that we have those variables we can pass them on to the install of the Topshelf exe:

    
```powershell
. $pathToServiceExe install -username `"$username`" -password `"$password`" --autostart
```    


Notice the backticks (\`) to ensure the double quotes are escaped.

In this way you can install all your services and only prompt your user for his credentials once!
