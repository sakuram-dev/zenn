---
title: "Androidアプリ開発者もRustを使いたい！"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [rust, android, mobile, kotlin, jni]
published: false
---


## AndroidでRustが使われているみたいだけど...？
近年、RustがAndroid界隈でも活発です。  

Rustは、メモリ安全性、並行性、パフォーマンスの向上など、多くの利点を持つプログラミング言語です。Googleも既存の機能をRustで書き直すことで、メモリリークによるエラーを劇的に減らすことができたと発表しています。

https://security.googleblog.com/2024/09/eliminating-memory-safety-vulnerabilities-Android.html

しかしそれは同じAndroidと言ってもアプリよりさらに下の深い部分...。アプリ開発者にとっては対岸の火事のような話かもしれません。しかしアプリ開発者もRustを使いたい！このビッグウェーブに乗りたい！という気持ちがあるのも事実（主に私が）。

そこで今回は、AndroidアプリケーションでRustを使用する方法を紹介します。

## AndroidアプリでRustを使用する方法

この記事では、AndroidアプリケーションでRustを使用する方法を紹介します。具体的には、以下の手順で進めます。
1. Rustのライブラリをクロスコンパイル
2. JNI(Java Native Interface)を使用してKotlinからRustライブラリを呼び出す

本稿では、例としてユーザー受け取った数値の素数判定をRustで行い、その結果をAndroidアプリケーションに表示するアプリケーションを作成してみましょう。

### 前提条件

- Android Studioがインストールされている
- Rustがインストールされている

### プロジェクト構成

今回は同一フォルダ内にAndroidプロジェクトとRustプロジェクトを作成します。  


```sh
~/AndroidAppWithRust
├── android/　#　Androidプロジェクト
└── rust/　# Rustプロジェクト
```

必ずしも同一フォルダ内である必要はありません。各々管理のしやすい構成を選択してください。

### Rustライブラリの作成

Rustプロジェクトを初期化し、必要なディレクトリとファイルを作成します。今回はライブラリを作成するため、`--lib`オプションを使用します。
```sh
cd ~/AndroidAppWithRust
cargo init --lib rust
```

これにより、`rust/`ディレクトリ内に基本的なRustプロジェクトが作成されます。

#### Cargo.tomlの設定
まず、`Cargo.toml`には以下のように設定します。

```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
jni = "0.19.0"
```

`cdylib`は、動的ライブラリを生成するためのcrate-typeです。指定することで、他の言語から、今回でいうとKotlinからライブラリを呼び出すことができます。  
https://doc.rust-lang.org/reference/linkage.html

`jni`は、JNI(Java Native Interface)を使用するためのライブラリです。JNIを使用することで、JavaコードからRustのメソッドを呼び出すことができます。
https://docs.rs/jni/latest/jni/


#### 素数判定の実装
Rustによる素数判定の実装は以下になります。

```Rust
// src/lib.rs
#[no_mangle]
pub extern "C" fn Java_com_example_androidappwithrust_RustLib_isPrime(env: *mut jni::JNIEnv, class: jni::objects::JClass, n: i32) -> bool {
    if n <= 1 {
        return false;
    }
    for i in 2..=((n as f64).sqrt() as i32) {
        if n % i == 0 {
            return false;
        }
    }
    true
}
```

関数名がなんだか煩雑ですが、これはJNIの仕様に従っています。
以下のような命名規則に従っています。
```
Java_パッケージ名_クラス名_メソッド名
```

これでRust側の実装は完了です。お疲れ様でした。

### Rustのクロスコンパイル
RustコードをAndroid向けにクロスコンパイルするには、適切なターゲットを追加してビルドします。  

#### Rustのターゲット追加

```sh
rustup target add aarch64-linux-android
```

#### .cargo/config.tomlの設定
Rustのターゲットを追加したら、`.cargo/config.toml`に以下の設定を追加します。

```toml
[target.aarch64-linux-android]
ar = "<path-to-ndk>/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android-ar"
linker = "<path-to-ndk>/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android21-clang"
```
`<path-to-ndk>`は、Android NDKのパスを指定してください。  
まだインストールしていない場合は、Android StudioのSDK Managerからインストールしてください。

#### クロスコンパイルの実行
次に、以下のコマンドでRustライブラリをクロスコンパイルします。

```sh
cargo build --release --target aarch64-linux-android
```
これにより、`librust_lib.so`が`release`ディレクトリに生成されます。

### Androidプロジェクトの作成
`android/`にAndroidプロジェクトを作成します。私はAndroid Studioから新規プロジェクトを作成しました。

なお、今回はビルド構成にKotlin DSLを使用しています。Groovy DSLを使用している場合は、それに合わせて読み直していただけると幸いです。

### AndroidプロジェクトへのRustライブラリの組み込み
RustライブラリをAndroidプロジェクトに組み込みます。  

今回はAndroidプロジェクト側に`jniLibs`ディレクトリを作成し、`librust_lib.so`をコピーします。

```sh
mkdir -p android/app/src/main/jniLibs/arm64-v8a
cp rust/target/aarch64-linux-android/release/librust_lib.so android/app/src/main/jniLibs/arm64-v8a/
```

#### Gradleビルドスクリプトの設定
作成した`jniLibs`ディレクトリをAndroidプロジェクト側のビルドスクリプトに追加します。`android/app/build.gradle.kts`に以下の設定を追加します。

```kotlin
android {
    ...
    sourceSets {
        main {
            jniLibs.srcDirs("src/main/jniLibs")
        }
    }
}
```
これで、AndroidプロジェクトにRustライブラリが組み込まれました。

#### メソッドの宣言
Rustライブラリの関数をKotlinから呼び出すためのJNIメソッドを宣言します。

```Kotlin
class RustLib {
    companion object {
        init {
            System.loadLibrary("rust_lib")
        }

        external fun isPrime(n: Int): Boolean
    }
}
```

これでKotlin側でRustライブラリの関数を呼び出す準備が整いました。  
`Rustlib.isPrime()`を呼び出すことで、Rustライブラリの`Java_com_example_androidappwithrust_RustLib_isPrime`関数を呼び出すことができます。

### おまけ: Kotlin実装と比較してみる

最後に、Kotlinでの素数判定とRustでの素数判定を比較してみました。

実装は以下を参照ください。
https://github.com/sakuram-dev/AndroidAppWithRust


結果はこちら。オーダーが1つ違う！！！
![](/images/AndroidAppWithRust.jpg)

この結果だけを持ってRustのほうが良いとは言い切れませんが、面白い結果にはなりましたね。

## 所感
- RustでAndroidアプリの機能を実装すること自体は可能、しかし面倒。
- パフォーマンスや並列性が重要な機能であれば使う価値があるかもしれない。
- しかしJavaやKotlinと比べてどこまで優位性があるのかはやってみないとわからない。
- Androidアプリ開発者が移行するというよりかは、RustエンジニアがAndroidアプリ開発に参入するという方が正しいかもしれない。

AndroidアプリでRustを使うことに興味がある方は、ぜひ試してみてください。

既にAndroidアプリにRustの実装を組み込んでいる方は、ぜひメリット等教えてください！