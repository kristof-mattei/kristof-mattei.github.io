---
author: Kristof
comments: true
date: 2014-11-28 09:40:41+00:00
layout: post
slug: frameworks-try-smart-angularjs-expressions
title: When frameworks try to be smart, AngularJS & Expressions
categories:
- AngularJS
- HTML
- Javascript
- Programming
---

One of my colleagues just discovered this bug/feature in AngularJS. Using an [ngIf](https://docs.angularjs.org/api/ng/directive/ngIf) on a string `"no"` will result in `false`.

HTML:

```html
<div ng-app>
    <div ng-controller="yesNoController">
        <div ng-if="yes">Yes is defined, will display</div>
        <div ng-if="no">No is defined, but will not display on Angular 1.2.1</div>
        <div ng-if="notDefined">Not defined, will not display</div>
    </div>
</div>
```


JavaScript:

```javascript
function yesNoController($scope) {
    $scope.yes = "yes";
    $scope.no = "no";
    $scope.notDefined = undefined;
}
```

Will print:

    
```
Yes is defined, will display
```
    


Let's read the documentation on [expression](https://docs.angularjs.org/guide/expression), to see where this case is covered.

.

.

.

.

Can you find it?

Neither can I.

Fix?

Use the double bang:

```
// ...
<div ng-if="!!no">No is defined, but we need to add a double bang for it to parse correctly</div>
// ...
```

JSFiddle can be found [here](http://jsfiddle.net/t56onm8o/5/).

For those who care, it's not a JS thing:

``` 
// execute this line in a console
alert("no" ? "no evaluated as true" : "no evaluated as false"); // will alert "no evaluated as true"
```
