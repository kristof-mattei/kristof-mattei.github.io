+++
title = "Be careful with npm shrinkwrap"
date = 2015-11-17 08:32:26+00:00

[taxonomies]
categories = ["AWS", "Javascript"]

[extra]
author = "Kristof"
+++

Recently we had the issue that we used version x of an npm package. However in the course of time this package was updated by the author and contained a critical bug, which broke our AWS deployments. Locally this was no issue because we had a version installed that satisfied the version requirements.

In order to prevent issues like this we looked at [npm shrinkwrap](https://docs.npmjs.com/cli/shrinkwrap). This writes a file called `npm-shrinkwrap.json` which 'freezes' all of the package-versions that are installed in current project.

Now this is dangerous, as we just found out.

The issue is that when an author decides to delete a package from the feed. Delete you ask? Yes, delete, gone, no trace, nothing.

What's the issue you might ask? You'll find it next time?

Not really.

Imagine you're using Elastic Beanstalk, which, based on certain triggers, can spawn a new instance of a server, or delete one.

Now today you release your application to your servers, and you `shrinkwrap` your packages.

Before you do that, you obviously clear your local npm cache (located in `%appdata%\npm-cache`), and your local `node_modules`. Then you do an `npm install` to verify every package is correctly installed, you do a few test runs, maybe on a local server. Then you package and send it of to AWS.

All runs well, you're happy, and your boss is happy.

Next week, for whatever reason, you get a high load on your servers. Elastic Beanstalk decides to add one more instance.

And then stuff starts to break. You get emails that its health is degraded. Then you get emails that its health severe.

At 2a.m. you open your laptop, and you start looking at the logs. There you find something in the lines of:

```
npm ERR! Linux 3.14.48-33.39.amzn1.x86_64
npm ERR! argv "/usr/bin/iojs" "/usr/bin/npm" "install"
npm ERR! node v2.4.0
npm ERR! npm  v2.13.0

npm ERR! version not found: node-uuid@1.4.4
npm ERR!
npm ERR! If you need help, you may report this error at:
npm ERR!     <https://github.com/npm/npm/issues>

npm ERR! Please include the following file with any support request:
npm ERR!     /app/npm-debug.log
```

What? You tested locally? What happened?

Okay, you fire up a console window. You make a test dir. You run `npm installnode-uuid@1.4.4`.

All goes well. Or does it?

Let's look at the output:

```
C:\__SOURCES>mkdir test

C:\__SOURCES>cd test

C:\__SOURCES\test>npm install node-uuid@1.4.4
npm http GET https://registry.npmjs.org/node-uuid/1.4.4
npm http 404 https://registry.npmjs.org/node-uuid/1.4.4
node-uuid@1.4.4 node_modules\node-uuid
```

Notice the `404`? I didn't... But it's important!

Now here's what happened: locally I had `node-uuid@1.4.4` in my cache, so he took that one, even though the package disappeared from the registry.

However: my new instance on Elastic Beanstalk didn't. That's why it failed.

So, solutions:

- Be careful when you `shrinkwrap`. Stuff might break in the future, as authors delete packages
- Create a private feed, that you curate
- As a package author, don't delete packages. Just don't. Other people might depend on you.
