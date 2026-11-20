# ChMate UA Spoof (Morphe patch)

ChMate 0.8.10.226 dev 用の、**Haiagaru と併用する** User-Agent 偽装アドオンパッチ（Morphe `.mpp`）です。
Haiagaru 本体のソース・パッケージは一切変更せず、独立した名前空間・設定ファイルで動作します。

*English summary at the end.*

> 非公式のサードパーティ製パッチです。ChMate・Haiagaru の作者とは無関係です。

## 対応バージョン（これ以外は非対応）

| 対象 | バージョン |
| --- | --- |
| ChMate | **0.8.10.226 dev**（versionCode 494, `jp.co.airfront.android.a2chMate`）のみ |
| Haiagaru | 公式 **patches-1.46.0.mpp**（areteruhiro/Haiagaru-Morphe リリース 1.46.0）で検証 |
| 0.8.10.191 dev | 未対応（将来対応予定） |
| 0.8.10.242 dev | **非対応** |

検証に使ったストック APK の SHA-256: `7290a076ec43ed37a7a7376e9240770577dd13b29f50635c4cd1112a0f4dd9c4`

## 機能

- ON/OFF（**初期値 OFF**）。OFF の間は Haiagaru のみの場合と同じ動作です。
- 任意の UA 文字列を入力可能（1〜1024 文字の半角英数記号。改行・全角は拒否）。何度でも変更でき、「元に戻す」で OFF＋入力消去＋`http.agent` 復元。
- プリセット: Dalvik Android 14 / 15 / 16、ブラウザ（端末の WebView 標準 UA）、旧 `Dalvik/2.1.0 (Linux; U; Android 4.0.3; HT-01 Build/XYZ0.123456.789)`。
- ON の場合:
  - 起動時（`Application.attachBaseContext` の先頭）に `System.setProperty("http.agent", 選択UA)`。
  - ChMate ネイティブ通信の **User-Agent ヘッダ全体** を選択した文字列そのものに置換（閲覧の GET/HEAD、書き込み・認証の POST、画像など）。`Monazilla/1.00 2chMate/…` の接頭辞も付きません（必要なら入力欄に自分で含めてください）。
  - ChMate ネイティブの `WebSettings.setUserAgentString` 呼び出し（Cloudflare 確認ダイアログの WebView、アプリ内ブラウザ）も選択 UA に揃えます。Cookie には触れません。
- 実行中に OFF へ切り替えた場合、`http.agent` を元の値へ戻し、起動時に偽装 `http.agent` から作られた UA 文字列の末尾も元に戻します（OFF のまま起動した場合は何も変更しません）。
- 設定ファイル `chmate_ua_spoof`（キー `ua_spoof_enabled` / `ua_spoof_agent`）は独自のもので、Haiagaru・ChMate の設定とは共有しません。

## 優先順位とフック位置

| 境界 | フック | 順序 |
| --- | --- | --- |
| okhttp3 Request（`o.zzdA`、URL `h`・メソッド `c`・ヘッダ `a`）の全コンストラクタ（3個） | `UaSpoof.prepareRequest(Object)` | Haiagaru `ViewingUserAgent.prepareRequest` → **本パッチ** → return |
| `java.net` の `connect`/`getInputStream`/`getOutputStream`/`getResponseCode`（177 箇所） | `UaSpoof.prepare(URLConnection)` | Haiagaru `ViewingUserAgent.prepare` → **本パッチ** → 元の呼び出し |
| `WebSettings.setUserAgentString`（`o.isOpenJSSEPreferred`＝Cloudflare 確認ダイアログ、`o.TTLandingPageActivity7`＝アプリ内ブラウザ） | `UaSpoofSettings.setUserAgentString` | ネイティブ値を置換。その後 Haiagaru の `ReadingAuth.loadChallenge` がそのまま動作 |
| `RoidonApp.attachBaseContext` 先頭 | `UaSpoofSettings.init(Context)` | `http.agent` 設定 |

本パッチのフックはすべて Morphe の `finalize` で挿入します。`finalize` は全パッチの `execute` 完了後に実行されるため、**パッチ名や選択順に関係なく** Haiagaru のフックの直後（＝最後）に入ります。パッチ済み APK で順序を静的に検証しています（`scripts/VerifyPatchedApk.java`）。

決定順:

1. **bbs.eddibb.cc の Haiagaru ReadingAuth 保存済みペア UA**（認証済みの読み込み GET/HEAD）: 一切置換しません。判定には Haiagaru の `app.morphe.extension.chmate.ReadingAuth.agent(url, method)` を限定的なリフレクションで呼び出します（読み取りのみ）。WebView でも、渡された UA が保存済みペア UA と一致する場合はそのまま残します。
2. ON: 上記以外の http(s) リクエストは選択 UA に置換（Haiagaru の閲覧用 UA 選択より後に適用されるため、本パッチが優先）。
3. OFF: 変更なし。

eddibb の読み込み認証（Haiagaru の `ReadingAuth.loadChallenge`）は従来どおり端末の WebView 標準 UA で行われ、成功時に保存されるペアと `cf_clearance` の対応は維持されます。認証完了前の確認画面（eddibb 以外のネイティブ Cloudflare 確認を含む）は、ネイティブの UA 設定時点で偽装 UA に揃うため、確認後の通信と同じ UA になります。

既知の注意点: ON の場合、Haiagaru が `files.catbox.moe` 画像に付けるブラウザ UA も選択 UA で上書きされます。画像が読めない場合は「ブラウザ」プリセットを使ってください。

## 必要なもの・使い方

1. ChMate **0.8.10.226 dev** のストック APK（上記 SHA-256 と一致するもの）。
2. Morphe Manager（または Morphe Desktop）。
3. 公式 Haiagaru `patches-1.46.0.mpp` と、本リポジトリのリリースの `patches-0.1.0.mpp` を両方追加し、両方のパッチを選択してパッチします（本リポジトリ URL `https://github.com/rikunode/chmate-ua-spoof-patches` をパッチソースとして追加することもできます）。
4. インストール後、ランチャーに **「UA Spoof」** アイコンが追加されます。そこで UA を選び、スイッチを ON にして「保存して適用」。確実に反映するには ChMate（Haiagaru）を再起動してください。

Morphe Desktop の例（署名鍵は各自のもの）:

```sh
java -jar morphe-desktop-<version>-all.jar patch \
  -p=patches-1.46.0.mpp -p=patches-0.1.0.mpp \
  -o=chmate-haiagaru-uaspoof.apk chmate-0.8.10.226-dev.apk
```

## ⚠️ 注意

- **Cloudflare の確認中は UA を変更しないでください。** 確認（`cf_clearance`）は UA と結び付いています。
- **どんぐりの育成中は UA を固定してください。** 変更すると状態が引き継がれない可能性があります。
- **UA を変えても IP 規制（IP BAN）は解除されません。**
- 各掲示板の利用規約に従ってください。本パッチの使用は自己責任です。

## 検証状況（v0.1.0）

検証済み:

- MPP は Android `classes.dex`（パッチ本体）と DEX 形式の拡張 `extensions/chmateuaspoof.mpe`（R8 で本パッケージのみ）を含みます。
- Morphe Desktop 1.19.1 で、ストック 226 APK に公式 Haiagaru 1.46.0 ＋本 MPP を適用し、エラーなし（Haiagaru／Save ChMate crash logs／Change ChMate package name／ChMate UA Spoof）。
- パッチ済み APK の静的検証: 3 コンストラクタすべてで「Haiagaru → 本パッチ → return」、177 箇所すべてで「Haiagaru prepare → 本パッチ → 元の呼び出し」、WebView 2 箇所、Haiagaru のフック数（`ViewingUserAgent` 293 箇所・`ReadingAuth` 3 箇所）が Haiagaru 単体の APK と同一。
- JVM テスト（`tests/jvm`、33 項目）: ローカル UA エコーサーバに対する GET/POST/HEAD、ON/OFF、ReadingAuth ペア優先（テストダブル使用）、実行中の OFF 復元、入力検証、プリセット。
- エミュレータ（Android 16 / API 36 x86_64、ローカルのエコーサーバのみ。実在の掲示板には送信していません）: 起動・設定画面の表示・保存がクラッシュなしで動作。ON ではネイティブの dat GET・SETTING.TXT GET・bbs.cgi POST がすべて選択 UA そのものになり、OFF（元に戻す）では 3 通信の UA が Haiagaru 単体と完全一致。

未検証:

- 実機（エミュレータのみ）、および Android 版 Morphe Manager でのパッチ（Morphe Desktop のみ）。
- 実際の Cloudflare 確認・bbs.eddibb.cc の ReadingAuth 認証フロー、アプリ内ブラウザの WebView UA の実行時動作（静的検証とテストダブルのみ）。
- 画像取得の実行時動作（フックは同じ okhttp／java.net 境界）。

## ビルド

JDK 17+ と Android SDK（`ANDROID_HOME`）が必要です。Morphe のビルドロジックは Git サブモジュールで固定しています。Morphe の Gradle プラグインは `GITHUB_ACTOR`/`GITHUB_TOKEN` が未設定だと失敗するため、何らかの値を設定してください（このビルドでは GitHub Packages の認証は不要でした）。

```sh
git clone --recurse-submodules https://github.com/rikunode/chmate-ua-spoof-patches.git
cd chmate-ua-spoof-patches
GITHUB_ACTOR=anonymous GITHUB_TOKEN=unused ./gradlew :patches:generatePatchesList
# => patches/build/libs/patches-0.1.0.mpp
bash tests/jvm/run.sh
java -cp <smali-dexlib2.jar>:<guava.jar> scripts/VerifyPatchedApk.java <patched.apk> [<haiagaru-only.apk>]
```

## English summary

Add-on Morphe patch for **ChMate 0.8.10.226 dev only**, designed to be applied together with the official Haiagaru 1.46.0 bundle (191 dev: planned; 242 dev: unsupported). Default OFF. When ON it sets `http.agent` at startup and replaces the **entire** User-Agent of native ChMate requests (GET/HEAD, POST posting/auth, images) and native WebView UA calls with the user-chosen string. Haiagaru's saved bbs.eddibb.cc ReadingAuth paired UA always wins for authenticated reads. All hooks are inserted in `finalize`, so they always run right after Haiagaru's hooks. Settings: separate launcher entry "UA Spoof". Do not change the UA during a Cloudflare challenge, keep it fixed while growing donguri, and note that changing the UA cannot lift IP bans.

## License

GPLv3. See [LICENSE](LICENSE). Morphe build logic (submodules) belongs to its respective authors.
