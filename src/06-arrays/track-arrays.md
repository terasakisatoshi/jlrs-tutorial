# 配列の追跡

同じ配列に対して複数の可変アクセサを作成するのは簡単です。応急処置として、jlrs は Rust コード内での可変エイリアスをある程度防ぐために管理されたデータを追跡することができます。配列は、すべての配列型で利用可能な `track_exclusive` と `track_shared` メソッドを使用して、排他的または共有で追跡することができます。`track_shared` は配列がすでに排他的に追跡されていない限り成功し、`track_exclusive` は排他的アクセスを強制します。

全体として、追跡を一貫して使用する限り、配列へのアクセスをより安全にすることができますが、Julia コード内でのアクセスについては認識していません。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    // Shared tracking
    handle.local_scope::<_, 1>(|mut frame| {
        let data = [1.0f64, 2., 3., 4.];
        let arr = TypedArray::<f64>::from_slice_copied(&mut frame, &data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        let tracked = arr.track_shared().expect("already tracked exclusively");
        assert!(arr.track_exclusive().is_err());
        assert!(arr.track_shared().is_ok());

        let accessor = tracked.bits_data();

        let elem = accessor[[1, 0]];
        assert_eq!(elem, 2.);
    });

    // Exclusive tracking
    handle.local_scope::<_, 1>(|mut frame| {
        let data = [1.0f64, 2., 3., 4.];
        let arr = TypedArray::<f64>::from_slice_copied(&mut frame, &data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        let tracked = arr.track_exclusive().expect("already tracked exclusively");
        assert!(arr.track_exclusive().is_err());
        assert!(arr.track_shared().is_err());

        let accessor = tracked.bits_data();

        let elem = accessor[[1, 0]];
        assert_eq!(elem, 2.);
    });
}
```
