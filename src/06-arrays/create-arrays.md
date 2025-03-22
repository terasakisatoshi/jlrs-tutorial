# 配列の作成

新しい配列を作成する関数は主に2つのクラスに分けられます。`Typed(Ranked)Array`は、与えられた`T`を使用して要素の型を構築する`new`のような関数を提供し、`(Ranked)Array`は、要素の型を引数として取る`new_for`のような後置された関数を提供します。

これらの関数は、要素の型に加えて、配列の希望する次元を引数として取ります。ランク4まで、`usize`のタプルを使用してこれらの次元を表現できます。また、`[usize; N]`、`&[usize; N]`、`&[usize]`を使用することも可能です。配列のランクと次元がコンパイル時に既知であり、それらが一致しない場合、コードはコンパイルに失敗します。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 2>(|mut frame| {
        let arr1 = TypedArray::<f32>::new(&mut frame, (2, 2))
            .expect("invalid size");
        assert_eq!(arr1.rank(), 2);

        let f32_ty = DataType::float32_type(&frame).as_value();
        let arr2 = RankedArray::<2>::new_for(&mut frame, f32_ty, [2, 2])
            .expect("invalid size");

        assert_eq!(arr1.element_type(), arr2.element_type());
    })
}
```

`new(_for)`関数は、要素が初期化されていない配列を返します[^1]。既存の`Vec`やスライスを`from_vec(_for)`や`from_slice(_for)`でラップすることも可能です。これらの関数は、要素が`T`型の配列に対して正しく配置されていることを要求します。要素のレイアウトが`U`の場合、このレイアウトは`T`に対して正しいものでなければなりません。この接続は、型コンストラクタとそのレイアウト型を接続する`HasLayout`トレイトで表現されます。型パラメータが省略されていない限り、これらは同じ型です。

`from_vec(_for)`関数は`Vec`の所有権を取得し、配列がGCによって解放されるときにドロップされます。`from_slice(_for)`関数は、代わりにRustからデータを借用します。`Value`と`Array`は、`'data`と呼ばれる第2のライフタイムを持っています。このライフタイムは、借用が終了した後にこの配列にアクセスされないように、借用のライフタイムに設定されます。Juliaはこのライフタイムを認識していないため、配列をグローバル変数に割り当てたり、バックグラウンドスレッドに送信したりして配列を生かし続けることを防ぐものはありません。これが起こらないことを保証するのはあなたの責任であり、これがJulia関数を呼び出すメソッドがunsafeである理由の一つです。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 2>(|mut frame| {
        let data = vec![1.0f64, 2., 3., 4.];
        let arr = TypedArray::<f64>::from_vec(&mut frame, data, (2, 2))
            .expect("incompatible type and layout")
            .expect("invalid size");
        assert_eq!(arr.rank(), 2);

        let data = vec![1.0f64, 2., 3., 4.];
        let f64_ty = DataType::float64_type(&frame).as_value();
        let arr2 = RankedArray::<2>::from_vec_for(&mut frame, f64_ty, data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        assert_eq!(arr.element_type(), arr2.element_type());
    });

    handle.local_scope::<_, 2>(|mut frame| {
        let mut data = vec![1.0f64, 2., 3., 4.];
        let arr = TypedArray::<f64>::from_slice(&mut frame, &mut data, (2, 2))
            .expect("incompatible type and layout")
            .expect("invalid size");
        assert_eq!(arr.rank(), 2);

        let mut data = vec![1.0f64, 2., 3., 4.];
        let f64_ty = DataType::float64_type(&frame).as_value();
        let arr2 = RankedArray::<2>::from_slice_for(&mut frame, f64_ty, &mut data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        assert_eq!(arr.element_type(), arr2.element_type());
    })
}
```

`from_slice_cloned(_for)`と`from_slice_copied(_for)`関数は、`new(_for)`を使用して配列を割り当て、指定されたスライスからこの配列に要素をクローンまたはコピーします。これらの関数は、`from_vec(_for)`のファイナライザと`from_slice(_for)`のライフタイム制限を回避しますが、要素をクローンまたはコピーするコストがかかります。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 2>(|mut frame| {
        let data = [1.0f64, 2., 3., 4.];
        let arr = TypedArray::<f64>::from_slice_cloned(&mut frame, &data, (2, 2))
            .expect("incompatible type and layout")
            .expect("invalid size");
        assert_eq!(arr.rank(), 2);

        let f64_ty = DataType::float64_type(&frame).as_value();
        let arr2 = RankedArray::<2>::from_slice_cloned_for(&mut frame, f64_ty, &data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        assert_eq!(arr.element_type(), arr2.element_type());
    });

    handle.local_scope::<_, 2>(|mut frame| {
        let data = [1.0f64, 2., 3., 4.];
        let arr = TypedArray::<f64>::from_slice_copied(&mut frame, &data, (2, 2))
            .expect("incompatible type and layout")
            .expect("invalid size");
        assert_eq!(arr.rank(), 2);

        let f64_ty = DataType::float64_type(&frame).as_value();
        let arr2 = RankedArray::<2>::from_slice_copied_for(&mut frame, f64_ty, &data, [2, 2])
            .expect("incompatible type and layout")
            .expect("invalid size");

        assert_eq!(arr.element_type(), arr2.element_type());
    });
}
```

最後に、2つの特殊な関数があります。`TypedVector::<Any>::new_any` は任意の型の要素を保持できるベクターを割り当てます。`TypedVector::<u8>::from_bytes` は、バイトのスライスとして参照できるものを `TypedVector<u8>` に変換できます。これは `TypedVector::<u8>::from_slice_copied` に似ています。

```rust,ignore
use jlrs::prelude::*;

fn main() {
    let handle = Builder::new().start_local().expect("cannot init Julia");

    handle.local_scope::<_, 1>(|mut frame| {
        let arr = VectorAny::new_any(&mut frame, 3).expect("invalid size");
        assert_eq!(arr.rank(), 1);
    });

    handle.local_scope::<_, 2>(|mut frame| {
        let data = [1u8, 2, 3, 4];
        let arr = TypedVector::<u8>::from_bytes(&mut frame, &data).expect("invalid size");
        assert_eq!(arr.rank(), 1);

        let data = "also bytes";
        let arr = TypedVector::<u8>::from_bytes(&mut frame, &data).expect("invalid size");
        assert_eq!(arr.rank(), 1);
    });
}
```

[^1]: 要素が他の管理されたデータを参照している場合、配列ストレージは0に初期化されます。
