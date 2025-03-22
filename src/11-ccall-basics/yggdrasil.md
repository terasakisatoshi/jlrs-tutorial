# Yggdrasil

RustFFT.jlのようなRustコードに依存するパッケージを追加して使用しようとする前に、システムからRustを完全に削除した場合でも、すべてが正しく動作することがわかります。すべてが正常に動作するのは、ライブラリをビルドするためのレシピが[Yggdrasil]に用意されているからです。Yggdrasilにレシピを提供すると、そのライブラリは事前にビルドされ、JLLパッケージとして利用可能になります。

Rustで書かれたライブラリのレシピは通常次のようになり、`build_tarballs.jl`という名前にする必要があります。

```julia
# Note that this script can accept some limited command-line arguments, run
# `julia build_tarballs.jl --help` to see a usage message.
using BinaryBuilder, Pkg

name = "{{crate_name}}"
version = v"0.1.0"

# Collection of sources required to complete build
sources = [
    GitSource("https://github.com/{{user}}/{{crate_name}}.git",
              "{full commit hash, e.g.: 52ab80563a07d02e3d142f85101853bbf5c0a8a1}"),
]

# Bash recipe for building across all platforms
script = raw"""
cd $WORKSPACE/srcdir/{{crate_name}}

cargo build--release --verbose
install_license LICENSE
install -Dvm 0755 "target/${rust_target}/release/"*{{crate_name}}.${dlext} "${libdir}/lib{{crate_name}}.${dlext}"
"""

# These are the platforms we will build for by default, unless further
# platforms are passed in on the command line
platforms = supported_platforms(; experimental=true)
# Rust toolchain for i686 Windows is unusable
filter!(p -> !Sys.iswindows(p) || arch(p) != "i686", platforms)

# The products that we will ensure are always built
products = [
    LibraryProduct("lib{{crate_name}}", :lib{{crate_name}}),
]

# Dependencies that must be installed before this package can be built
dependencies = [
    Dependency("Libiconv_jll"; platforms=filter(Sys.isapple, platforms)),
]

# Build the tarballs.
build_tarballs(ARGS, name, version, sources, script, platforms, products, dependencies;
               julia_compat="1.6", compilers=[:c, :rust])
```

このレシピは、`{{user}}`と`{{crate_name}}`を置き換えた後、純粋にRustで書かれたクレートに対して機能するはずです。クレートが他の動的ライブラリに依存している場合、そのライブラリのレシピも用意されている必要があり、それらを依存関係のリストに追加できます。これはこのチュートリアルの範囲外ですので、詳細は[BinaryBuilder.jl documentation]を参照してください。

最後に注意すべき点は、BinaryBuilder.jlが独自のRustツールチェーンを使用していることです。Rustツールチェーンの[レシピ]で使用されるRustのバージョンを確認できます。

レシピをローカルでテストするには、`julia build_tarballs.jl`を実行します。実際には、`--verbose`および`--debug`フラグを設定することが有用です。`--debug`が設定されている場合、問題を診断して解決するために失敗時にデバッグプロンプトが開かれます。成功した場合、`products`ディレクトリにコンパイルされたライブラリが見つかり、それを手動でコンパイルしたかのように使用できます。

すべてが期待通りに動作することを確認したら、レシピをYggdrasilに提供できます。Yggdrasilリポジトリをフォークし、レシピ用の新しいディレクトリを作成してそのディレクトリに配置します。`"[{{crate_name}}] version 0.1.0"`のようなメッセージでこれらの変更をコミットし、PRを開きます。すべてが順調に進み、PRが受け入れられた場合、`{{crate_name}}_jll`という名前の新しいパッケージが公開されます。

このJLLパッケージを使用するには、Juliaに追加するだけです。

```julia
(@v1.10) pkg> add {{crate_name}}_jll
   Resolving package versions...
  Downloaded artifact: {{crate_name}}
    Updating `~/.julia/environments/v1.10/Project.toml`
  [54eccfce] + {{crate_name}}_jll v0.1.0+0
    Updating `~/.julia/environments/v1.10/Manifest.toml`
  [54eccfce] + {{crate_name}}_jll v0.1.0+0
Precompiling project...
  1 dependency successfully precompiled in 2 seconds. 86 already precompiled. 1 skipped during auto due to previous errors.

julia> using {{crate_name}}_jll

julia> {{crate_name}}_jll.lib{{crate_name}}_path
"/path/to/lib{{crate_name}}.so"

julia> ccall((:some_exported_function, {{crate_name}}_jll.lib{{crate_name}}_path), Cvoid, ())

```

[Yggdrasil]: https://github.com/JuliaPackaging/Yggdrasil
[BinaryBuilder.jl documentation]: https://docs.binarybuilder.org/stable/
[recipe for the Rust toolchain]: https://github.com/JuliaPackaging/Yggdrasil/blob/master/0_RootFS/Rust/build_tarballs.jl

It seems like you haven't pasted any content yet. Please provide the Markdown content you would like translated, and I'll assist you with the translation.
