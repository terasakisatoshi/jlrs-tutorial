# アプリケーションのテスト

Juliaを組み込んだバイナリクレートをテストする際には、Juliaは一度しか初期化できないことを念頭に置く必要があります。これを確実にする最も簡単な方法は、別の`tests`ディレクトリに統合テストを限定し、ファイルごとに1つのテストを定義することです。`tests`ディレクトリ内の各ファイルは別々のプロセスで実行され、単一のテストにより、単一のスレッドからのみJuliaを使用することが保証されます。このアプローチはすべてのランタイムに適用されます。

```rust,ignore
use jlrs::prelude::*;

fn test_case_1<'target, Tgt: Target<'target>>(_target: &Tgt) {}

fn test_case_2<'target, Tgt: Target<'target>>(_target: &Tgt) {}

#[test]
fn test_fn() {
    let handle = Builder::new().start_local().expect("cannot init Julia");
    handle.local_scope::<_, 0>(|frame| {
        test_case_1(&frame);
        test_case_2(&frame);
    });
}
```

この制約はドクトテストには適用されません。各ドクトテストは別々のプロセスで実行されます。

[^1]: `--test-threads=1`を設定しても、ファイルごとに複数のテストを許可するわけではありません。異なるテストは順次実行されますが、異なるスレッドから実行されます。
