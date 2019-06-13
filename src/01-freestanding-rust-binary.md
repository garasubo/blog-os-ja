# A Freestanding Rust Binary

私達自身のオペレーティングシステム(以下、OS)カーネルを作っていく最初のステップは標準ライブラリとリンクしない Rust の実行環境をつくることです。 これにより、基盤となる OS がないベアメタル上で Rust のコードを実行することができるようになります (訳注: Bare machine の日本語版 Wikipedia ページが見当たらなかったのでリンクを省いています)。

<!-- more -->

このブログの内容は [GitHub] 上で公開・開発されています。 何か問題や質問などがあれば issue をたててください (訳注: リンクは原文(英語)のものになります)。 この記事の完全なソースコードは[`post-01` ブランチ][post branch]にあります。

[GitHub ]: https://github.com/phil-opp/blog_os
[post branch]: https://github.com/phil-opp/blog_os/tree/post-01

<!-- toc -->

## 導入

OS カーネルを書くためには、いかなる OS の機能にも依存しないコードが必要となります。 つまり、スレッドやヒープメモリ、ネットワーク、乱数、標準出力、その他 OS による抽象化や特定のハードウェアを必要とする機能は使えません。 私達は自分自身で OS とそのドライバを書こうとしているので、これは理にかなっています。

これは [Rust の標準ライブラリ][Rust standard library]をほとんど使えないということを意味しますが、それでも私達が使うことのできる Rust の機能はたくさんあります。 例えば、[イテレータ][iterators]や[クロージャ][closures]、[パターンマッチング][pattern matching]、 [`Option`][option] や [`Result`][result] 型に[文字列フォーマット][string formatting]、そしてもちろん[所有権システム][ownership system]を使うことができます。 これらの機能により、[未定義動作][undefined behavior]や[メモリ安全性][memory safety]を気にせずに、高い水準で表現力豊かにカーネルを書くことができます。

[option]: https://doc.rust-lang.org/core/option/
[result]:https://doc.rust-lang.org/core/result/
[Rust standard library]: https://doc.rust-lang.org/std/
[iterators]: https://doc.rust-lang.org/book/ch13-02-iterators.html
[closures]: https://doc.rust-lang.org/book/ch13-01-closures.html
[pattern matching]: https://doc.rust-lang.org/book/ch06-00-enums.html
[string formatting]: https://doc.rust-lang.org/core/macro.write.html
[ownership system]: https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html
[undefined behavior]: https://www.nayuki.io/page/undefined-behavior-in-c-and-cplusplus-programs
[memory safety]: https://tonyarcieri.com/it-s-time-for-a-memory-safety-intervention

Rust で OS カーネルを書くには、基盤となる OS なしで動く実行環境をつくる必要があります。 そのような実行環境はフリースタンディング環境やベアメタルのように呼ばれます。

この記事では、フリースタンディングな Rust のバイナリをつくるために必要なステップを紹介し、なぜそれが必要なのかを説明します。 もし最小限の説明だけを読みたいのであれば **[概要](#概要)** まで飛ばしてください。

## 標準ライブラリの無効化

デフォルトでは、全ての Rust クレートは[標準ライブラリ][standard library]にリンクされています。標準ライブラリはスレッドやファイル、ネットワークのような OS の機能に依存しています。 また OS と密接な関係にある C の標準ライブラリ( `libc` )にも依存しています。 私達の目的は OS を書くことなので、 OS 依存のライブラリを使うことはできません。 そのため、 [`no_std` attribute] を使って標準ライブラリが自動的にリンクされるのを無効にします。

[standard library]: https://doc.rust-lang.org/std/
[`no_std` attribute]: https://doc.rust-lang.org/1.30.0/book/first-edition/using-rust-without-the-standard-library.html

新しい Cargo プロジェクトをつくるところから始めましょう。 もっとも簡単なやり方はコマンドラインで以下を実行することです。

```
cargo new blog_os --bin --edition 2018
```

プロジェクト名を `blog_os` としましたが、もちろんお好きな名前をつけていただいても大丈夫です。 `--bin`フラグは実行可能バイナリを作成することを、 `--edition 2018` は[2018エディション][2018 edition]を使用することを明示的に指定します。 コマンドを実行すると、 Cargoは以下のようなディレクトリ構造を作成します:

[2018 edition]: https://rust-lang-nursery.github.io/edition-guide/rust-2018/index.html

```
blog_os
├── Cargo.toml
└── src
    └── main.rs
```

`Cargo.toml` にはクレートの名前や作者名、[セマンティックバージョニング][semantic version]に基づくバージョンナンバーや依存関係などが書かれています。 `src/main.rs` には私達のクレートのルートモジュールと `main` 関数が含まれています。 `cargo build` コマンドでこのクレートをコンパイルして、 `target/debug` ディレクトリの中にあるコンパイルされた `blog_os` バイナリを実行することができます。

[semantic version]: http://semver.org/

### The `no_std` Attribute

今のところ私達のクレートは暗黙のうちに標準ライブラリをリンクしています。 [`no_std` attribute]を追加してこれを無効にしてみましょう:

```rust
// main.rs

#![no_std]

fn main() {
    println!("Hello, world!");
}
```

(`cargo build` を実行して)ビルドしようとすると、次のようなエラーが発生します:

```
error: cannot find macro `println!` in this scope
 --> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^
```

これは [`println` マクロ][`println` macro]が標準ライブラリに含まれているためです。 `no_std` で標準ライブラリを無効にしたので、何かをプリントすることはできなくなりました。 `println` は標準出力に書き込むのでこれは理にかなっています。 [標準出力][standard output]は OS によって提供される特別なファイル記述子であるためです。

[`println` macro]: https://doc.rust-lang.org/std/macro.println.html
[standard output]: https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29

では、 `println` を削除し `main` 関数を空にしてもう一度ビルドしてみましょう:

```rust
// main.rs

#![no_std]

fn main() {}
```

```
> cargo build
error: `#[panic_handler]` function required, but not found
error: language item required, but not found: `eh_personality`
```

この状態では `#[panic_handler]` 関数と `language item` がないというエラーが発生します。

## Panic の実装

`panic_handler` attribute は[パニック]が発生したときにコンパイラが呼び出す関数を定義します。 標準ライブラリには独自のパニックハンドラー関数がありますが、 `no_std` 環境では私達の手でそれを実装する必要があります:

[パニック]: https://doc.rust-lang.org/stable/book/ch09-01-unrecoverable-errors-with-panic.html

```rust
// in main.rs

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

[`PanicInfo` パラメータ]には、パニックが発生したファイルと行、およびオプションでパニックメッセージが含まれます。 この関数は戻り値を取るべきではないので、]"never" 型(`!`)][“never” type]を返すことで[発散する関数][diverging function]となります。 今のところこの関数でできることは多くないので、無限にループするだけです。

[`PanicInfo` パラメータ]: https://doc.rust-lang.org/nightly/core/panic/struct.PanicInfo.html
[diverging function]: https://doc.rust-lang.org/1.30.0/book/first-edition/functions.html#diverging-functions
[“never” type]: https://doc.rust-lang.org/nightly/std/primitive.never.html

## The `eh_personality` Language Item

language item はコンパイラによって内部的に必要とされる特別な関数や型です。 例えば、[`Copy`] トレイトはどの型が[コピーセマンティクス][`Copy`]を持っているかをコンパイラに伝える language item です。 [実装][copy code]を見てみると、 language item として定義されている特別な `#[lang = "copy"]` attribute を持っていることが分かります。

[`Copy`]: https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html
[copy code]: https://github.com/rust-lang/rust/blob/485397e49a02a3b7ff77c17e4a3f16c653925cb3/src/libcore/marker.rs#L296-L299

独自に language item を実装することもできますが、これは最終手段として行われるべきでしょう。 というのも、language item は非常に不安定な実装であり型検査も行われないからです(なので、コンパイラは関数が正しい引数の型を取っているかさえ検査しません)。 幸い、上記の language item のエラーを修正するためのより安定した方法があります。

`eh_personality` language item は[スタックアンワインド][stack unwinding] を実装するための関数を定義します。 デフォルトでは、パニックが起きた場合には Rust はアンワインドを使用してすべてのスタックにある変数のデストラクタを実行します。 これにより、使用されている全てのメモリが確実に解放され、親スレッドはパニックを検知して実行を継続できます。 しかしアンワインドは複雑であり、いくつかの OS 特有のライブラリ(例えば、Linux では [libunwind] 、Windows では[構造化例外][structured exception handling])を必要とするので、私達の OS には使いたくありません。

[stack unwinding]: http://www.bogotobogo.com/cplusplus/stackunwinding.php
[libunwind]: http://www.nongnu.org/libunwind/
[structured exception handling]: https://msdn.microsoft.com/en-us/library/windows/desktop/ms680657(v=vs.85).aspx

### アンワインドの無効化

他にもアンワインドが望ましくないユースケースがあります。 そのため、Rust には代わりに[パニックで中止する][abort on panic]オプションがあります。 これにより、アンワインドのシンボル情報の生成が無効になり、バイナリサイズが大幅に削減されます。 アンワインドを無効にする方法は複数あります。 もっとも簡単な方法は、`Cargo.toml` に次の行を追加することです:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

これは dev プロファイル(`cargo build` に使用される)と release プロファイル(`cargo build --release` に使用される)の両方でパニックで中止するようにするための設定です。 これで `eh_personality` language item が不要になりました。

[abort on panic]: https://github.com/rust-lang/rust/pull/32900

これで上の2つのエラーを修正しました。 しかし、コンパイルしようとすると別のエラーが発生します:

```
> cargo build
error: requires `start` lang_item
```

私達のプログラムにはエントリポイントを定義する `start` language item がありません。

## The `start` attribute

`main` 関数はプログラムを実行したときに最初に呼び出される関数であると思うかもしれません。 しかし、ほとんどの言語には[ランタイムシステム][runtime system]があり、これはガベージコレクション(Java など)やソフトウェアスレッド(Go のゴルーチン)などを処理します。 ランタイムは自身を初期化する必要があるため、`main` 関数の前に呼び出す必要があります。 これにはスタック領域の作成と正しいレジスタへの引数の配置が含まれます。

[runtime system]: https://en.wikipedia.org/wiki/Runtime_system

標準ライブラリをリンクする一般的な Rust バイナリでは、`crt0` ("C runtime zero")と呼ばれる C のランタイムライブラリで実行が開始され、C アプリケーションの環境が設定されます。 その後 C ランタイムは、`start` language item で定義されている [Rust ランタイムのエントリポイント][rt::lang_start]を呼び出します。 Rust にはごくわずかなランタイムしかありません。 これは、スタックオーバーフローを防ぐ設定やパニック時のバックトレースの表示など、いくつかの小さな処理を行います。最後に、ランタイムは `main` 関数を呼び出します。

[rt::lang_start]: https://github.com/rust-lang/rust/blob/bb4d1491466d8239a7a5fd68bd605e3276e97afb/src/libstd/rt.rs#L32-L73

私達のフリースタンディングな実行可能ファイルは今のところ Rust ランタイムと `crt0` へアクセスできません。なので、私達は自身でエントリポイントを定義する必要があります。 `start` language item を実装することは `crt0` を必要とするのでここではできません。代わりに `crt0` エントリポイントを直接上書きしなければなりません。

### エントリポイントの上書き

Rust コンパイラに通常のエントリポイントを使いたくないことを伝えるために、`#![no_main]` attribute を追加します。

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`main` 関数を削除したことに気付いたかもしれません。 `main` 関数を呼び出す基盤となるランタイムなしには置いていても意味がないからです。 代わりに、OS のエントリポイントを独自の `_start` 関数で上書きしていきます:

```rust
#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

Rust コンパイラが `_start` という名前の関数を実際に出力するように、`#[no_mangle]` attributeを用いて[名前修飾][name mangling]を無効にします。 この attribute がないと、コンパイラはすべての関数にユニークな名前をつけるために、 `_ZN3blog_os4_start7hb173fedf945531caE` のようなシンボルを生成します。 次のステップでエントリポイントとなる関数の名前をリンカに伝えるため、この属性が必要となります。

また、(指定されていない Rust の呼び出し規約の代わりに)この関数に [C の呼び出し規約][C calling convention]を使用するようコンパイラに伝えるために、関数を `extern "C"` として定義する必要があります。 `_start`という名前をつける理由は、これがほとんどのシステムのデフォルトのエントリポイント名だからです。

[name mangling]: https://en.wikipedia.org/wiki/Name_mangling
[C calling convention]: https://en.wikipedia.org/wiki/Calling_convention

戻り値の型である `!` は関数が発散している、つまり値を返すことができないことを意味しています。 エントリポイントはどの関数からも呼び出されず、OS またはブートローダから直接呼び出されるので、これは必須です。なので、値を返す代わりに、エントリポイントは例えば OS の [`exit` システムコール][`exit` system call]を呼び出します。 今回はフリースタンディングなバイナリが返されたときマシンをシャットダウンするようにすると良いでしょう。 今のところ、私達は無限ループを起こすことで要件を満たします。

[`exit` system call]: https://en.wikipedia.org/wiki/Exit_(system_call)

`cargo build` を実行すると、見づらいリンカエラーが発生します。

## Linker Errors

The linker is a program that combines the generated code into an executable. Since the executable format differs between Linux, Windows, and macOS, each system has its own linker that throws a different error. The fundamental cause of the errors is the same: the default configuration of the linker assumes that our program depends on the C runtime, which it does not.

To solve the errors, we need to tell the linker that it should not include the C runtime. We can do this either by passing a certain set of arguments to the linker or by building for a bare metal target.

### Building for a Bare Metal Target

By default Rust tries to build an executable that is able to run in your current system environment. For example, if you're using Windows on `x86_64`, Rust tries to build a `.exe` Windows executable that uses `x86_64` instructions. This environment is called your "host" system.

To describe different environments, Rust uses a string called [_target triple_]. You can see the target triple for your host system by running `rustc --version --verbose`:

[_target triple_]: https://clang.llvm.org/docs/CrossCompilation.html#target-triple

```
rustc 1.35.0-nightly (474e7a648 2019-04-07)
binary: rustc
commit-hash: 474e7a6486758ea6fc761893b1a49cd9076fb0ab
commit-date: 2019-04-07
host: x86_64-unknown-linux-gnu
release: 1.35.0-nightly
LLVM version: 8.0
```

The above output is from a `x86_64` Linux system. We see that the `host` triple is `x86_64-unknown-linux-gnu`, which includes the CPU architecture (`x86_64`), the vendor (`unknown`), the operating system (`linux`), and the [ABI] (`gnu`).

[ABI]: https://en.wikipedia.org/wiki/Application_binary_interface

By compiling for our host triple, the Rust compiler and the linker assume that there is an underlying operating system such as Linux or Windows that use the C runtime by default, which causes the linker errors. So to avoid the linker errors, we can compile for a different environment with no underlying operating system.

An example for such a bare metal environment is the `thumbv7em-none-eabihf` target triple, which describes an [embedded] [ARM] system. The details are not important, all that matters is that the target triple has no underlying operating system, which is indicated by the `none` in the target triple. To be able to compile for this target, we need to add it in rustup:

[embedded]: https://en.wikipedia.org/wiki/Embedded_system
[ARM]: https://en.wikipedia.org/wiki/ARM_architecture

```
rustup target add thumbv7em-none-eabihf
```

This downloads a copy of the standard (and core) library for the system. Now we can build our freestanding executable for this target:

```
cargo build --target thumbv7em-none-eabihf
```

By passing a `--target` argument we [cross compile] our executable for a bare metal target system. Since the target system has no operating system, the linker does not try to link the C runtime and our build succeeds without any linker errors.

[cross compile]: https://en.wikipedia.org/wiki/Cross_compiler

This is the approach that we will use for building our OS kernel. Instead of `thumbv7em-none-eabihf`, we will use a [custom target] that describes a `x86_64` bare metal environment. The details will be explained in the next post.

[custom target]: https://doc.rust-lang.org/rustc/targets/custom.html

### Linker Arguments

Instead of compiling for a bare metal system, it is also possible to resolve the linker errors by passing a certain set of arguments to the linker. This isn't the approach that we will use for our kernel, therefore this section is optional and only provided for completeness. Click on _"Linker Arguments"_ below to show the optional content.

<details>

<summary>Linker Arguments</summary>

In this section we discuss the linker errors that occur on Linux, Windows, and macOS, and explain how to solve them by passing additional arguments to the linker. Note that the executable format and the linker differ between operating systems, so that a different set of arguments is required for each system.

#### Linux

On Linux the following linker error occurs (shortened):

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x12): undefined reference to `__libc_csu_fini'
          /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x19): undefined reference to `__libc_csu_init'
          /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x25): undefined reference to `__libc_start_main'
          collect2: error: ld returned 1 exit status
```

The problem is that the linker includes the startup routine of the C runtime by default, which is also called `_start`. It requires some symbols of the C standard library `libc` that we don't include due to the `no_std` attribute, therefore the linker can't resolve these references. To solve this, we can tell the linker that it should not link the C startup routine by passing the `-nostartfiles` flag.

One way to pass linker attributes via cargo is the `cargo rustc` command. The command behaves exactly like `cargo build`, but allows to pass options to `rustc`, the underlying Rust compiler. `rustc` has the `-C link-arg` flag, which passes an argument to the linker. Combined, our new build command looks like this:

```
cargo rustc -- -C link-arg=-nostartfiles
```

Now our crate builds as a freestanding executable on Linux!

We didn't need to specify the name of our entry point function explicitly since the linker looks for a function with the name `_start` by default.

#### Windows

On Windows, a different linker error occurs (shortened):

```
error: linking with `link.exe` failed: exit code: 1561
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1561: entry point must be defined
```

The "entry point must be defined" error means that the linker can't find the entry point. On Windows, the default entry point name [depends on the used subsystem][windows-subsystems]. For the `CONSOLE` subsystem the linker looks for a function named `mainCRTStartup` and for the `WINDOWS` subsystem it looks for a function named `WinMainCRTStartup`. To override the default and tell the linker to look for our `_start` function instead, we can pass an `/ENTRY` argument to the linker:

[windows-subsystems]: https://docs.microsoft.com/en-us/cpp/build/reference/entry-entry-point-symbol

```
cargo rustc -- -C link-arg=/ENTRY:_start
```

From the different argument format we clearly see that the Windows linker is a completely different program than the Linux linker.

Now a different linker error occurs:

```
error: linking with `link.exe` failed: exit code: 1221
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1221: a subsystem can't be inferred and must be
          defined
```

This error occurs because Windows executables can use different [subsystems][windows-subsystems]. For normal programs they are inferred depending on the entry point name: If the entry point is named `main`, the `CONSOLE` subsystem is used, and if the entry point is named `WinMain`, the `WINDOWS` subsystem is used. Since our `_start` function has a different name, we need to specify the subsystem explicitly:

```
cargo rustc -- -C link-args="/ENTRY:_start /SUBSYSTEM:console"
```

We use the `CONSOLE` subsystem here, but the `WINDOWS` subsystem would work too. Instead of passing `-C link-arg` multiple times, we use `-C link-args` which takes a space separated list of arguments.

With this command, our executable should build successfully on Windows.

#### macOS

On macOS, the following linker error occurs (shortened):

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: entry point (_main) undefined. for architecture x86_64
          clang: error: linker command failed with exit code 1 […]
```

This error message tells us that the linker can't find an entry point function with the default name `main` (for some reason all functions are prefixed with a `_` on macOS). To set the entry point to our `_start` function, we pass the `-e` linker argument:

```
cargo rustc -- -C link-args="-e __start"
```

The `-e` flag specifies the name of the entry point function. Since all functions have an additional `_` prefix on macOS, we need to set the entry point to `__start` instead of `_start`.

Now the following linker error occurs:

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: dynamic main executables must link with libSystem.dylib
          for architecture x86_64
          clang: error: linker command failed with exit code 1 […]
```

macOS [does not officially support statically linked binaries] and requires programs to link the `libSystem` library by default. To override this and link a static binary, we pass the `-static` flag to the linker:

[does not officially support statically linked binaries]: https://developer.apple.com/library/content/qa/qa1118/_index.html

```
cargo rustc -- -C link-args="-e __start -static"
```

This still not suffices, as a third linker error occurs:

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: library not found for -lcrt0.o
          clang: error: linker command failed with exit code 1 […]
```

This error occurs because programs on macOS link to `crt0` (“C runtime zero”) by default. This is similar to the error we had on Linux and can be also solved by adding the `-nostartfiles` linker argument:

```
cargo rustc -- -C link-args="-e __start -static -nostartfiles"
```

Now our program should build successfully on macOS.

#### Unifying the Build Commands

Right now we have different build commands depending on the host platform, which is not ideal. To avoid this, we can create a file named `.cargo/config` that contains the platform specific arguments:

```toml
# in .cargo/config

[target.'cfg(target_os = "linux")']
rustflags = ["-C", "link-arg=-nostartfiles"]

[target.'cfg(target_os = "windows")']
rustflags = ["-C", "link-args=/ENTRY:_start /SUBSYSTEM:console"]

[target.'cfg(target_os = "macos")']
rustflags = ["-C", "link-args=-e __start -static -nostartfiles"]
```

The `rustflags` key contains arguments that are automatically added to every invocation of `rustc`. For more information on the `.cargo/config` file check out the [official documentation](https://doc.rust-lang.org/cargo/reference/config.html).

Now our program should be buildable on all three platforms with a simple `cargo build`.

#### Should You Do This?

While it's possible to build a freestanding executable for Linux, Windows, and macOS, it's probably not a good idea. The reason is that our executable still expects various things, for example that a stack is initialized when the `_start` function is called. Without the C runtime, some of these requirements might not be fulfilled, which might cause our program to fail, e.g. through a segmentation fault.

If you want to create a minimal binary that runs on top of an existing operating system, including `libc` and setting the `#[start]` attribute as described [here](https://doc.rust-lang.org/1.16.0/book/no-stdlib.html) is probably a better idea.

</details>

## 概要

A minimal freestanding Rust binary looks like this:

`src/main.rs`:

```rust
#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`Cargo.toml`:

```toml
[package]
name = "crate_name"
version = "0.1.0"
authors = ["Author Name <author@example.com>"]

# the profile used for `cargo build`
[profile.dev]
panic = "abort" # disable stack unwinding on panic

# the profile used for `cargo build --release`
[profile.release]
panic = "abort" # disable stack unwinding on panic
```

To build this binary, we need to compile for a bare metal target such as `thumbv7em-none-eabihf`:

```
cargo build --target thumbv7em-none-eabihf
```

Alternatively, we can compile it for the host system by passing additional linker arguments:

```bash
# Linux
cargo rustc -- -C link-arg=-nostartfiles
# Windows
cargo rustc -- -C link-args="/ENTRY:_start /SUBSYSTEM:console"
# macOS
cargo rustc -- -C link-args="-e __start -static -nostartfiles"
```

Note that this is just a minimal example of a freestanding Rust binary. This binary expects various things, for example that a stack is initialized when the `_start` function is called. **So it probably for any real use of such a binary, more steps are required**.

## What's next?

The [next post] explains the steps needed for turning our freestanding binary into a minimal operating system kernel. This includes creating a custom target, combining our executable with a bootloader, and learning how to print something to the screen.

[next post]: ./second-edition/posts/02-minimal-rust-kernel/index.md
