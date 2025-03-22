# `julia_module!`

前の章では、jlrsを使用せずにRustコードをJuliaに公開する動的ライブラリを作成しました。jlrsを使用することで、カスタムタイプのサポートの向上、コード生成、Juliaコードとの統合など、多くの追加機能が提供されます。主な欠点は、ライブラリが異なるバージョンのJuliaと互換性がなく、バージョン機能で選択された特定のバージョン用にコンパイルされることです。

この章では、`julia_module!`マクロを使用して、定数、型、関数をJuliaにエクスポートします。これを使用するには、`jlrs-derive`と`ccall`機能を有効にする必要があります。

```toml
[package]
name = "julia_module_tutorial"
version = "0.1.0"
edition = "2021"

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"

[features]
julia-1-6 = ["jlrs/julia-1-6"]
julia-1-7 = ["jlrs/julia-1-7"]
julia-1-8 = ["jlrs/julia-1-8"]
julia-1-9 = ["jlrs/julia-1-9"]
julia-1-10 = ["jlrs/julia-1-10"]
julia-1-11 = ["jlrs/julia-1-11"]

[lib]
crate-type = ["cdylib"]

[dependencies]
jlrs = { version = "0.21", features = ["jlrs-derive", "ccall"] }
```

動的ライブラリをビルドする際には、`local-rt`のようなランタイム機能を有効にしないことが重要です。

以下は`julia_module!`の最小限の例です：

```rust,ignore
use jlrs::prelude::*;

julia_module! {
    become julia_module_tutorial_init_fn;

    // module content...
}
```

このマクロは単一の関数`julia_module_tutorial_init_fn`に変換され、JlrsCore.jlの`@wrapmodule`マクロと共に使用できます：

```julia
module JuliaModuleTutorial
using JlrsCore.Wrap

@wrapmodule("/path/to/libjulia_module_tutorial", :julia_module_tutorial_init_fn)

function __init__()
    @initjlrs
end
end
```

これが私たちが書く必要のあるすべてのJuliaコードであり、`@wrapmodule`マクロがモジュールの内容を生成します。簡潔さのために、以下のセクションのコードサンプルでは、このモジュール定義の省略形として`module JuliaModuleTutorial ... end`と書きます。
