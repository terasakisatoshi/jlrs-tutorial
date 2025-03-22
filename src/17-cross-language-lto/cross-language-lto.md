# クロス言語LTO

jlrsの中心には、Cで書かれた小さな静的ライブラリがあります。このライブラリは以下の目的を果たします：

 - JuliaのC APIの実装詳細を隠します。
 - マクロや静的インライン関数で実装された機能を公開します。
 - 後方互換性のない変更に対する回避策を提供します。

多くの操作はこのライブラリに委任されており、関数を呼び出すオーバーヘッドと比べると非常に安価です。このライブラリはCで書かれているため、これらの関数はインライン化されることはありません。

このライブラリを`clang`でビルドする場合、`clang`と`rustc`が同じメジャーLLVMバージョンを使用している場合に、`lto`機能を使ってクロス言語LTOを有効にできます。`rustc -vV`を使って、どのバージョンのclangを使用する必要があるかを確認できます。

```bash
> rustc -vV
rustc 1.80.1 (3f5fd8dd4 2024-08-06)
binary: rustc
commit-hash: 3f5fd8dd41153bc5fdca9427e9e05be2c767ba23
commit-date: 2024-08-06
host: x86_64-unknown-linux-gnu
release: 1.80.1
LLVM version: 18.1.7
```

関連情報は最終行にあります：LLVM 18が使用されているので、clang-18を使用する必要があります。

```bash
RUSTFLAGS="-Clinker-plugin-lto -Clinker=clang-18 -Clink-arg=-fuse-ld=lld -Clink-args=-rdynamic" \
CC=clang-18 \
cargo build --release --features {{julia_version}}
```

クロス言語LTOはLinuxでのみテストされており、アプリケーションや動的ライブラリに対して有効にできます。これはJuliaコードのパフォーマンスには影響を与えず、中間ライブラリを呼び出すRustコードにのみ影響します。
