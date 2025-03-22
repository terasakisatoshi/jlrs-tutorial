# パッケージとその他のカスタムコードの読み込み

これまでに行ったことはすべて、jlrsとJuliaで直接利用可能な標準機能に関するものでした。最悪の場合、カスタムタイプを定義するためにいくつかのコードを評価する必要がありました。この基本的な機能を利用できるのは良いことですが、パッケージも利用したいと考えるのは当然です。

ターゲットとするJuliaのバージョンにインストールされているパッケージは、`LocalHandle::using`を使って読み込むことができます。[^1]

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 1>(|mut frame| {
        let dot = Module::main(&frame).global(&mut frame, "dot");
        assert!(dot.is_err());
    });

    // Safety: LinearAlgebra is a valid package name
    unsafe {
        handle.using("LinearAlgebra")
    }.expect("Package does not exist");

    handle.local_scope::<_, 1>(|mut frame| {
        let dot = Module::main(&frame).global(&mut frame, "dot");
        assert!(dot.is_ok());
    });
}
```

`dot`関数は、`handle.using("LinearAlgebra")`を呼び出すまで`Main`モジュールに定義されません。これは内部的には単に`using LinearAlgebra`を評価するだけです。インポートを制限するには、`using`または`import`文を手動で構築し、`Value::eval_string`で評価する必要があります。

読み込むパッケージはすべて事前にインストールされている必要があります。REPLとは異なり、インストールされていないパッケージを使用しようとすると、インストールを促すプロンプトが表示されるのではなく、単に失敗します。パッケージが読み込まれた後、そのルートモジュールには`Module::package_root_module`でアクセスできます。

カスタムJuliaコードを含むファイルを読み込むのも同様です。任意のファイルを`LocalHandle::include`で読み込み、評価することができます。これは指定されたパスで`Main.include`を呼び出します。ローカル開発には適していますが、コードを配布する際にファイルへの正しいパスを見つけるのは問題になることがあります。この場合、`include_str!`マクロを使ってファイルの内容を含め、`Value::eval_string`で評価する方が良いでしょう。

[^1]: [`JULIA_DEPOT_PATH`環境変数]を変更しない限り

[`JULIA_DEPOT_PATH`環境変数]: https://docs.julialang.org/en/v1/manual/environment-variables/#JULIA_DEPOT_PATH
