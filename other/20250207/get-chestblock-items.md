# 【統合版】チェストに入っているアイテムを取得する
統合版のマインクラフトでチェストに入っているアイテムを取得できたらな～と思うことはありませんか？</br>
Java版であれば `/data get` コマンドを使うことで NBT としてアイテムスタックを取得することができます。</br>
統合版ではそもそも NBT や `data` コマンドが存在しませんので、コマンドを使ってアイテムを取得することはできません。が、Script API を使うことで取得できることができたので、備忘録として残しておきます。

# チェストの中身を調べる
例えば、座標 `-47 64 392` に設置したチェストの中身を調べたいと思います。

```ts
import { BlockInventoryComponent, ScriptEventCommandMessageAfterEvent, system, Vector3, world } from "@minecraft/server";

system.afterEvents.scriptEventReceive.subscribe((event: ScriptEventCommandMessageAfterEvent) => {
    if(event.id === "mcwithcode:getItem") {
        getChestItems();
    }
})

function getChestItems() {
    const locate:Vector3 = {x: -47, y: 64, z: 392};
    const inventory = world.getDimension("overworld").getBlock(locate).getComponent("inventory") as BlockInventoryComponent;
    
    for(let i: number = 0; i < 27; i++) {
        if (inventory.container.getItem(i) === undefined) {
            world.sendMessage(`スロット${i}番目：空です`);
            continue;
        } else {
            let items = inventory.container.getItem(i);
            world.sendMessage(`スロット${i}番目：${items.typeId}は${items.amount}個あります`);
        }
    }
}
```

## 実行結果
例えば、チェストの中身をこんな感じにしておきます。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/other/20250207/media/01.webp" vspace="10">

この状態で、下記のコマンドを実行してスクリプトを発火させます。

```txt
/scriptevent mcwithcode:getItem
```

アイテムが存在する場合はその詳細、存在しない場合は「空です」と表示されます。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/other/20250207/media/result.gif" vspace="10">

## 解説
まずはチェストの中身を調べるためのトリガー（発火条件）ですが、`scriptEventReceive` を使用することで、定義したイベントIDをコマンドから実行することで動かすことができます。</br>
`event` 引数のIDプロパティに代入され、文字列比較が行われて一致した時に `getChestItems()` メソッドが動きます。</br>

チェストを構成しているJSONファイルには `inventory` コンポーネントを含むため、`getComponent()` メソッドにてインベントリ情報を参照します。このメソッドはチェストが存在する場合に `BlockComponent` クラスとして値を返却します。</br>

しかし、このクラスにはインベントリ情報をもつプロパティやメソッドが存在しないため、代わりに `BlockInventoryComponent` クラスで型アサーションします。（ここが引っ掛けで、難しいポイントです。）</br>

```ts
const inventory = world.getDimension("overworld").getBlock(locate).getComponent("inventory") as BlockInventoryComponent;
```

インベントリ情報はこのクラスの `container` プロパティにて保存されており、`getItem()` メソッドでアイテム情報を取得できます。引数には調べたいスロット番号を入れます。返ってくる値は `ItemStack` 型となりますので、もし複数のアイテムを取得して管理したい場合は配列で定義しておくと良さそうですね。また、スロット番号の情報は持っていませんので、もし使う場合はこちらでスロット番号を管理できる型を定義してやりましょう。

## おまけ - アイテムの置き換え
例えば、特定の防具アイテムをチェストに入れたときに、エンチャントをつけてあげる仕組みを作ることができます。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/other/20250207/media/result2.gif" vspace="10">

これはそのスロット番号に存在する防具の部位に合わせて、ルートテーブルからエンチャント済みの防具を `/loot give` コマンドで置き換えています。</br>

同じスロット番号に存在するアイテムを差し替えるときには、スロット番号を管理できる型があると良いです。

```ts
type itemSlotStack = {
    itemSlot: number,
    itemStack: ItemStack
};
```

ということで、今回はマイクラのTech寄りの内容でした～👏
