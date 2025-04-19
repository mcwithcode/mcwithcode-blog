# ID検索機能の開発裏話 - Azure SQL DB 編

# 背景
これは以前の記事でもお話した要素も含むのですが、

- エンチャントIDをJava版、統合版で一気に調べたい
- 調べやすさと見やすさを両立したい

がメインですね。Wikiとかでも検索はできるのですが、やはり広告が多すぎるのと画面レイアウトが微妙で、検索がしづらいです。（多分広告を貼るスペースを確保しているためだと思われる。）

なので、こういった問題を解消したい＆自分の使い勝手の良い形にしたいなぁという経緯です～。

# 設計

## IDをどうやって管理するか
今回は SQL を使うことにしました。理由としてはエンチャントのIDリストに加えて、レベルごとの効果、エンチャント可能なアイテムなどを管理したかったからですね。

マイクラのアプデで、もしかするとエンチャントレベルが変わったり、アイテムが追加されたりする可能性もありますので、これをそれぞれのテーブルで管理できたらなーと考えました。

- エンチャントID (EnchantIdList)
- エンチャントのレベル別の効果 (EnchantLevels)
- エンチャント対象となるアイテムリスト (EnchantTargetItems)
- エンチャント可能なアイテムリスト (EnchantableItems)

この4つのテーブルを使用し、それぞれこのようなリレーションで構成しています。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/web/20250420/media/erd.webp" vspace="10">

命名はちょっと悩みました。例えばエンチャント可能なアイテムリストを Items にすると、今後アイテムIDリストを作りたいときに重複してしまいますし、かといって EnchantableItems にすると、エンチャントと対象となるアイテムの結びつけに使う中間テーブルの命名が難しくなってしまいます。

エンチャントIDと、エンチャント対象のアイテムリストは多対多の関係になります。例えば「耐久力」のエンチャントはどんな防具や武器にもエンチャントができますね。なので、耐久力のエンチャントから見て、その対象（アイテム）は多数になります。

反対に、アイテムにはエンチャントを複数つけることができますね。例えば、剣をエンチャントするとなったら「耐久力」や「ドロップ増加」、「修繕」をつけることもあります。となれば、アイテムから見て、その対象（エンチャント）は多数になります。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/web/20250420/media/relation.webp" vspace="10">

多対多を構成する場合は中間テーブルで解決できるので、外部キーを持たせてリレーションします。

```cs
// EnchantIdList --- EnchantLevels
builder.Entity<EnchantLevels>()
    .HasOne(e => e.EnchantIdList)
    .WithMany(e => e.EnchantLevels)
    .HasForeignKey(e => e.EnchantId)
    .OnDelete(DeleteBehavior.Cascade);

// EnchantIdList --- EnchantableItems
builder.Entity<EnchantableItems>()
    .HasOne(e => e.EnchantIdList)
    .WithMany(e => e.EnchantableItems)
    .HasForeignKey(e => e.EnchantId)
    .OnDelete(DeleteBehavior.Cascade);

// EnchantTargetItems --- EnchantableItems
builder.Entity<EnchantableItems>()
    .HasOne (e => e.EnchantTargetItem)
    .WithMany(e => e.EnchantableItems)
    .HasForeignKey(e => e.EnchantTargetItemId)
    .OnDelete(DeleteBehavior.Cascade);
```

これでマイグレーションしてDBを更新します。そして、DBへ書き込む際には EnchantLevels テーブルも使用しますので、Modelでの処理はこんな感じになります。

```cs
public async Task CreateEnchantIdAsync(EnchantIdViewModel vm)
{
    // ここでデータを書き込む（省略）
    var model = new EnchantIdModel{ } 

    // ここで中間テーブルへ書き込む
    model.EnchantableItems = vm.SelectedTargetItemIdList
        .Select(itemId => new EnchantableItems
        {
            EnchantTargetItemId = itemId,
            EnchantIdList = model
        }).ToList();

    _dbContext.EnchantIdList.Add(model);
    await _dbContext.SaveChangesAsync();
    
    // ここでレベル別の内容を書き込む
    foreach (var (description, i) in vm.EnchantLevelDescriptions.Select((d, i) => (d, i))) 
    {
        var level = new EnchantLevels
        {
            EnchantId = model.Id,
            Level = i + 1,
            Description = description,
        };
        _dbContext.EnchantLevels.Add(level);
    }
    await _dbContext.SaveChangesAsync();
}
```

カスケードになっているので、削除の際には親レコードを削除すると自動的に子レコードも削除されます。

## どうやって見せるか
エンチャントIDを調べやすいように表形式にしました。あとはエンチャントをするコマンドを打つ時に、「このアイテム、エンチャントできたっけ？」問題を解消するために、エンチャント可能なアイテムも表示しようと考えていました。が、表形式のまま表示させようとすると横長になって見づらいので、ボタンを押したら詳細を表示できたほうがいいなと思い、モーダルを実装しました。ページ遷移すると戻る操作が増えるのでストレスになったりしますね。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/web/20250420/media/modal.webp" vspace="10">

マイクラのバージョン 1.20 から名称がJava版と統合されたようで、統合版名称は不要かと思いましたが翻訳が不十分なものもあるようで、用意しておいて正解でした。名称が同じ場合はJava版の名称のみをタイトルに表示するようにしています。

## 管理画面の設計と実装
今度は管理画面のほうですが、今回はエンチャント対象のアイテムと、エンチャントID、レベル別の説明と色々記入するものがありますので大変です。

画面遷移図はこんな感じです。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/web/20250420/media/admin.webp" vspace="10">

順番としてはエンチャント対象のアイテムを追加してから、エンチャントIDを登録していきます。設計上ではエンチャント可能アイテムがNULLでも、エンチャントIDの登録自体はできるようになっているので、あとから結びつけもできます。

エンチャントレベル別の説明については、レベルが2以上になったときに入力フォームが出て記入できるようにしました。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/web/20250420/media/admin2.webp" vspace="10">

実装方法としてはこんな感じです。

```js
<div id="enchantLevelContainer" style="display: none; flex-direction: column; gap: 5px;">
    <p><label class="label-dark" style="margin-right: 5px;">レベルごとの効果</label></p>
    <div id="enchantLevelField"></div>
</div>

function selectedMaxLevel(existingDescriptions = []) {
    const maxLevel = parseInt(document.getElementById("maxLevel").value, 10);
    const enchantLevelContainer = document.getElementById("enchantLevelContainer");
    const effectContainer = document.getElementById("enchantLevelField");

    effectContainer.innerHTML = "";

    if (maxLevel >= 2) {
        enchantLevelContainer.style.display = "flex";
        for (let i = 1; i <= maxLevel; i++) {
            const field = document.createElement("div");
            field.style.display = "flex";
            field.style.gap = "10px";
            field.style.alignItems = "center";
            field.style.marginBottom = "10px";

            const value = existingDescriptions[i - 1] || "";

            field.innerHTML = `
                <p style="flex: none;"><strong>レベル${i}</strong></p>
                <input class="text-box" name="EnchantLevelDescriptions[${i - 1}]" value="${value}" 
                style="width: 100%;" type="text" placeholder="例）効果が${i * 2}%上昇します。" />
            `;

            effectContainer.appendChild(field);
        }
    } else {
        enchantLevelContainer.style.display = "none";
    }
}
```

ここまでをまとめまして、左が皆さんが見ている閲覧用ページ、右が管理用ページで表示してみます。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/web/20250420/media/display.webp" vspace="10">

管理側で設定したエンチャントの詳細を、モーダルで表示できる仕組みになっています。ちなみに、今回もサーバーキャッシュを使用しているので一度アクセスすればかなり早いはず・・？です！

# おわりに
今回は画面デザインよりもDB設計が大変でした。最初は NoSQL でもいいのかなとは思ったのですが、将来的に「エンチャント対象アイテム」をもとに、エンチャント可能なIDリストを逆引き？検索できるようにしたいので SQL を選択しました。

結果として、エンチャント対象アイテムが増えたり、エンチャントレベルが変更になったときに柔軟に対応できそうなので良かったかなと思っています。

最近、機能面ばかりのアプデが多いので資料まわりも充実させていきたいですね～💪