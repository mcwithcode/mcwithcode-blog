# 【Script API】Server UI 2.0.0 の新要素
日本時間 6/18 にマインクラフトの公式クリエイターチャンネルにてアップデート情報が公開されました。Script API の `server-ui` が 2.0.0 がリリースされ、新しいUIコントロールが使えるようになったので触ってみました。

<iframe width="560" height="315" src="https://www.youtube.com/embed/Jpubvfks9As?si=fPKlThWI_vx4GlbL" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
 
# 新要素
[公式の情報](https://learn.microsoft.com/ja-jp/minecraft/creator/documents/update1.21.90?view=minecraft-bedrock-stable)によれば新しく追加されたUIコントロールは

- Header
- Label
- Divider
- Tooltip

の4つです。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/addon/20250621/media/new_ui.webp" vspace="10">

[公式サイト](https://www.minecraft.net/ja-jp/article/minecraft-1-21-90-bedrock-changelog)では変更箇所が細かく記載されているので、参考になると思います。

## Header
ヘッダーを記述することができます。第1引数に文字列を記述します。
```ts
form.header("ここはヘッダー");
```

## Label
ラベル（文字列）を記述することができます。これまで ActionFormData の `body()` メソッドでしか記載できなかったものが、ModalFormData でも使えるようになります。</br>
第1引数に文字列を記述します。
```ts
form.label("ここはラベル1");
form.label("ここはラベル2");
```

## Divider
セクションの仕切り（横線）を入れることができます。
```ts
form.divider();
```

## Tooltip
補足情報を設置できます。これは ModalFormData でのみ使えます。
```ts
const dropdownOption: ModalFormDataDropdownOptions = {
    defaultValueIndex: 0,
    tooltip: "ツールチップ"
}
  
form.dropdown("メニュー", ["A", "B", "C"], dropdownOption);
```

# 使用例
## ActionFormData
ActionFormData は複数の選択肢の中から1つを選び、その結果を使用することができるUIです。Tooltipのみ使えません。

```ts
import { ItemStack, ItemUseAfterEvent, world, Player } from "@minecraft/server";
import { ActionFormData } from "@minecraft/server-ui";

world.afterEvents.itemUse.subscribe((event: ItemUseAfterEvent) => {
  const player = event.source as Player;
  const itemStack = event.itemStack as ItemStack;

  if(itemStack.typeId === "minecraft:diamond") {
    showActionForm(player);
  }
});

function showActionForm(player: Player) {
  const form = new ActionFormData();
  form.title("モーダル");
  form.header("メニュー");
  form.label("ランチメニュー");
  form.button("ハンバーグ定食");
  form.button("焼きサバ定食");
  form.divider();
  form.label("汁物");
  form.button("みそ汁");
  form.button("豚汁（＋100円）");
  form.divider();
  form.show(player);
}
```

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/addon/20250621/media/action_form.webp" vspace="10">

## ModalFormData
ModalFormDataは入力された複数の値を使用することができるUIです。バージョンが2.0系になったことで、入力コントロールの引数にも変更がありました。

例えば、以下のUIコントロールの場合
- Dropdown
- Slider
- TextField
- Toggle

第3引数や第4引数などに追加オプションがありましたが、別途インタフェースが用意されています。この中に Tooltip が追加されました。

```ts
import { ItemStack, ItemUseAfterEvent, world, Player } from "@minecraft/server";
import { ModalFormData, ModalFormDataDropdownOptions, ModalFormDataSliderOptions, ModalFormDataTextFieldOptions, ModalFormDataToggleOptions } from "@minecraft/server-ui";

// ここはアイテムトリガー
world.afterEvents.itemUse.subscribe((event: ItemUseAfterEvent) => {
  const player = event.source as Player;
  const itemStack = event.itemStack as ItemStack;

  if(itemStack.typeId === "minecraft:diamond") {
    showActionForm(player);
  }
});

// UIはここ
function showModalForm(player: Player) {
  const form = new ModalFormData
  form.title("フォーム");
  form.header("メニュー");
  form.label("§dお得なランチセット§r");

  const dropdownOption: ModalFormDataDropdownOptions = {
    defaultValueIndex: 0,
    tooltip: "一つ選んで下さい"
  }
  form.dropdown("定食", ["焼肉定食", "焼きサバ定食", "唐揚げ定食"], dropdownOption);

  const toggleOption: ModalFormDataToggleOptions = {
    defaultValue: false,
    tooltip: "+100円で飲み放題にできます。"
  }
  form.toggle("飲み放題プラン", toggleOption);

  const sliderOption: ModalFormDataSliderOptions = {
    defaultValue: 0,
    tooltip: "不要な場合は0にしてください",
    valueStep: 1
  }
  form.slider("取り皿の枚数", 0, 6, sliderOption);

  const textFieldOption: ModalFormDataTextFieldOptions = {
    defaultValue: "",
    tooltip: "アレルギーがあれば教えてください"
  }
  form.textField("備考欄", "", textFieldOption);
  
  form.divider();
  form.submitButton("注文する");
  form.show(player);
}
```

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/addon/20250621/media/modal_form.webp" vspace="10">

# おわりに
新しいUIコントロールが追加されたことで、より情報量を載せられるようになりました。

- ツールチップはホバーしないと表示されず、表示されると他の要素が隠れる
- ラベルは常に表示状態になるがスペースをとる

あまり使いすぎると逆にわかりにくくなることもあるので、例えばラベルとツールチップの使い分けなどを考えたいところです。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/addon/20250621/media/cover.webp" vspace="10">