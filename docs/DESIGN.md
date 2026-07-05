# musicbattle 設計書 (v1)

- Repository: `github.com/Saber5656/musicbattle`
- Tagline: "Generate competitive intro-quiz battles from your personal music library."
- License: MIT（前提）/ クラウド送信なし / 個人 OSS・最小実装で早期リリース
- 作成日: 2026-07-05

---

## 1. コンセプトと既存イントロクイズとの位置づけ

musicbattle は、ローカルの音楽フォルダから自動でイントロクイズを生成し、
1 台の PC（+ スピーカーや TV 接続）を同席メンバーで囲んで遊ぶ**ホットシート型パーティゲーム**である。
フォルダを選ぶだけで選曲・頭出し・正解表示・スコア管理が自動化され、
これまで「出題者役」だった人も含めて全員がプレイヤーとして参加できる。

対象ユーザーは**ローカル音楽ファイルの所有者**（CD リッピング世代、DL 購入派、Bandcamp / DJ ユーザー）という
ストリーミング全盛期のニッチであることを自覚する。その代わり、配信カタログの「みんなの流行曲」ではなく
**この家・この仲間のライブラリ**から出題されるため、家族の懐メロや友人内の思い出曲で固有の盛り上がりが作れる。
これは既存のどのクイズサービスも提供していない体験である。

**既存のイントロクイズ・音楽クイズとの位置づけ**

| 既存 | 音源 | 前提となるもの | musicbattle との違い |
|---|---|---|---|
| SongPop 等の商用音楽クイズアプリ | 運営が権利処理した配信曲 | アカウント登録・ネット接続・課金 | 音源は自分のフォルダ。オフラインで動き、登録も課金もない |
| ストリーミング API 連携クイズ（OSS に多数） | Spotify 等のカタログ | 対象サービスのアカウント + API キー | 外部 API の仕様変更・提供終了リスクを構造的に負わない。ローカルファイルだけで完結 |
| TV 番組式の人力イントロクイズ | 手元の音源 | 出題者が選曲・頭出し・採点を全部担う | 選曲・頭出し・正解表示・スコアを自動化。出題者もプレイヤーになれる |
| Kahoot! 等の汎用クイズプラットフォーム | 自作問題（音源の扱いは限定的） | 参加者全員のスマホ + ネット接続 | 端末は 1 台だけ。音楽ファイルから問題を自動生成する |

つまり「**自分のライブラリ × 同じ部屋 × 端末 1 台**」に特化したイントロクイズが存在理由。
ネットワーク対戦や早押しシステムで勝負せず、**導入の軽さ（URL を開いてフォルダを選ぶだけ）**で差別化する。

---

## 2. v1 スコープ

| 区分 | 項目 | 備考 / 理由 |
|---|---|---|
| 入れる | ローカル音楽フォルダの選択とスキャン | File System Access API + `webkitdirectory` フォールバック（§4） |
| 入れる | メタデータ読み取り（曲名 / アーティスト / アルバム / カバーアート） | music-metadata。タグ欠落時はファイル名にフォールバック |
| 入れる | クイズセット生成（ランダム N 曲、既定 10 曲） | ライブラリからの重複なし抽選 |
| 入れる | イントロ再生（冒頭 1 / 3 / 5 / 10 秒、既定 5 秒） | 短いフェードで途切れ感を消す（§5）。「もう一度」ボタンあり |
| 入れる | 曲名当てモード（口頭回答） | v1 の核。判定は人間がやる |
| 入れる | 4 択モード | 誤答 3 つを同ライブラリの他曲から自動生成。表記ゆれ正規化つき |
| 入れる | 正解表示（曲名 + アーティスト + ジャケット）と「続きを流す」 | 答え合わせ後にサビへ続けられるのがパーティ的に効く |
| 入れる | ホストによる手動ポイント付与 + スコアボード（2〜8 人） | 誤タップ用の Undo 1 段つき |
| 入れる | リザルト画面と「同じ設定でもう一戦」 | 連戦がパーティの基本動線 |
| 入れる | 完全クライアントサイド / GitHub Pages 配信 | バックエンドなし。§7・§8 |
| 入れない | ネットワーク対戦・ルーム機能 | シグナリングサーバ運用が発生し「完全クライアントサイド」と矛盾。ホットシートに絞る |
| 入れない | スマホ早押しバズァー | **最大の絞り**。早押し判定は遅延補正・接続管理・同期で実装量が数倍になる。同席前提なら「誰が早かったか」は人間が判定でき、ホストのポイント付与で代替できる。v2 の口として §11 の外に残す |
| 入れない | 自動採点（音声認識・テキスト入力判定） | 表記ゆれ判定が沼。ホスト判定の方が確実で、場も暖まる |
| 入れない | イントロ以外の出題（ランダム区間・サビ当て） | 再生開始点の検出が別課題。v2 |
| 入れない | ライブラリの永続キャッシュ（IndexedDB） | v1 はセッション毎スキャンで十分速い設計にする（§5）。v1.x で検討 |
| 入れない | プレイリスト（m3u 等）読み込み | フォルダ選択で十分。入口を増やさない |
| 入れない | 音声の切り出し保存・録音・エクスポート | §7 の著作権への態度と矛盾するため、**v2 以降も含め持たない** |
| 入れない | ストリーミングサービス連携 | アカウント要求・API リスクはコンセプトに反する |
| 入れない | PWA / オフラインインストール | ホスト PC はネット接続が通常ある。v1.x で検討 |
| 入れない | モバイル UI 最適化 | ホスト用途は PC 前提（§3）。閲覧できれば十分 |

---

## 3. 対応プラットフォームと優先順位

ホスト画面 1 枚のゲームなので、対応の軸は OS ではなく**ブラウザ**である。

| 優先度 | 環境 | フォルダ選択 | 判断 | 理由 |
|---|---|---|---|---|
| 1 (v1 推奨) | PC の Chrome / Edge（Chromium 系） | File System Access API（`showDirectoryPicker`） | **推奨環境** | ディレクトリを再帰走査でき、数千曲でも列挙が速い。ホスト用途なら「PC の Chrome で開いてください」で割り切れる |
| 2 (v1 動作対応) | PC の Safari 16+ / Firefox | `<input webkitdirectory>` フォールバック | 対応 | File System Access API 非対応（下記）。毎回フォルダを選び直しになり、巨大フォルダでは列挙が重い旨を UI に明記 |
| 3 | モバイルブラウザ（iOS Safari / Android Chrome） | 実質不可〜不安定 | 見送り | ローカルフォルダアクセスの制約が大きく、スピーカー常設のホスト用途にも合わない。README で非対象と明記 |

**File System Access API の対応状況（調査結果）**: `showDirectoryPicker` は Chrome 86+ / Edge 86+ / Opera 72+ のみ。
Firefox は実ディスクピッカーを標準化ポジションで「harmful」と評価し全バージョン非対応、
Safari も OPFS（Origin Private File System）のみで実フォルダのピッカーは非対応・実装表明なし。
一方 `webkitdirectory` は Chrome 30+ / Firefox 50+ / Safari 11.1+ / Edge 14+ で使えるため、
**入口を 2 系統用意すれば主要デスクトップブラウザを全てカバーできる**（§4 で抽象化）。

**音声コーデック対応表（調査結果）**: スキャン対象拡張子は下表の「対応」形式とする。

| 形式 | Chrome / Edge | Firefox | Safari | v1 の扱い |
|---|---|---|---|---|
| mp3 | ✅ | ✅ | ✅ | 対応 |
| m4a (AAC) | ✅ | ✅（OS デコーダ利用。Linux はコーデック未導入だと不可） | ✅ | 対応 |
| flac | ✅ 56+ | ✅ 51+ | ✅ 11+ | 対応 |
| wav | ✅ | ✅ | ✅ | 対応 |
| ogg / opus | ✅ | ✅ | △ 保証なし | 対応（Safari では再生不可時にスキップ） |
| **DRM 保護ファイル**（Apple Music のダウンロード等） | ❌ | ❌ | ❌ | **再生不可と README・UI に明記**。デコード失敗として除外・自動補充（§5） |

---

## 4. 技術選定

### 比較

**UI / ビルド**

| 候補 | サイズ / 特性 | 判定 |
|---|---|---|
| **Preact + Vite（採用）** | ~4KB。画面 3 枚 + 単純な状態なら十分。静的出力が GitHub Pages と好相性 | ✅ |
| React | エコシステムは厚いがこの規模では過剰。バンドル増 | ❌ |
| Vanilla TS | 可能だが、スコアボード等の状態同期を手書きするコストの方が高い | ❌ |

**メタデータ読み取り**

| 候補 | ブラウザ対応 | タグ対応 | 実績 | 判定 |
|---|---|---|---|---|
| **music-metadata（採用）** | `parseBlob(File)` でブラウザ動作を公式サポート | ID3v1/v2・Vorbis（flac/ogg）・iTunes/MP4（m4a）を網羅。`selectCover` でカバーアート取得 | 週 57 万 DL 規模・活発に保守 | ✅ |
| jsmediatags | ブラウザ可 | ID3 中心。flac/ogg が弱い | 週 4 千 DL 規模で更新停滞 | ❌ |
| 自前 ID3 パーサ | — | ID3v2 だけでも仕様が重く、m4a/flac まで書くのは本末転倒 | — | ❌ |

補足: 旧 `music-metadata-browser` パッケージは不要。現行の `music-metadata` 本体がブラウザ入力（Blob/File）を直接サポートする。

**イントロ再生方式**

| 候補 | 特性 | 判定 |
|---|---|---|
| **HTMLAudioElement + Blob URL（採用）** | ストリーミングデコードで再生開始が速く、曲全体を PCM 展開しないためメモリが小さい。停止精度は数十 ms 程度だがパーティ用途には十分 | ✅ |
| Web Audio `decodeAudioData` | `start(when, offset, duration)` でサンプル精度だが、**曲全体を PCM 展開**するため 1 曲で数十 MB 級・開始も遅い | ❌ 過剰 |
| 折衷: HTMLAudio + `MediaElementSource` → GainNode | 音量フェード（§5）だけ Web Audio に任せる。v1 は `audio.volume` の簡易ランプで開始し、ブツ切り感が気になれば導入 | 保留（v1 では未使用） |

### 採用スタック

| 層 | 技術 | 理由 |
|---|---|---|
| 言語 | TypeScript | クイズ生成・スキャンのデータ契約（Track / Round）を型で固定 |
| ビルド | Vite | 軽量・高速。GitHub Pages 向け静的出力が容易 |
| UI | Preact + 素の CSS | 上表。状態管理ライブラリは入れない |
| フォルダ入力 | File System Access API / `webkitdirectory` の 2 系統を `LibraryLoader` で抽象化 | §3 の対応表どおり主要ブラウザをカバー |
| メタデータ | music-metadata（`parseBlob` + `selectCover`） | 上表 |
| 再生 | HTMLAudioElement + `URL.createObjectURL(file)` | 上表。停止は `timeupdate` 監視 + 音量フェード |
| テスト | Vitest | クイズ生成・正規化（§5）は純関数なので単体テストが容易 |
| CI/CD | GitHub Actions → GitHub Pages | main への push で自動デプロイ |
| ランタイム依存 | **preact と music-metadata の 2 つだけ** | 依存最小を売りにする |

---

## 5. アーキテクチャ

主要コンポーネントは 5 つ。全てブラウザ内で完結し、サーバ側コードは存在しない。

```
┌───────────────────────────────────────────────────────────────┐
│ Browser（完全ローカル・送信ゼロ）                                  │
│                                                               │
│  ┌────────────────┐  showDirectoryPicker() /                  │
│  │ LibraryLoader   │◀─ <input webkitdirectory>                │
│  │ (走査+拡張子filter)│                                          │
│  └───────┬────────┘                                           │
│          │ FileRef[]（列挙のみ・未パース）                         │
│  ┌───────▼────────┐   music-metadata parseBlob                │
│  │ MetadataScanner │──▶ Track{title, artist, album,           │
│  │ (遅延・対象曲のみ) │          coverBlob?, file}                │
│  └───────┬────────┘                                           │
│          │ Track[]（クイズ対象 + 誤答プール）                      │
│  ┌───────▼────────┐                                           │
│  │ QuizEngine      │ 抽選・4択誤答生成・進行状態（純TSモジュール）    │
│  └───────┬────────┘                                           │
│          │ Round{track, choices?}                              │
│  ┌───────▼────────┐      ┌─────────────────────────────┐     │
│  │ IntroPlayer     │      │ UI (Preact)                  │     │
│  │ (HTMLAudio +    │◀────▶│ Setup / Game / Result 画面    │     │
│  │  fade + 秒数制御) │      │ + Scoreboard(手動ポイント付与) │     │
│  └────────────────┘      └─────────────────────────────┘     │
└───────────────────────────────────────────────────────────────┘
```

### クイズ生成フロー（遅延スキャンが肝）

全曲のメタデータを先に読むと 1 万曲級ライブラリで分単位の待ちが発生するため、
**「列挙 → 抽選 → 当選曲だけパース」**の 2 段階にする。

1. **列挙**: フォルダを再帰走査し、対応拡張子（§3 の表）のファイル参照だけを集める。パースしないので数千曲でも数秒以内
2. **抽選**: クイズ対象 N 曲（既定 10）+ 4 択用の誤答プール（3N + 予備）をランダム抽出
3. **パース**: 当選分だけ `parseBlob` でメタデータを読む（進捗バー表示）。読めないファイル・DRM 疑い（デコード不能）は除外し、残プールから自動補充
4. **タイトル正規化**: 曲名を正規化（trim / 大文字小文字 / `feat.〜`・`(Remix)` 等の括弧書き除去）し、正解と同名・別バージョンの曲を誤答候補から除外。タグ欠落時はファイル名（拡張子除去）を曲名として使う
5. **ラウンド列生成**: `Round[]` を確定。4 択モードでは各ラウンドに誤答 3 曲を割り当てる

パースは対象数十曲に限定されるためメインスレッドの async 逐次で開始し、実測で遅ければ Worker 化する（差し替え可能な境界にしておく）。

### 再生制御（IntroPlayer）

```
[再生ボタン]                                  設定秒数 T（1/3/5/10）
   │ click（ユーザージェスチャ起点。autoplay 制限を踏まない）
   ▼
URL.createObjectURL(file) → audio.currentTime = 0 → play()
   │ 音量 0 → 1 のフェードイン（~50ms）
   ▼
timeupdate / rAF で currentTime を監視
   │ T - 0.2s 到達で音量 1 → 0 のフェードアウト → T で pause()
   ▼
[もう一度] → 同区間を再再生（回数無制限。ホストの裁量）
[正解を見る] → 正解表示 + [続きを流す]（pause 位置から再生継続。サビで盛り上がる用）
[次の曲へ] → pause() + revokeObjectURL でメモリ解放 → 次ラウンド
```

- 停止精度は ±数十 ms だがフェードアウトで知覚上は問題にならない。サンプル精度が欲しくなったら §4 の折衷案（GainNode）に差し替える
- 再生はすべてボタンクリック起点に統一し、**自動で次ラウンドの再生を始めない**（autoplay ポリシー回避と、ホストの進行権を兼ねる）
- Blob URL とカバーアート URL はラウンド終了時に必ず `revokeObjectURL`（§10 P2）

### データモデル（コア型）

| 型 | 主フィールド | 備考 |
|---|---|---|
| `FileRef` | `file: File`（webkitdirectory）or `handle: FileSystemFileHandle` | 入口 2 系統の差を吸収 |
| `Track` | `title, artist?, album?, coverBlob?, file` | 正規化済み `titleKey` を持つ |
| `Round` | `track, choices?: Track[4], answered: boolean` | 4 択時のみ choices |
| `Player` | `name, score` | 2〜8 人 |
| `GameSettings` | `introSec(1/3/5/10), rounds(N), mode(call/choice)` | 既定: 5 秒 / 10 曲 / 口頭 |

---

## 6. UI/UX

### 画面遷移（3 画面で完結）

```
[Setup] フォルダ選択 → スキャン → プレイヤー登録 → 設定
   ▼
[Game]  ラウンド進行 × N 曲（再生 → 回答 → 正解表示 → ポイント付与）
   ▼
[Result] 順位表彰 → 「同じ設定でもう一戦」/「フォルダからやり直す」
```

### Setup 画面

- 最初の画面は「**音楽フォルダを選ぶ**」大ボタン 1 個 +
  「ファイルはこの PC の外に出ません 🔒」バッジ（タップで §7 の説明）+ 対応形式の 1 行表記
- Safari / Firefox では自動的に `webkitdirectory` 入口に切り替え、「毎回選び直しになります / Chrome 推奨」を小さく表示
- スキャン結果は「🎵 1,234 曲見つかりました」と件数だけ表示（一覧は出さない。待たせない）
- プレイヤー登録は名前チップの追加式（2〜8 人）。設定（秒数 / 曲数 / モード）は既定値入りで**そのまま「スタート」を押せる**

### Game 画面（ホスト画面 1 枚。TV 出力を想定して文字は大きく）

```
┌──────────────────────────────────────────────┐
│  Round 3 / 10                    ♪ 5秒モード    │
│                                              │
│              ┌────────────┐                  │
│              │     ？      │  ← 正解表示で       │
│              │  (ジャケット) │    ジャケット+曲名   │
│              └────────────┘    +アーティスト     │
│                                              │
│   [▶ 再生]   [🔁 もう一度]   [👁 正解を見る]      │
│   （4択モード時: A〜D の曲名カードを表示）           │
│                                              │
│  ポイント: [ゆう 3] [あき 2] [パパ 0] [誰も正解せず] │
│  （正解表示後に活性化。タップで +1 / Undo 1 段）     │
│                            [次の曲へ →]        │
└──────────────────────────────────────────────┘
```

- ゲーム進行: 再生 →（口頭で回答が飛び交う）→ ホストが「正解を見る」→ ジャケットどんっ + 「続きを流す」→ 正解者のチップをタップ → 次の曲へ
- 同時正解は複数人への付与可。誤タップは Undo で戻す
- キーボードショートカット: `Space` = 再生 / もう一度、`Enter` = 正解表示、`1`〜`8` = 各プレイヤーへ付与、`→` = 次の曲。ホストがキーボードだけで回せる
- 4 択モードでは再生と同時に A〜D を表示（回答は口頭で「B！」と叫ぶ。画面タップはホストのみ）

### 初回体験（勝負は 60 秒）

1. URL を開く → フォルダ選択ボタンとプライバシー 1 行だけの画面（説明を読ませない）
2. フォルダ選択 → 数秒で「1,234 曲見つかりました」→ 名前を 2 つ入れて「スタート」
3. 1 曲目の再生ボタンを押した瞬間がこのアプリのピーク。**ここまで 60 秒以内**を設計目標にする
4. 音源を同梱できない（§7）ためデモモードは作らない。README のデモ GIF が疑似体験を担う

### エッジケースの扱い

| ケース | v1 の挙動 |
|---|---|
| タグなし・タグ破損 | ファイル名（拡張子除去）を曲名として出題。ジャケットはプレースホルダ表示 |
| 再生不可ファイル（DRM / 非対応コーデック） | スキャン時のデコード確認では検出しきれないため、再生エラー時に「この曲は再生できないためスキップします（DRM 保護の可能性）」と表示し予備曲から自動補充 |
| ライブラリが曲数不足 | N > 曲数なら N を自動縮小して通知。4 曲未満なら 4 択モードを無効化 |
| 曲の長さ < 設定秒数 | そのまま曲末尾まで再生（特別処理なし） |
| 冒頭が無音の曲 | v1 は素直に 0 秒から再生（無音スキップは v2） |
| スキャン中のキャンセル | いつでも中断してフォルダ選択に戻れる |

---

## 7. プライバシー設計と著作権への態度

### プライバシー原則

1. **音楽ファイル・メタデータ・再生履歴をブラウザ外に送る経路をそもそも作らない**（バックエンド・API・外部 SaaS が存在しない）
2. **構造的に証明可能にする**（設定でオフではなく、能力として不可能に）
3. **非開発者には UI の 1 行で、開発者には技術で伝える**

| 層 | 施策 |
|---|---|
| アーキテクチャ | 全処理（走査・パース・再生・スコア）をブラウザ内で実行。アップロード API なし |
| CSP | `connect-src 'self'` / `media-src 'self' blob:` / `img-src 'self' blob: data:` を宣言。外部送信はブラウザレベルでブロック |
| 計測ゼロ | アナリティクス・外部フォント・CDN を使わない。GitHub Pages のアクセスログ以上の情報を持たない |
| ファイルアクセス | File System Access API は読み取り専用モードで要求（書き込み権限を求めない） |
| OSS | 全コード公開 + GitHub Actions ビルドで配信物とソースの対応を検証可能 |
| 検証手順の公開 | README に「DevTools → Network タブを開いて 1 ゲーム遊ぶ → 音楽ファイルのリクエストが 1 件も飛ばないことを確認」を記載 |

### 著作権への態度（README にも明記する）

- musicbattle は「**ユーザーが所有する音源を、ユーザーの端末上でその場再生する**」ものであり、立ち位置はメディアプレイヤーと同じ。家庭内や同席の友人との集まりという**私的利用の範囲での再生**を想定する
- **音声の切り出し保存・録音・エクスポート・再配布・ネットワーク送信の機能を一切実装しない**。ファイルの一部たりともアプリの外に出力しない。これは v1 の省略ではなく、プロジェクトの恒久方針とする
- **DRM の回避は行わない**。DRM 保護ファイル（Apple Music のダウンロード等）は「再生できないファイル」として除外されるだけで、解除・変換の機能は持たない
- リポジトリ・デモ・テストに音楽ファイルを同梱しない。デモ GIF は権利上問題のない音源（自作 / CC0）で撮影する
- 公衆の場・営利イベントでの使用が各地域の著作権法上許されるかはユーザー自身の責任である旨を README に免責として記載する

---

## 8. 配布方法

| 項目 | 内容 |
|---|---|
| ホスティング | GitHub Pages（`https://saber5656.github.io/musicbattle/`）。独自ドメインは後回し |
| デプロイ | GitHub Actions: main への push → Vite build → Pages deploy。手作業ゼロ |
| バージョニング | タグ + GitHub Releases（CHANGELOG）。静的アプリのため「更新 = 再訪で最新」 |
| インストール | 不要。「URL を開く」が唯一の導入手順（README の 1 クリック導線） |
| ライセンス | MIT。§7 の著作権姿勢と免責を README に記載 |
| 将来 | PWA 化（v1.x）、`quiz-engine` の npm 切り出しは需要が出た場合のみ |

---

## 9. README 構成案（英語）

```markdown
<バナー画像: ジャケット風タイルとタイトルロゴの横長 PNG>

# musicbattle 🎵
> Generate competitive intro-quiz battles from your personal music library.

<デモ GIF: フォルダ選択 → イントロ再生 → 正解ドン → ポイント付与 の 15 秒画面録画>

**👉 Play now: https://saber5656.github.io/musicbattle/**
No install, no account, no upload — your music never leaves your browser.

## How it works
1. Pick your local music folder (mp3 / m4a / flac / ogg / wav)
2. musicbattle builds an intro quiz — 1 / 3 / 5 / 10 second intros
3. Gather friends around one screen, shout the answer, host awards points

## Features
- 🎧 Plays from YOUR files — family classics beat streaming charts
- 🎛 Title-call mode & 4-choice mode (wrong answers auto-picked from your library)
- 🏆 Hot-seat party scoring for 2–8 players, keyboard-driven hosting
- 🔒 100% client-side. No uploads, no tracking, works with DevTools open

## Browser support
- 表: Chrome / Edge (recommended, folder picker) vs Safari / Firefox (fallback,
  re-select folder each time) / mobile not supported

## Privacy & copyright
- Your files never leave the device (CSP-enforced; verify in the Network tab)
- No recording, clipping, or export features — playback only
- DRM-protected files (e.g. Apple Music downloads) cannot be played
- Intended for private listening with friends at home

## Development（clone / npm i / npm run dev の 3 行）
## License
MIT
```

ポイント: デモ GIF と「Play now」URL がファーストビューに収まること。
バッジは license / deploy (Pages CI) / release の 3 つまで。

---

## 10. リスクと実装前検証項目

| 優先度 | 項目 | 内容 | 検証方法 |
|---|---|---|---|
| P0 | **フォルダ → 再生パイプライン全体の成立** | `showDirectoryPicker` 走査 → music-metadata の `parseBlob`（mp3/m4a/flac の実ファイル + カバーアート取得）→ HTMLAudio での冒頭 5 秒再生・フェード・停止、が実ライブラリで通るか。DRM ファイルがどの段階でどう失敗するか | **捨てプロトタイプを最初に書く（Issue #1）** |
| P0 | 大規模ライブラリでの遅延スキャン性能 | 数千〜1 万曲フォルダで「列挙数秒・クイズ開始まで 10 秒以内」が成立するか。列挙とパースの実測時間を取る | 同プロトタイプ |
| P1 | autoplay ポリシー | 全再生をクリック起点にした設計（§5）で警告・ブロックが出ないこと。「もう一度」「続きを流す」の連続操作も確認 | 同プロトタイプ + 実装時 |
| P1 | `webkitdirectory` フォールバックの実用性 | Safari / Firefox で数千ファイルのフォルダを選択した際の所要時間とメモリ。上限の案内文言が必要か | Safari / Firefox 実機で測定 |
| P1 | DRM ファイルの検出タイミング | 拡張子では判別できないケースがあるため「再生エラー時スキップ + 補充」の UX が破綻しないか（連続スキップ時の体感） | spike で Apple Music ダウンロードファイルを混ぜて確認 |
| P2 | 曲名の表記ゆれによる 4 択破綻 | 同一曲の別バージョン（feat. / Remix / Live）が誤答に混入しないか。正規化ルール（§5）の網羅度 | 実ライブラリでクイズ生成を回して目視 |
| P2 | Blob URL / カバーアートのメモリリーク | 長時間の連戦でメモリが増え続けないか。`revokeObjectURL` の徹底 | DevTools Memory で 3 戦連続を確認 |
| P3 | 名称の衝突 | "musicbattle" 類似の既存プロダクト・商標の簡易確認 | リリース前に検索 |

**最重要リスク**: P0 の 2 件。File System Access API・music-metadata・HTMLAudio の 3 者連結と
遅延スキャンの性能はこの企画の成立条件そのものなので、UI を書く前に捨てられるプロトタイプで必ず確認する。

---

## 11. v1 Issue 分割案（8 個）

- **#1 `Spike: validate folder-to-intro-playback pipeline on a real library`** — ラベル: `spike`, `design`
  UI なしの捨てプロトタイプで、`showDirectoryPicker` 走査 → 抽選 → music-metadata `parseBlob`（カバーアート含む）→ HTMLAudio で冒頭 5 秒再生 + フェード停止、を実ライブラリで検証する。P0 リスク（パイプライン成立・大規模フォルダの列挙/パース性能・DRM ファイルの挙動）を潰す。
  受け入れ条件: mp3 / m4a / flac の実ファイルで再生まで通ること、数千曲フォルダの列挙・パース実測時間と DRM ファイルの失敗挙動が Issue コメントに記録されていること。

- **#2 `Set up TypeScript + Preact + Vite project with Pages deploy`** — ラベル: `infra`
  Vite + Preact + TypeScript の雛形、Vitest、GitHub Actions → GitHub Pages の自動デプロイ、§7 の CSP（`connect-src 'self'` 等）を設定する。
  受け入れ条件: main への push でプレースホルダ画面が `saber5656.github.io/musicbattle/` に公開され、CSP ヘッダ（meta）が有効であること。

- **#3 `Implement library loader with File System Access API and webkitdirectory fallback`** — ラベル: `enhancement`
  `showDirectoryPicker` の再帰走査と `<input webkitdirectory>` の 2 系統を `FileRef[]` に正規化する `LibraryLoader` を実装する。対応拡張子フィルタ、件数表示、キャンセル、非対応ブラウザの自動フォールバックを含む。
  受け入れ条件: Chrome で API 経由・Firefox / Safari でフォールバック経由のフォルダ読み込みが動き、対応外拡張子が除外されること。

- **#4 `Implement lazy metadata scanning with title normalization`** — ラベル: `enhancement`
  列挙 → 抽選 → 当選曲のみ `parseBlob` する遅延スキャン（§5）を実装する。タグ欠落時のファイル名フォールバック、曲名正規化（feat. / 括弧書き除去）、読めないファイルの除外と自動補充、進捗表示を含む。
  受け入れ条件: 数千曲フォルダでクイズ開始まで 10 秒以内（#1 の実測に基づく）。タグなしファイルでも出題可能なこと。

- **#5 `Implement quiz engine with round generation and 4-choice distractors`** — ラベル: `enhancement`
  ランダム N 曲の重複なし抽選、4 択誤答（同ライブラリから 3 曲、正規化キーで正解と同名を除外）、ラウンド進行状態、曲数不足時の縮小処理を純 TS モジュールとして実装する。
  受け入れ条件: Vitest で抽選の重複なし・誤答の同名除外・曲数不足時の挙動が単体テストされていること。

- **#6 `Implement intro player with duration settings and fade control`** — ラベル: `enhancement`
  HTMLAudio + Blob URL による冒頭 1 / 3 / 5 / 10 秒再生、フェードイン / アウト、もう一度、正解表示後の「続きを流す」、再生エラー時のスキップ + 補充、`revokeObjectURL` によるメモリ解放を実装する。
  受け入れ条件: 全再生がクリック起点で autoplay 警告が出ないこと。再生不可ファイルが UI 上スキップされ、3 戦連続でメモリが単調増加しないこと。

- **#7 `Build setup, game, and result screens with manual scoring`** — ラベル: `enhancement`, `ux`
  §6 の 3 画面を実装する。プレイヤー登録（2〜8 人）、ポイント付与チップ + Undo、キーボードショートカット、TV 出力を想定した大きめタイポグラフィ、リザルトと「同じ設定でもう一戦」を含む。
  受け入れ条件: フォルダ選択から 1 ゲーム完走 → 再戦までマウスのみ / キーボードのみの両方で操作できること。

- **#8 `Write README with banner, demo GIF, and play-now link`** — ラベル: `docs`
  §9 の構成で英語 README を作成する。バナー・15 秒デモ GIF（権利上問題のない音源で撮影）・「Play now」1 行導線・ブラウザ対応表・プライバシー / 著作権節（DRM 非対応の明記を含む）を収める。
  受け入れ条件: デモ GIF と Play now リンクがファーストビューに収まり、バッジが 3 個以内で、§7 の著作権姿勢が反映されていること。

推奨着手順: #1 → #2 → (#3, #4 並行) → #5 → (#6, #7 並行) → #8。
スマホ早押しバズァー・ネットワーク対戦は v1 の Issue に含めない（要望が実際に来たら v2 として起票する）。

---

## 参考資料（技術検証の根拠）

- File System Access API の対応状況: [Can I use — File System Access API](https://caniuse.com/native-filesystem-api), [MDN — File System API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API), [MDN — Window.showDirectoryPicker()](https://developer.mozilla.org/en-US/docs/Web/API/Window/showDirectoryPicker), [Chrome for Developers — The File System Access API](https://developer.chrome.com/docs/capabilities/web-apis/file-system-access)
- `webkitdirectory` フォールバック: [Smashing Magazine — Uploading Directories At Once With webkitdirectory](https://www.smashingmagazine.com/2017/09/uploading-directories-with-webkitdirectory/), [LambdaTest — Directory selection from file input](https://www.lambdatest.com/web-technologies/input-file-directory)
- メタデータ読み取りライブラリ: [music-metadata — npm](https://www.npmjs.com/package/music-metadata), [Borewit/music-metadata — GitHub](https://github.com/Borewit/music-metadata), [npm trends — jsmediatags vs music-metadata](https://npmtrends.com/jsmediatags-vs-music-metadata-vs-musicmetadata)
- 音声コーデック対応: [Can I use — AAC audio file format](https://caniuse.com/aac), [TestMu AI — FLAC: Browser Support](https://www.testmuai.com/learning-hub/flac-browser-support/), [Wikipedia — HTML audio](https://en.wikipedia.org/wiki/HTML_audio), [Mozilla Support — Audio and video files in Firefox](https://support.mozilla.org/en-US/kb/audio-and-video-firefox), [MDN — Codec selection (WebCodecs)](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API/Codec_selection)

---

## Changelog

- 2026-07-05: 初版
