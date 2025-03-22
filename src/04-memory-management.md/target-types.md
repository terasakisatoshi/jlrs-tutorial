# ターゲットタイプ

ターゲットタイプはまだ指定されておらず、フレームでさえもその正確なタイプが詳述されていません。以下の表は、すべてのターゲットタイプ、`'target` に使用されるライフタイム、およびそれがルート化ターゲットかどうかを示しています。

| タイプ                                | ルート化 |
| ----------------------------------- | ------- |
| `LocalGcFrame<'target>`             | はい     |
| `&mut LocalGcFrame<'target>`        | はい     |
| `UnsizedLocalGcFrame<'target>`      | はい     |
| `&mut UnsizedLocalGcFrame<'target>` | はい     |
| `LocalOutput<'target>`              | はい     |
| `&'target mut LocalOutput`          | はい     |
| `LocalReusableSlot<'target>`        | はい     |
| `&mut LocalReusableSlot<'target>`   | はい[^1] |
| `GcFrame<'target>`                  | はい     |
| `&mut GcFrame<'target>`             | はい     |
| `Output<'target>`                   | はい     |
| `&'target mut Output`               | はい     |
| `ReusableSlot<'target>`             | はい     |
| `&mut ReusableSlot<'target>`        | はい[^1] |
| `AsyncGcFrame<'target>`             | はい     |
| `&mut AsyncGcFrame<'target>`        | はい     |
| `Unrooted`                          | いいえ   |
| `StackHandle<'target>`              | いいえ   |
| `Pin<&'target mut WeakHandle>`      | いいえ   |
| `ActiveHandle<'target>`             | いいえ   |
| `&Tgt where Tgt: Target<'target>`   | いいえ   |

これらのターゲットは、ローカルターゲット、動的ターゲット、および非ルート化ターゲットの3つの異なるグループに属します。

[^1]: `(Local)ReusableSlot` への可変参照はデータをルート化しますが、結果にスコープのライフタイムを割り当てるため、スコープを離れるまで結果が生存することができます。このスロットは再利用可能ですが、データが `'target` ライフタイム全体にわたってルート化され続けることは保証されません。このため、非ルート化参照が返されます。
