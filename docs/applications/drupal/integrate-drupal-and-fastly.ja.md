# DrupalとFastlyの統合

## 前提条件

* Drupal 7+
* FastlyサービスID
* パージ権限を持つFastly APIトークン

## URLベースのパージングを持つDrupal 7

1. [Fastly Drupalモジュール](https://www.drupal.org/project/fastly)をダウンロードしてインストールします。
2. FastlyサービスIDとAPIトークンを設定します。
3. 必要に応じてWebhooksを設定します（例えば、キャッシュパージが送信されたときにSlackにピングを送るなど）。
4. Drupal 7ではURLベースのパージングのみが可能です（シンプルパージング）。
5. `settings.php`でDrupalのクライアントIPを変更します：

```php title="Drupal 7のためのsettings.phpの変更"
$conf['reverse_proxy_header'] = 'HTTP_TRUE_CLIENT_IP';
```

## キャッシュタグパージングを持つDrupal 10+

Composerを使用してモジュールの最新バージョンを取得します：

```bash title="Fastly Drupalモジュールと依存関係のダウンロード"
composer require drupal/fastly drupal/http_cache_control drupal/purge
```

次のモジュールを有効化する必要があります：

* `fastly`
* `fastlypurger`
* `http_cache_control` (2.x)
* `purge`
* `purge_ui` (技術的にはオプションですが、本番環境で有効にしておくと非常に便利です)
* `purge_processor_lateruntime`
* `purge_processor_cron`
* `purge_queuer_coretags`
* `purge_drush` (Drを通じてパージするために便利です) Drush、ここに[コマンドのリスト](https://git.drupalcode.org/project/purge/-/blob/8.x-3.x/README.md#drush-commands)があります。

### DrupalのFastlyモジュールを設定する

FastlyのサービスIDとAPIトークンを設定します。サイトIDは自動的に生成されます。ランタイム環境変数を使用するか、`/admin/config/services/fastly`で見つけることができる設定フォームを編集することができます：

* `FASTLY_API_TOKEN`
* `FASTLY_API_SERVICE`

#### パージオプションを設定する

* キャッシュタグのハッシュ長：4
* パージ方法：ソフトパージを使用する

ほとんどのサイトでは`4`文字のキャッシュタグが十分で、_数百万_のエンティティを持つサイトでは`5`文字のキャッシュタグが適している可能性があります（キャッシュタグの衝突を減らすため）。

!!! 注意
    ソフトパージングを使用するべきです。これは、Fastlyのオブジェクトが完全に追い出されるのではなく、古いものとしてマークされ、元の場所がダウンしている場合に使用できるようになることを意味します（[古くなったものを提供する](https://developer.fastly.com/solutions/tutorials/stale/)機能と共に）。

![パージング用のFastly管理UI。キャッシュタグの長さとソフトパージの使用に関する設定オプションを示しています](../images/fastly-cachetag.png)

#### 古いコンテンツのオプションを設定する

サイトに適したオプションを設定します。最小1時間（` 3600`）、最大1週間（`604800`）。一般的には以下のような設定が適しています：

1. 再検証時にステール - オン、`14440`秒
2. エラー時にステール - オン、`604800`秒

![Fastly管理者UIのステール設定](../images/fastly-swr.png)

必要に応じてウェブフックを設定します（キャッシュパージが送信されたときに例えばSlackにピングを送ることができます）。

![Fastly管理者UIのウェブフック設定](../images/fastly-webhook.png)

### Purgeモジュールの設定

パージページ`/admin/config/development/performance/purge`を訪れます。

以下のオプションを設定します：

#### キャッシュ無効化

* Drupal Origin: タグ
* Fastly: E、タグ、URL

![パージ管理者UIのパージャー設定](../images/fastly-invalidate.png)

#### キュー

* キューワー：コアタグキューワー、パージブロック（複数）
* キュー：データベース
* プロセッサ：コアプロセッサ、レイトランタイムプロセッサ、パージブロック（複数）

![パージ管理者UIのキュー設定](../images/fastly-queue.png)

これは、Drupalの組み込みコアタグキューワー（キューにタグを追加）を使用し、キューはデータベース（デフォルト）に保存され、キューは次のものによって処理されることを意味します。

* Cronプロセッサ
* レイトランタイムプロセッサ

Cronプロセッサが動作するためには、Cronが動作することを確認する必要があります。 あなたのサイトで実行します。理想的には毎分です。`cli`ポッドで手動で実行して、`purge_processor_cron_cron()`がエラーなく実行されていることを確認できます。

```bash title="cronの開始"
[drupal8]production@cli-drupal:/app$ drush cron -v
...
[notice] purge_processor_cron_cron()の実行を開始、node_cron()の実行は21.16msかかりました。
```

`遅延ランタイムプロセッサ`は各ページ読み込みの`hook_exit()`で実行され、これはキューに入ってくるほぼ同時にパージを処理するのに有用です。

両方を持つことで、パージができるだけ早く行われることを保証します。

### 最適なキャッシュヘッダー設定

初期設定では、DrupalはブラウザとFastlyで異なるキャッシュ寿命を設定する力を持っていません。したがって、Drupalで長いキャッシュ寿命を設定すると、ブラウザがページをキャッシュしている場合、エンドユーザーはそれらを見ることができません。[HTTP Cache Control](https://www.drupal.org/project/http_cache_control)モジュールの`2.x`バージョンをインストールすると、キャッシュの何がどれだけの期間有効であるかについて、より多くの柔軟性を得ることができます。

ほとんどのサイトでは、適切なデフォルト設定は次のとおりです。

* 共有キャッシュの最大寿命：1ヶ月
* ブラウザキャッシュの最大寿命：10分
* 404キャッシュの最大寿命：15分 分
* 302キャッシュの最大寿命：1時間
* 301キャッシュの最大寿命：1時間
* 5xxキャッシュの最大寿命：キャッシュなし

!!! 注意
    これは、あなたのサイトがページ上に存在するすべてのコンテンツを表現する正確なキャッシュタグを持っていることに依存しています。

### 真のクライアントIP

私たちはFastlyを設定して、実際のクライアントIPをHTTPヘッダー`True-Client-IP`で返すようにしています。`settings.php`で以下の変更を行うことで、Drupalがこのヘッダーを尊重するように設定することができます：

```php title="Drupal < 8.7.0用のsettings.phpの変更"
$settings['reverse_proxy'] = TRUE;
$settings['reverse_proxy_header'] = 'HTTP_TRUE_CLIENT_IP';
```

しかし、Drupal 8.7.0では、[この機能を削除する変更がありました](https://www.drupal.org/node/3030558)。以下のスニペットを使用して同じ目標を達成することができます

```php title="Drupal >= 8.7.0用のsettings.phpの変更"
/**
 * DrupalにTrue-Client-IP HTTPヘッダーを使用するよう指示します。
 */
if (isset($_SERVER['HTTP_TRUE_CLIENT_IP'])) {
  $_SERVER['REMOTE_ADDR'] = $_SERVER['HTTP_TRUE_CLIENT_IP'];
}
```

### Drush統合

```php title="settings.php"
 fastly:
   fastly:purge:all (fpall)                                                    サービス全体をパージ。
   fastly:purge:key (fpkey) キーでキャッシュをパージする。
   fastly:purge:url (fpurl)                                                    URLでキャッシュをパージする。

## cURLを使用してFastlyキャッシュヘッダーを表示する

この関数を使用してください：（LinuxとMac OSXで動作します）

```bash title="cURL function"
function curlf() { curl -sLIXGET -H 'Fastly-Debug:1' "$@" | grep -iE 'X-Cache|Cache-Control|Set-Cookie|X-Varnish|X-Hits|Vary|Fastly-Debug|X-Served|surrogate-control|surrogate-key' }
```

```bash title="Using cURL"
$ curlf https://www.example-site-fastly.com
cache-control: max-age=601, public, s-maxage=2764800
surrogate-control: max-age=2764800, public, stale-while-revalidate=3600, stale-if-error=3600
fastly-debug-path: (D cache-wlg10427-WLG 1612906144) (F cache-wlg10426-WLG 1612906141) (D cache-fra19179-FRA 1612906141) (F cache-fra19122-FRA 1612906141)
fastly-debug-ttl: (H cache-wlg10427-WLG - - 3) (M cache-fra19179-FRA - - 0)
fastly-debug-digest: 1118d9fefc8a514ca49d49cb6ece04649e1acf1663398212650bb462ba84c381
x-served-by: cache-fra19179-FRA, cache-wlg10427-WLG
x-cache: MISS, HIT
x-cache-hits: 0, 1
vary: Cookie, Accept-Encoding
```

上記のヘッダーから次のことが分かります：

* HTML ページはキャッシュ可能です
* ブラウザはページを601秒間キャッシュします
* Fastlyはページを32日間（`2764800`秒）キャッシュします
* 階層型キャッシュが適用されています（エッジPoPはウェリントン、シールドPoPはフランスに位置しています）
* HTMLページはエッジPoPでキャッシュヒットしました

### Fastlyに手動でパージリクエストを送信する

特定のページを手動でキャッシュから削除したい場合、方法があります。

```bash title="単一のURLでFastlyをパージ"
curl -Ssi -XPURGE -H 'Fastly-Soft-Purge:1' -H "Fastly-Key:$FASTLY_API_TOKEN" https://www.example.com/subpage
```

キャッシュタグでパージすることもできます：

```bash title="キャッシュタグでFastlyをパージ"
curl -XPOST -H 'Fastly-Soft-Purge:1' -H "Fastly-Key:$FASTLY_API_TOKEN" https://api.fastly.com/service/$FASTLY_API_SERVICE/purge/<surrogatekey>
```

また、これを少し楽にするために[Fastly CLI](https://github.com/fastly/cli)を使用することもできます。