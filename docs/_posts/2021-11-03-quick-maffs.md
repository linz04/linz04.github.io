---
layout: post
title: Cyber Jawara Quick Maffs
subtitle: Typer Bug v8
categories: CTF
tags: [writeups, v8]
---


## Description
> 1 + 1 = 2, e^2 that's 9? maybe? idk.\
> Author: circleous#0587

You can download the challenge archive from [here](https://google.com)
And for build the challenge, you can read the README.md


## Patch Analysis
First, lets see the patch file. We dont need to understand every single line of code, but we should be able to find the vuln.

{% highlight javascript linenos %}
diff --git a/src/compiler/js-create-lowering.cc b/src/compiler/js-create-lowering.cc
index 50a523c606..0b46267c4f 100644
--- a/src/compiler/js-create-lowering.cc
+++ b/src/compiler/js-create-lowering.cc
@@ -683,9 +683,6 @@ Reduction JSCreateLowering::ReduceJSCreateArray(Node* node) {
         length_type.Max() <= kElementLoopUnrollLimit &&
         length_type.Min() == length_type.Max()) {
       int capacity = static_cast<int>(length_type.Max());
-      // Replace length with a constant in order to protect against a potential
-      // typer bug leading to length > capacity.
-      length = jsgraph()->Constant(capacity);
       return ReduceNewArray(node, length, capacity, *initial_map, elements_kind,
                             allocation, slack_tracking_prediction);
     }
diff --git a/src/compiler/operation-typer.cc b/src/compiler/operation-typer.cc
index e7dc51bd2d..121a7a7107 100644
--- a/src/compiler/operation-typer.cc
+++ b/src/compiler/operation-typer.cc
@@ -411,7 +411,7 @@ Type OperationTyper::NumberCosh(Type type) {
 
 Type OperationTyper::NumberExp(Type type) {
   DCHECK(type.Is(Type::Number()));
-  return Type::Union(Type::PlainNumber(), Type::NaN(), zone());
+  return Type::PlainNumber();
 }
 
 Type OperationTyper::NumberExpm1(Type type) {
{% endhighlight %}

We can see on the patch above, line 23 and 24
The typer set `Math.exp to be PlainNumber();`

And here is the same code with syntax highlighting:

```javascript
var foo = function(x) {
  return(x + 5);
}
foo(3)
```

And here is the same code yet again but with line numbers:

{% highlight javascript linenos %}
var foo = function(x) {
  return(x + 5);
}
foo(3)
{% endhighlight %}

## Boxes
You can add notification, warning and error boxes like this:

### Notification

{: .box-note}
**Note:** This is a notification box.

### Warning

{: .box-warning}
**Warning:** This is a warning box.

### Error

{: .box-error}
**Error:** This is an error box.
