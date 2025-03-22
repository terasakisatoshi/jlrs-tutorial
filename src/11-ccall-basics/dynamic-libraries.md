# 動的ライブラリ

埋め込みの例はかなり作為的ですが、Juliaを埋め込むアプリケーションから関数を公開する必要があることはほとんどありません。むしろ、Rustのクレートが有用な機能を実装しており、それをJuliaに公開したいというケースの方が多いでしょう。

Juliaは実行時に動的ライブラリをロードできます。[^1] ここでは、先ほど定義した`add`関数を公開する新しい動的ライブラリを作成します。まず、新しいクレートを作成します。[^2]

```bash
cargo new julia_lib --lib
```

クレートタイプを`cdylib`に変更する必要があります。これは`Cargo.toml`で設定できます。

```toml
[package]
name = "julia_lib"
version = "0.1.0"
edition = "2021"

[profile.release]
panic = "abort"

[profile.dev]
panic = "abort"

[lib]
crate-type = ["cdylib"]

[dependencies]
```

jlrsを依存関係として追加する必要はありません。動的ライブラリでjlrsを使用する利点と欠点については次の章で説明します。

`lib.rs`の内容を次のコードに置き換えます。

```rust,ignore
#[no_mangle]
pub unsafe extern "C" fn add(a: f64, b: f64) -> f64 {
    a + b
}
```

関数は`#[no_mangle]`で注釈されており、名前がマングルされないようにしています。`cargo build`でビルドした後、ライブラリは`target/debug`にあります。Linuxでは`libjulia_lib.so`、macOSでは`libjulia_lib.dylib`、Windowsでは`libjulia_lib.dll`という名前になります。さあ、使ってみましょう！

`julia_lib`のルートディレクトリでJulia REPLを開き、次のコードを評価します。

```julia
julia> using Libdl

julia> handle = dlopen("./target/debug/libjulia_lib")
Ptr{Nothing} @0x0000000000e2a620

julia> func = dlsym(handle, "add")
Ptr{Nothing} @0x000073b0dec02100

julia> ccall(func, Float64, (Float64, Float64), 1.0, 2.0)
3.0
```

ライブラリを開く際に拡張子を指定する必要がないことに注意してください。

ライブラリがライブラリ検索パスにある場合、開いたり関数ポインタを取得したりする必要すらなく、直接参照できます。

```julia
julia> ccall((:add, "libjulia_lib"), Float64, (Float64, Float64), 1.0, 2.0)
3.0
```

[^1]: WindowsではGNUツールチェーンの使用が推奨されます。MSVCツールチェーンを使用することも可能かもしれませんが、これはテストされていません。

[^2]: 公開したいクレートがすでにC APIを提供している場合、中間クレートは必要ありません。ライブラリを直接ビルドし、既存のAPIに適応させることができます。
