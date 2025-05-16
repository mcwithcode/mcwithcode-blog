# 【統合版マイクラ】エンティティが持っているアイテムを取得する

[以前の記事](https://www.mcwithcode.com/blog/get-chestblock-items-20250207)で、チェストに入っているアイテムを取得する方法を紹介しましたが、ふと思いました。エンティティの持ち物も取得できるのでは？と。

</br>

もちろん、統合版マイクラなので `data get entity` は使えません。Script API を使って取得していきます。

# エンティティの持ち物を調べる
エンティティを指定する方法はいろいろありますが、今回はネームタグ（名前）を指定する方法でいきます。

```ts
import {
    Entity,
    EntityInventoryComponent,
    ItemUseAfterEvent,
    Player,
    world,
} from "@minecraft/server";
import { ModalFormData } from "@minecraft/server-ui";

world.afterEvents.itemUse.subscribe((event: ItemUseAfterEvent) => {
    if (event.itemStack.typeId === "minecraft:stick") {
        showItemModal(event.source as Player);
    }
});

async function showItemModal(player: Player) {
    const modal = new ModalFormData();
    modal.title("持ち物チェック");
    modal.textField("エンティティの名前", "");
    modal.submitButton("調べる");

    const response = await modal.show(player);

    if (response.canceled) {
        return;
    }

    const nameTag = response.formValues[0] as string;
    getItems(nameTag, player);
}

function getItems(nameTag: string, player: Player) {
    const entities = world.getDimension("overworld").getEntities();
    const entity = entities.find((e: Entity) => e.nameTag === nameTag);

    if (!entity) {
        player.sendMessage(`エンティティ "${nameTag}" が見つかりませんでした.`);
        return;
    }

    const inventory = entity.getComponent("minecraft:inventory") as EntityInventoryComponent;
    if (!inventory || !inventory.container) {
        player.sendMessage(`"${nameTag}" はインベントリを持っていません.`);
        return;
    }

    const container = inventory.container;
    for (let i = 0; i < container.size; i++) {
        const item = container.getItem(i);
        if (item) {
            player.sendMessage(`スロット${i}: ${item.typeId} ×${item.amount}`);
        } else {
            player.sendMessage(`スロット${i} は空です`);
        }
    }
}
```

## 実行結果
まずはプレイヤーの持ち物がこんな状態だったとします。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/addon/20250517/media/cover.webp" vspace="10">

今回は木の棒を振ると、ダイアログが表示されて、名前を指定できるようになっています。調べたいエンティティの名前をいれて実行すると、

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/addon/20250517/media/player_item.gif" vspace="10">

こんな感じで取得できます。

</br>

今度は村人です。村人もインベントリを持っていますので、名前をつけてあげれば持ち物を取得できます。試しにパンを96個渡してみました。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/addon/20250517/media/villager_item.gif" vspace="10">

しっかり取得できていますね。農家さんは種や骨粉なんかも受け取ってくれるので、例えばアイテムの所持状態を検知して、自動で補充（渡す）ような仕組みを作っても面白いかもしれません。

## 解説
今回は木の棒を振ったらモーダルが表示され、そこに名前を記入できるようにしました。`ModalFormData` はGUIの上から順番に値を取得できますので、入力フォームの最初の要素である 0 番目から取得します。安全ため型アサーションを嚙ませています。

```ts
world.afterEvents.itemUse.subscribe((event: ItemUseAfterEvent) => {
    if (event.itemStack.typeId === "minecraft:stick") {
        showItemModal(event.source as Player);
    }
});

async function showItemModal(player: Player) {
    const modal = new ModalFormData();
    modal.title("持ち物チェック");
    modal.textField("エンティティの名前", "");
    modal.submitButton("調べる");

    const response = await modal.show(player);

    if (response.canceled) {
        return;
    }

    const nameTag = response.formValues[0] as string;
    getItems(nameTag, player);
}
```

受けとった名前は string 型として関数の引数へ渡します。まずはワールドに存在するエンティティを全取得し、`find` 関数を使って名前が一致するものを抽出します。

```ts
const entities = world.getDimension("overworld").getEntities();
const entity = entities.find((e: Entity) => e.nameTag === nameTag);
```

一致した場合、そのエンティティの持つコンポーネント `minecraft:inventory` を参照し、アイテムを取得します。このとき、`EntityInventoryComponent` クラスで型アサーションするのがポイントですね。（ここは以前のブロックと同じ）

</br>

`EntityInventoryComponent` クラスには `container` プロパティがあり、ここにアイテムが `ItemStack` 型として格納されます。インベントリの総数は `size` プロパティで取得できるので、for文の条件式で使って配列の要素数を超えたアクセスをしないようにします。

あとは `ItemStack` 型で保存されているアイテム名、数量をそれぞれ `typeId` と `amount` プロパティで参照してあげればOKですね～

```ts
const container = inventory.container;
for (let i = 0; i < container.size; i++) {
    const item = container.getItem(i);
    if (item) {
        player.sendMessage(`スロット${i}: ${item.typeId} ×${item.amount}`);
    } else {
        player.sendMessage(`スロット${i} は空です`);
    }
}
```

</br>

前回はチェストに保存されたアイテム、今回はエンティティに保存されたアイテムを取得する方法についてやってみました！

これができるなら今持っているアイテムを、遠隔地のチェストに入れたり、他のプレイヤーにコピーしたり何かもできそうですねぇ🤔