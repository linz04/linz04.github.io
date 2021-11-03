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

Untuk download challengenya kalian bisa download di [sini](https://google.com)<br>


## Setup
Didalam zip file sudah ada README.md bisa kalian baca, tapi disini saya akan jelaskan cara setup dan build untuk challengenya dari awal.
  1. Install depot_tools
  ```sh
  $ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
  $ echo "export PATH=/home/your_path/path/depot_tools:$PATH" >> ~/.bashrc
  $ source ~/.bashrc
  ```
  2. Next extract challengenya, dan dialamnya sudah ada README.md
  ```sh
  $ fetch v8 # https://chromium.googlesource.com/chromium/tools/depot_tools.git
  $ cd v8
  $ ./build/install-build-deps.sh
  $ git checkout 5fe0aa3bc79c0a9d3ad546b79211f07105f09585
  $ git apply path/to/maffs.patch # challange patch
  $ git apply path/to/v8_global.patch # disable built-in d8 global
  $ ./tools/dev/v8gen.py x64.release
  $ ninja -C ./out.gn/x64.release #Release version
  $ ./tools/dev/v8gen.py x64.debug
  $ ninja -C ./out.gn/x64.debug # Debug version
  ```

Untuk build v8 akan memakan waktu lumayan lama tergantung kecepatan koneksi kalian sama spec laptop kalian.
Untuk Debugging bisa menggunakan gdb
```sh
$ gdb -q d8
$ run --shell script.js
```
`--shell` disini agar v8 tidak langsung exit saat kita running script kita, untuk contoh script saya seperti ini
```js
function foo() {
    console.log("Hello World!\n");
}
foo();
```
![Test](/assets/images/quickmaffs/quick1.png)

**Now we are ready to HACK!**





## Patch Analysis
Sekarang kita baca patch `maffs.patch` yang ada di soal untuk yang `v8_global.patch` tidak ada yang perlu kita analisis.<br>
Kita fokus ke `maffs.patch`.

{% highlight diff linenos %}
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

Disini saya jelasin sedikit maksud patch diatas:
  - Line 1 code tersebut patch file `src/compiler/js-create-lowring.cc` untuk code yang di patch bisa diliat pada line 9-11 3 baris code di deleted.
  - The patch was patcehd code to `src/compiler/operation-typer.cc` we can see on line 15.

Can u guys see the bug?<br>
Sebelum patch typer set<br>
`Math.exp to be Union(Type::PlainNumber(), Type::NaN(), zone())` this mean the output will be either a PlainNumber or a NaN.
But after the patch, we can see on the patch above on line 24.
The typer set `Math.exp to be PlainNumber();` this mean the output will be always a PlainNumber.

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
