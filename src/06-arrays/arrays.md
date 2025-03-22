# 配列

これまで、比較的単純な型を扱ってきましたが、ここで `Array` 型を見てみましょう。この型をより複雑にしている最初の要素は、その型パラメータである要素型 `T` とランク `N` です。これらは、jlrs の他の型パラメータとは異なる特別な処理が施されています。

この特別な処理には、単一の基本型 `ArrayBase<T, const N: isize>` と、4つの可能なケースのエイリアスが含まれます：

 - `Array == ArrayBase<Unknown, -1>`
 - `TypedArray<T> == ArrayBase<T, -1>`
 - `RankedArray<const N: isize> == ArrayBase<Unknown, N>`
 - `TypedRankedArray<T, const N: isize> == ArrayBase<T, N>`

このリストからわかるように、配列の要素型とランクを無視することが可能です。既知の要素型は `ConstructType` を実装する必要があり、既知のランクは0以上です。

いくつかの追加の特殊化された型エイリアスがあります：

 - `Vector == RankedArray<1>`
 - `TypedVector<T> == TypedRankedArray<T, 1>`
 - `VectorAny == TypedVector<Any>`
 - `Matrix == RankedArray<2>`
 - `TypedMatrix<T> == TypedRankedArray<T, 2>`

Julia の配列の要素は列優先順序、つまり「F」または「Fortran」順序で格納されます。シーケンス `1, 2, 3, 4, 5, 6` は次のような2 x 3の行列にマッピングされます：

```text
1  3  5
2  4  6
```

つまり、

```julia
[1 3 5; 2 4 6]
```
