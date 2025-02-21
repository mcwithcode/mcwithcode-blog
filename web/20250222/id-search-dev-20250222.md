# ID検索機能の開発裏話
久しぶりの Web 開発まわりのお話です。</br>
エフェクトID検索機能を作るにあたり、悩んだことやら技術的な内容にどう取り組んだかを書いてみます！

# 背景
マイクラのIDリストなんかWeb上で検索すればいくらでも出てくるのですが、例えば Minecraft Wiki とかで調べると、非常に見づらいんですよね。広告が多すぎてスクロールするたびに表示されたり、検索しづらかったり、余計な情報が多かったりと。

適度な広告はいいと思うんですが（このサイトも貼ってますし）、多すぎるとUXが最悪です。そして、Java版と統合版で別になっているサイトやブログもありますよね。なので、全部1箇所にまとめて検索しやすいサイトにしてしまおうと、そう思ったわけです。それに私個人も使いますしお寿司🍣</br>

ということで！IDリストを作ろうと思ったきっかけは</br>

- コマンドやプログラミングでマイクラのIDを使う機会が多々ある
- Java版、統合版のIDを一気に見たい（見比べたい）
- 調べやすい・見やすい・余計な情報がない（純粋にIDだけを調べたい）を作りたい

ですね！

# 設計

## IDをどうやって管理するか
マイクラのIDってかなり数があるので、データベースで保存するか躊躇するところではありました。</br>

最初、ローカルにファイルを生成してそこからリスト作成しようと思っていました。が、結局ファイルが蓄積されていくとサーバーのストレージを圧迫しますし、大量のアクセスが来たときにさばけそうにないかもなーと思い、データベースで管理することにしました。</br>

また、統合版とJava版でIDが異なるものや、存在しない場合にどうするかも考えました。エディション別にページを分けてもいいかなとは思いましたが、導線が増えて操作しづらいのと、IDを比較したい人もいるかと思い、一つのリストに統合しました。</br>

あとはデータベースの選定ですが、単純な情報の羅列であればリレーションの必要がありませんので、今回は NoSQL なデータベースの Azure Cosmos DB を使用することにしました。将来的にエフェクトやアイテムの詳細情報とかを扱う機会があれば、例えばエフェクトのIDを記録しておくテーブルと、エフェクトの詳細を管理するテーブルを繋いで、詳細情報を参照する仕組みにもできます。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/web/20250222/media/uml.webp" vspace="10">

が、とても面倒ですし、そんなときこそ Wiki を見ろって話ですよねｗ</br>

ということで「余計な情報がない」の条件クリアです。

## どうやって見せるか
Table 形式でリストにするか、Card形式でアイコンを目立たせるか悩みました。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/web/20250222/media/design-raf.webp" vspace="10">

でも個人的（ユーザー）にはIDを調べに来ているという目的がありますので、アイコンを目立たせる必要はないですよね。デザイン的にはカードのほうが映えますけど、UX的にはサクッと検索できる＆比較しやすい表形式のほうが良いのかなと。</br>

あとはエディション別とか、バージョンとかの検索条件とかもどうしようかと思っていたのですが、エディションに関しては表なのでまとめて掲載できますし、バージョンの違いなんてあまり気にしないかなと。存在しなければコマンド打ったときに弾かれるので、気にする人はごく少数だと思います。（例えば特定のバージョンでModやプラグイン開発している人が、バニラデータを使うなど）

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/web/20250222/media/design-figma.webp" vspace="10">

結論、「キーワード検索だけでええやんｗ」となりました。これで「調べやすい」と「見やすい」はクリアできたかなと思います。

## 構成図
そんなこんなで今の構成はこんな感じです。画像を byte で管理したとき色々と痛い目を見たので、blob に保存するようにしました。

<img src="https://raw.githubusercontent.com/mcwithcode/mcwithcode-blog/refs/heads/main/web/20250222/media/image.webp" vspace="10">

# 実装

## Cosmos DB への接続
このサイトは ASP.NET Core MVC を使用していますので、ゴリゴリ NuGet パッケージで解決できます。</br>

公式が [Microsoft.Azure.Cosmos](https://www.nuget.org/packages/Microsoft.Azure.Cosmos) を出してくれているので、あとはドキュメントを漁ったり、ChatGPTと相談しながら実装すればなんとかなります。</br>

Cosmos DB は階層になっています。詳しくは [MS Learn](https://learn.microsoft.com/ja-jp/azure/cosmos-db/resource-model) にて。
- データベース (mcwithcode-minecraft-idlist の部分)
- コンテナ (effect の部分)
- アイテム (Items の部分)

データベースの中にいくつかのコンテナ（RDBでいうところのテーブル）が格納されていて、そのコンテナの中にアイテム（RDBでいうところのレコード）が作られていきます。（そういえば昔、コンテナのことをコレクションといっていたような・・・？🤔）</br>

アクセスするまでの実装として、例えばこんな感じに書きます。（このサイトの構成的に管理用と一般公開用でプロジェクトが別れているので、共通機能はライブラリとして別に実装しています。）

```cs
private readonly CosmosClient _client;
private readonly string _databaseName = "データベース名";

// Cosmos DB を使うための準備をコンストラクタでやる
public CosmosDbService(IConfiguration configuration) 
{
    var connectionString = configuration.GetConnectionString("接続文字列");
    _client = new CosmosClient(connectionString);
}

// Cosmos DB からコンテナ情報を取得し、もしなければ新しく作成
private async Task<Container> GetContainer(string containerName) 
{
    var database = _client.GetDatabase(_databaseName);
    await database.CreateContainerIfNotExistsAsync(containerName, "/id");
    return database.GetContainer(containerName);
}

// アイテム（エフェクトID）を取得する処理
public async Task<List<T>> GetItemAsync<T>(string containerName) where T : class
{
    var container = await GetContainer(containerName);
    var queryable = container.GetItemLinqQueryable<T>();

    using var feedIterator = queryable.ToFeedIterator();
    var results = new List<T>();

    while (feedIterator.HasMoreResults)
    {
        var response = await feedIterator.ReadNextAsync();
        results.AddRange(response);
    }

    return results;
}
```

1つ1つ解説していくと長くなりそうなので、まぁ余裕があれば説明する感じでｗ</br>

このアイテムを取得するまでの流れ、結構面倒なので処理を共通化してしまおうということで、ジェネリックメソッドで実装しています。念の為型制約を加えていますが、インターフェースは実装していないのでいつかやろうかなと思ってます・・・。</br>

ジェネリックメソッドを使えば、エフェクトIDだけでなくアイテムIDやらパーティクルIDやらで、モデルが変わったとしても使い回せるのが便利なんですよねぇ。

実際は JSON 形式で保存されているので、例えば 移動速度上昇(speed) に関する情報がほしいときは、

```cs
public async Task<List<T>> GetItemsByKeywordAsync<T>(string keyword, string containerName) where T : class
{
    var container = await GetContainer(containerName);
    var queryable = container.GetItemLinqQueryable<T>(allowSynchronousQueryExecution: true);

    var feedIterator = queryable
        .Where(item => item.ToString().Contains(keyword))
        .ToFeedIterator();

    var results = new List<T>();
    while (feedIterator.HasMoreResults)
    {
        var response = await feedIterator.ReadNextAsync();
        results.AddRange(response);
    }

    return results;
}
```

こんな感じに書いて、引数の `container` に "effect", `keyword` に "speed" と入れれば一致したアイテムを取り出せるわけです。

```json
{
    "id": "31edccac-a297-4c91-bfeb-a5eaaef340c3",
    "texture": "https://hogehoge/manage-texture/effect/speed.png",
    "javaName": "移動速度上昇",
    "javaId": "minecraft:speed",
    "bedrockName": "移動速度上昇",
    "bedrockId": "speed",
}
```

あとはいい感じに加工して View へデータを渡して表形式にするだけですﾈ。

## サーバにキャッシュを持たせる
ただ、上記の実装だと毎回 Cosmos DB にアクセスするじゃないですかぁ・・。何が問題かって**負荷もお金もかかるんですよね！！**</br>
ということで、鯖に一定期間データを保持してもらいます。

```cs
using Microsoft.Extensions.Caching.Memory;

private readonly IMemoryCache _memoryCache;

public IdListController(CosmosDbService cosmosDbService, IMemoryCache memoryCache)
{
    _cosmosDbService = cosmosDbService;
    _memoryCache = memoryCache;
}

public async Task<IActionResult> Effect(string keywords)
{
    var containerName = "effect";
    var cacheKey = "EffectIdList";

    // キャッシュに残ってないときだけ Cosmos DB へリクエスト
    if (!_memoryCache.TryGetValue(cacheKey, out List<EffectIdModel> effectIdList)) 
    {
        effectIdList = await _cosmosDbService.GetItemAsync<EffectIdModel>(containerName);
        _memoryCache.Set(cacheKey, effectIdList, TimeSpan.FromMinutes(60));
    }

    // ここはゴリ押しwww
    if (!String.IsNullOrEmpty(keywords))
    {
        effectIdList = effectIdList
            .Where(item => item.JavaId.Contains(keywords) ||
                item.JavaName.Contains(keywords) ||
                item.BedrockName.Contains(keywords) ||
                item.BedrockId.Contains(keywords))
            .ToList();
    }

    return View(effectIdList);
}

```

てな感じで、ユーザーが最後にアクセスした時間から1時間後にはキャッシュがクリアされる仕組みになっています。キャッシュを保持している間はDBへアクセスしないので、バチクソ早いです。キーワード検索、だいたい 0.08 秒でした。しゅごい👏

# おわりに
デザイン構想やデータベース選定、実装まで色々経験できたいい機会だったかなーと思います。</br>

だんだん趣味レベルから逸脱してきているような・・・そんな気もしますが、やっぱり趣味開発は楽しいですね！</br>

今回はエフェクトIDを公開しましたが、アイテムIDやらパーティクルIDやら、マイクラにはIDが大量にありますからね・・・。カテゴリごとに管理の仕方や見せ方も変えないとなーということで、まったり考えてみます。温泉にでも浸かりながら～♨️

