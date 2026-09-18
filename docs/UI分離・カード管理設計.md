# ルナ🌙ガチャ UI分離・カード管理設計

> この文書は、N05「鏡」UI分離プロトタイプ、カード表示構造監査、段階導入案、読み取り専用カード管理UI案を記録する技術設計書である。
> 調査の正本は `origin/main` の `35032f8a4c492b9ab08d9b754db1f8d4c5a80fe5` とする。
> この文書は設計資料であり、未採用の案を現行実装として扱わない。

## 用語と区分

- **現行仕様**: `origin/main` のコード・素材から確認した事実。
- **採用候補**: N05試作で検討中、または次回以降に採否を判断する案。
- **将来案**: 現時点では実装しない拡張設計。

## 1. 現在のリポジトリ構造とカード管理状況

現行ゲームは `index.html` にHTML、CSS、JavaScript、`CARD_POOL` をまとめた単一HTML構成である。カード画像は `assets/N/` と `assets/R/` に置かれている。

- N: `assets/N/N01_..._large.webp` / `..._mini.webp`
- R: `assets/R/R01_..._large.*` / `..._mini.*`
- SR / SSR / UR: 現時点ではカード用large / mini素材ディレクトリ・素材参照がない。

`CARD_POOL` は `index.html` 内にあり、抽選対象・図鑑対象・素材参照の正本である。現時点で本番ゲームとは別のカードデータファイル、管理台帳、adminページは存在しない。

## 2. レアリティ別の現在の実装状況

| レアリティ | 枚数 | large / mini | 現在のカード表示方式 |
|---|---:|---|---|
| N | 12 | 全カードにlarge / miniあり | 完成画像方式 |
| R | 12 | 全カードにlarge / miniあり | 完成画像方式 |
| SR | 2 | なし | CSSフォールバック |
| SSR | 2 | なし | CSSフォールバック |
| UR | 1 | なし | CSSフォールバック |

現行のSR / SSR / URは、`name` と `quote` を持つ抽選カードであり、完成カード画像ではない。表示時は既存の月アイコン、レアリティ文字、名称、台詞からなるフォールバックUIへ進む。

## 3. large / mini の現在の表示責務

既存の共通関数は `cardAsset(c, role)`、`hasCardMini(c)`、`hasCardLarge(c)`、`cardImage(c, role, className)` である。

| 表示箇所 | 使用role |
|---|---|
| 1連開示 `showCard()` | `large` |
| 10連の旧開始表示 `buildTenStart()` | `mini` |
| 10連タップ開示・完成結果 `renderResultCardFace()` | `mini` |
| 図鑑一覧 `renderCollection()` | `mini` |
| 図鑑詳細 `openCollectionDetail()` | `large` |

画像を持つカードには `hasCardImage` が付き、既存CSSは画像を主表示にする。図鑑詳細では外側のレアリティ、名前、台詞、獲得回数を非表示にするため、完成画像に焼き込まれた情報と重複しない。

## 4. N05「鏡」UI分離プロトタイプ

### 目的

カード画像にロゴ、レアリティ、タイトル、台詞、メッセージを焼き込む方式から、素イラストとHTML/CSS UIを重ねる方式へ移行可能かをN05のみで検証する。

### 試作構造（採用候補）

```text
N05の素イラスト（large / mini）
  +
カードデータ（タイトル、ローマ字、台詞、メッセージ）
  +
共通HTML/CSSオーバーレイ
  +
正式ロゴ画像
```

largeではレアリティ、タイトル、ローマ字、台詞、メッセージ、ロゴを重ね、miniでは台詞とメッセージを表示しない設計とした。Nカードでは枠を設けない前提で試作した。

### 試作結果と制約

- N05専用のlarge / miniプレビューとDEVボタンを隔離worktree内に作成済み。
- 本番の `CARD_POOL`、既存N05完成画像、抽選、図鑑、GitHub mainには未接続・未反映。
- 添付されたロゴ原本は透明PNGではなく白背景を含むJPEGだった。そのため試作では原本画像を`img`で使いつつ、白背景を目立たせない表示調整を行った。
- 正式展開時は、透過PNGの正式ロゴ原本を用意する必要がある。

## 5. UI分離方式の基本方針

**採用候補**として、既存の `large` / `mini` の役割を変えず、画像の中身だけを「完成画像」または「素イラスト + 共通UI」に分岐させる。

- 既存完成画像カード: 従来どおりlarge / miniの画像を直接表示。
- UI分離カード: roleに応じた素イラストへ共通UIを重ねて表示。
- 素材の役割は維持する。10連でlargeを使わず、1連・詳細でminiを使わない。
- 既存N/Rの一括移行は行わない。

## 6. `presentation.mode = 'overlay'` の設計

**採用候補**のカード定義例:

```js
{
  id: 'SR01',
  rarity: 'SR',
  name: 'ルナ【風の音】',
  large: 'assets/SR/SR01_raw_large.webp',
  mini: 'assets/SR/SR01_raw_mini.webp',
  presentation: {
    mode: 'overlay',
    title: '風の音',
    roman: 'Kaze no Oto',
    quote: '……',
    message: '……',
    logoVariant: 'formal-luna',
    frameVariant: 'sr-default'
  }
}
```

`presentation` は任意プロパティとする。未設定カードは既存方式へ戻るため、既存N/Rと互換を保てる。

## 7. presentation各プロパティの役割

| プロパティ | 役割 |
|---|---|
| `mode` | `overlay` のとき共通UI描画を選ぶ。未設定なら完成画像方式またはフォールバック。 |
| `title` | 表示用の日本語タイトル。 |
| `roman` | ローマ字・英字表記。 |
| `quote` | カードの台詞。large中心に表示する。 |
| `message` | 情緒メッセージ・情景文。large中心に表示する。 |
| `logoVariant` | 使用する正式ロゴ素材の種類を選ぶための識別子。 |
| `frameVariant` | レアリティ・デザイン別の共通フレームを選ぶ識別子。 |

miniは原則として台詞・メッセージを省略し、largeは両方を表示する。実際の表示量や位置は正式UIデザイン確定後に決める。

## 8. 既存N/R完成画像方式との互換方針

既存N/Rの `large` / `mini` パスと表示経路は変更しない。N01〜N12およびR01〜R12は、UIが焼き込まれた正式large / miniによる**completed-image方式**として正式完成扱いとする。現時点でoverlay方式への一括移行、作り直し、再マスターは行わない。

N05「鏡」のoverlay試作は新方式の技術検証であり、既存N05の正式完成画像を置き換えない。将来N/Rをoverlay方式へリマスターする可能性は残すが、現在の開発対象には含めない。

`presentation` がないカードは、現在の `cardImage()` と同じ結果を返す分岐を維持する。

この方針により、N/R内に焼き込まれた枠、ロゴ、レアリティ、タイトル、台詞をそのまま保護できる。UI分離カードだけが新しいレンダラーへ進む。

## 9. 新規SRからの段階導入案

1. N05のiPhone実機確認でUI分離方式を評価する。
2. 採用時、共通レンダラーを導入するが既存カードは従来分岐に残す。
3. SR01「風の音」から、素イラスト + overlay UI方式を最初の正式導入候補として実装する。
4. 1連、10連、図鑑一覧、図鑑詳細でlarge / mini責務を確認する。
5. SRの正式frameVariantを確定する。
6. N/Rへの展開は、既存完成画像を維持するか別タスクで判断する。

## 10. 共通カードレンダラー構想

**将来案**として、`renderCardVisual(card, role, context)` を追加する。

```text
presentation.mode === 'overlay'
  → 素イラスト + 共通UIを返す
それ以外
  → 現在と同じ画像またはフォールバックUIを返す
```

`context` は1連、10連、図鑑一覧、図鑑詳細を区別するための補助情報である。素材の選択は常に `role` が担い、large / miniの責務をUI描画層に混在させない。

本番とadminで同じoverlay表示を確認したい場合は、レンダラーを副作用なしの共通スクリプトへ切り出す。

## 11. R以上の正式フレームの現状

**現行仕様として、R以上の独立した正式フレーム素材・共通CSS仕様は未実装である。**

- N/Rは完成画像の中に枠・ロゴ・レアリティ等が焼き込まれている。
- `assets/R/` はlarge / mini完成画像であり、別フレーム素材ではない。
- `assets/SR/`、`assets/SSR/`、`assets/UR/` の完成カード素材はない。
- CSSにはSR/SSR/UR向けの色、背景、発光など、旧フォールバックUIの差分がある。

よって、既存CSSを「正式フレーム」として再利用することはしない。

## 12. frameVariantによる将来のフレーム管理案

**将来案**として、フレームをカードごとに直接CSSで書かず、共通variantとして扱う。

```js
const CARD_UI_VARIANTS = {
  'n-default': { /* 正式決定後のN共通UI */ },
  'sr-default': { /* 正式決定後のSR共通UI */ }
};
```

レンダラーは `cardUi--sr-default` のようなクラスを出力する。正式フレームがPNG/SVG/CSSのどれで確定しても、カードデータは `frameVariant` の値だけを持つ。現時点では `sr-default` のデザイン自体を実装しない。

## 13. 読み取り専用 `admin.html` 構想

**将来案**として、ゲーム画面と分離した静的・閲覧専用ページを追加する。

```text
/admin.html
/assets/admin/admin.css
/assets/admin/admin.js
/assets/admin/card-management.js
```

管理ページはカード編集、保存、CARD_POOL書き換え、GitHub反映を行わない。想定機能は一覧、絞り込み、素材状態、large / miniプレビューである。

静的に公開されるadmin.htmlはアクセス制御ではない。未公開情報や機密メモを置かない。

## 14. CARD_POOLと制作管理メタデータの分離方針

ゲーム用のCARD_POOLは、抽選と実表示に必要なデータだけを持つ。制作管理情報はCARD_POOLへ大量追加せず、IDをキーにした別台帳にする。

```text
CARD_POOL
  ID / rarity / name / large / mini / presentation / storyImages

CARD_MANAGEMENT
  制作状態 / 実装状態 / 実機確認 / 採用状態 / メモ
```

adminはIDで両者を結合して表示する。未実装カードはCARD_POOLに入れず、管理台帳だけに置ける。

## 15. `card-management.js` 構想

**将来案**:

```js
const CARD_MANAGEMENT = {
  SR01: {
    title: '風の音',
    productionStatus: 'large採用',
    implementationStatus: '未実装',
    deviceTestStatus: '未確認',
    adoptionStatus: '候補',
    notes: 'mini制作待ち',
    plannedPresentation: {
      mode: 'overlay',
      frameVariant: 'sr-default'
    }
  }
};
```

本番実装済みカードのID・レアリティ・素材パスはCARD_POOLを優先する。未実装カードの予定タイトルや予定状態は管理データ側で扱う。

## 16. 将来の `cards-data.js` 共有化

adminが本番と同じカード定義を安全に読むため、将来的にはCARD_POOLを副作用なしの共有データへ切り出す。

```text
/assets/cards-data.js
  window.LUNA_CARD_POOL = [ ... ];

index.html
  const CARD_POOL = window.LUNA_CARD_POOL;

admin.html
  const CARD_POOL = window.LUNA_CARD_POOL;
```

`index.html` の文字列をadmin側から解析する方法は壊れやすく、採用しない。カード定義の共有化は、admin実装時に最小変更で行う候補である。

## 17. 将来 `cards.json` へ移行する場合

**将来案**:

```text
/data/cards.json
/data/card-management.json
```

- `cards.json`: ゲームで使うカード定義。
- `card-management.json`: adminのみで使う制作管理情報。

JSON化では非同期読み込み、起動前ロード、失敗時表示、キャッシュを新設計する必要がある。現行の単一HTML構成では、まず `cards-data.js` 共有化のほうが変更量を抑えられる。今回JSON移行は行わない。

## 18. 管理画面で想定する項目と機能

カードごとに以下を一覧表示する。

- ID
- レアリティ
- タイトル
- large / mini素材の有無
- UI方式（完成画像方式 / overlay方式 / fallback）
- frameVariant
- 制作状態
- 実装状態
- 実機確認状態
- 採用状態
- メモ

想定機能:

- N / R / SR / SSR / URで絞り込み
- 制作状態で絞り込み
- large / mini不足の抽出
- miniサムネイル表示
- カード選択によるlarge / miniプレビュー
- UI方式とframeVariantの表示
- 未完成・未実装・未確認カードの一覧

## 19. large / miniの画像サイズ・縦横比の自動検査案

admin表示時に各素材を読み込み、以下を計測・表示する。

- 画像ロード成功 / 失敗
- naturalWidth / naturalHeight
- 実際の縦横比
- 想定role（large / mini）
- 期待比率との差異警告

ただし、既存N/Rの完成画像には比率差や固有デザインがあり得るため、最初は「警告のみ」とする。自動判定で素材を不正・不採用に変更しない。largeとminiの取り違え検出には、ID、role、実寸、プレビューを併用する。

## 20. 現時点のリスク・注意事項

- CARD_POOLと管理台帳を別々に手入力すると、ID、レアリティ、素材パスがずれる危険がある。
- adminと本番が別のoverlayレンダラーを持つと、見た目が一致しない。
- 現行の画像カード用CSSはlargeを概ね1.2、miniを2:3として扱う箇所がある。新素材の比率は管理画面で可視化する必要がある。
- 現行の `title` と `message` はCARD_POOLの一部に存在するが、本番表示コードでは読まれていない。
- `getCardStoryImages()` は将来UR向けの安全な取得関数として存在するが、現時点ではCARD_POOLにstoryImagesを持つカードも、表示機能もない。
- `solemnConfirm()`、旧劇場コード、旧扇状カード開始処理など、主経路ではない旧コード候補が存在する。影響範囲が未確定のため削除しない。
- admin.htmlを公開するだけでは閲覧制限にならない。
- N05試作ロゴの原本問題は、正式透明PNGが支給されるまで本番採用の判断材料に含める。

## 21. 推奨する段階的な実装順序

1. N05 UI分離プロトタイプをiPhone実機で確認する。
2. 採否と正式透過ロゴ素材を確定する。
3. 制作状態の値と管理台帳の項目を固定する。
4. `card-management.js` と読み取り専用admin一覧を追加する。
5. CARD_POOLを副作用なしの `cards-data.js` に共有化する。
6. adminでID結合、絞り込み、素材状態、large / miniプレビューを追加する。
7. 共通カードレンダラーを導入し、既存カードは従来分岐のまま維持する。
8. SR01「風の音」を最初の正式overlayカードとして実装・実機確認する。
9. SRのframeVariantを確定する。
10. N/RやSSR/URへの展開は別タスクで判断する。

## 22. overlay文字レイヤーの配置ルール（採用候補）

### 22.1 N05 v3を今後の調整ベースとする

N05「鏡」UI分離プロトタイプは、第3版の方向性を以後の調整ベースとする。
これは正式本番UIの確定を意味しない。正式制作時にはiPhone実機で、以下をカードごとに調整する。

- フォントサイズ
- 文字位置
- 字間・行間
- 星の大きさと位置
- タイトルブロックの構成
- quote / message の横幅、改行、サイズ、行間、ごく小さな傾き

第3版で使用した書体構成は、現時点の採用候補として次を基準にする。

- title: Klee One
- quote / message: Zen Kurenaido
- roman: Patrick Hand

フォントは外部通信に依存せず、正式導入時はライセンス文を同梱したローカルWebフォントとして扱う。

### 22.2 quote / message は独立したUI文字レイヤーとする

quote / message は、素イラストに焼き込まず、HTML/CSSによる独立したUI文字レイヤーとして配置する。カードごとに素イラストを確認し、人物・表情・手・ポーズ・視線・重要な背景・小物・光源を避けた、最も自然な空きスペースを選ぶ。

- 左下・右下などの固定位置を共通ルールにしない。
- 必要に応じて左右上下、改行、サイズ、行間、横幅、ごく小さな傾きを個別調整する。
- 自然な空きスペースがない場合は、無理にquote / messageを表示せず、配置困難として報告する。
- miniではquote / messageを表示しない。

### 22.3 禁止する表現

文字レイヤーをイラスト世界の一部へ変換しない。以下を禁止する。

- 看板、貼り紙、壁、床、本、ノート、スマホなど、絵の中の小物へ文章を書き込むこと。
- 既存背景文字を書き換えること。
- 吹き出しを追加して、キャラクターの発言に見せること。
- 文章配置のために小物を追加・変更すること。
- 素イラスト自体を加工すること。

この方針は、既存完成画像方式のN/Rを即時移行するものではない。今後のoverlayカードを制作する際の共通UI設計ルールとして扱う。

## 現在地点

- N05 UI分離プロトタイプは作成済み。
- iPhone実機確認はまだ。
- 本番mainには未導入。
- overlay方式は正式採用前。
- N05実機確認後に採否を判断する。
- SR01「風の音」を新方式の正式導入候補として検討中。
- admin管理画面は設計段階で未実装。
- N05 v3は、安定した情報構成に控えめな手描き感を加える採用候補として作成済み。正式導入・実機採否は未決定。
- N01〜N12およびR01〜R12はcompleted-image方式の正式完成カードとして維持する。overlay方式への移行はしない。
- 新規SRは、SR01「風の音」から素イラスト + overlay UI方式の実カード検証を行い、large / miniとiPhone実機表示を通じて正式方式を確定する予定である。
