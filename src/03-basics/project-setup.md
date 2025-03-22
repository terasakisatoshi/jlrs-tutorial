# プロジェクトのセットアップ

まず、`cargo`を使って新しいバイナリパッケージを作成します。

```bash
cargo new julia_app --bin
```

`Cargo.toml`を開き、`jlrs`を依存関係として追加し、`local_rt`機能を有効にします。パニック時には中断し[^1]、バージョン機能を再エクスポートします。

```toml
[package]
name = "julia_app"
version = "0.1.0"
edition = "2021"

[features]
julia-1-6 = ["jlrs/julia-1-6"]
julia-1-7 = ["jlrs/julia-1-7"]
julia-1-8 = ["jlrs/julia-1-8"]
julia-1-9 = ["jlrs/julia-1-9"]
julia-1-10 = ["jlrs/julia-1-10"]
julia-1-11 = ["jlrs/julia-1-11"]

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"

[dependencies]
jlrs = {version = "0.21", features = ["local-rt"]}
```

バージョン機能を有効にせずにアプリケーションをビルドしようとすると、次のエラーが表示されます。

```text
error: A Julia version must be selected by enabling exactly one of the following version features:
           julia-1-6
           julia-1-7
           julia-1-8
           julia-1-9
           julia-1-10
           julia-1-11
```

Julia 1.10がインストールされ、[依存関係の章]に従って環境が設定されている場合、`julia-1-10`機能を有効にした後、ビルドと実行が成功するはずです。

```bash
cargo build --features julia-1-10
```

LinuxでJuliaを埋め込む際には、`-rdynamic`リンカーフラグを設定することが重要です。そうしないと、Juliaのパフォーマンスが低下します。[^2] このフラグは、`RUSTFLAGS`環境変数を使用してコマンドラインで設定できます。

`RUSTFLAGS="-Clink-args=-rdynamic" cargo build --features julia-1-10`

このフラグは、プロジェクトのルートディレクトリにある`config.toml`ファイルで設定することも可能です。

```toml
[target.linux]
rustflags = [ "-C", "link-args=-rdynamic" ]
```

[依存関係の章]: ../01-dependencies/julia.md

[^1]: 特定の状況では、パニックが健全性の問題を引き起こす可能性があるため、中断する方が良いです。

[^2]: 詳細な理由としては、Juliaが常に使用するスレッドローカルデータがあるためです。このデータに効果的にアクセスするためには、アプリケーション内で定義されている必要があり、最もパフォーマンスの高いTLSモデルを使用できます。`-rdynamic`リンカーフラグを設定することで、`libjulia`はアプリケーション内の定義を見つけて利用できます。このフラグが設定されていない場合、Juliaはより遅いTLSモデルにフォールバックし、パフォーマンスに大きな悪影響を及ぼします。これはLinuxでのみ重要であり、macOSやWindowsのユーザーはこの点を完全に無視できます。これらのプラットフォームではTLSモデルの概念が存在しないためです。
