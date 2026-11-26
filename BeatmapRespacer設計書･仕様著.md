## 📄 仕様書・設計書

独立ツール（名称：**`BeatmapRespacer`**）のシステム仕様書・設計書です。

---

### 1. システム概要と要件定義

#### 1.1 目的

`BeatmapLinker(旧称：osuLazer2TaikoNauts)` で同期された `osuTaiko` フォルダーおよび `client.realm` をもとに、TaikøNauts の `category.def`・`box.def`・`order.def` を自動生成し、複数の切り口（タイトル順、難易度順、ステータス別、コレクション、カスタム）で快適な選曲環境を構築する独立ツールを提供する。

#### 1.2 起動・設定管理仕様（GUI / CLI ハイブリッド）

* **GUIモード（エクスプローラーから直接起動時）**：
  * 全体設定、カテゴリーごとの詳細設定（`order.def` / `box.def`）、およびカスタムフォルダーの追加・編集を行える画面を表示する。
  * 「保存して実行」または「設定保存」により、同階層の `config.json` に設定を書き出す。
  * 「無効のカテゴリーを消去」が ON の状態で生成を開始した場合、**削除対象となるカテゴリー名の一覧を表示する確認ポップアップ**を表示し、ユーザーの承認後に処理を実行する。

* **CLIモード（ターミナル・バッチファイルから起動時）**：
  * 同階層の `config.json` を読み込み、設定通りに生成・更新処理を自動実行する。
  * `config.json` が存在しない場合、未設定状態のテンプレート `config.json` を出力した上でエラーメッセージを表示し、終了コード `1` で終了する。
  * 「無効のカテゴリーを消去」が ON の場合は、標準出力へ削除対象カテゴリー名をログ出力しユーザーに処理を続行しても大丈夫か尋ねる。実行時引数に `--delete-invalid-category` が指定された場合はログ出力のみ行い、ユーザー確認無しに処理を続行する。

---

### 2. 外部設計（ディレクトリ構成・UI仕様・定義ファイル仕様）

#### 2.1 入出力パスとディレクトリ構成イメージ

参照元である `osu_taiko_path`（シンボリックリンク群と `manifest.json` が存在するフォルダー）とは独立して、出力先ルートディレクトリ **`output_path`** を指定できる設計とする。

* `output_path` が空欄の場合は、`osu_taiko_path` の親フォルダーを出力先として自動採用する。
* 「タイトル順に `osuTaiko` フォルダーを使う」トグルが **ON** の場合：
  * タイトル順カテゴリーのみ `osu_taiko_path` 内に直接生成し、それ以外のカテゴリーは `output_path` 配下に生成する。
* 「タイトル順に `osuTaiko` フォルダーを使う」トグルが **OFF** の場合：
  * タイトル順を含むすべてのカテゴリーを `output_path` 配下に生成する（`osu_taiko_path` 内には一切書き込まない）。
* 実体ファイルやシンボリックリンクは複製せず、すべて各定義ファイルから `osu_taiko_path` への相対パス参照（`Path.GetRelativePath` 等）によって構成する。

```text
[参照元: osu_taiko_path] (例: D:\TaikoNauts\Songs\osuTaiko)
 ├── manifest.json                            # osuLazer2TaikoNauts が生成したキャッシュ
 ├── (category.def / order.def)               # ※「タイトル順にosuTaikoフォルダーを使う」ON時
 ├── (Custom_01/ ...)                         # ※ 同上（タイトル順に追加されたカスタムフォルダー）
 └── TitleA_ArtistA_UUID/
      ├── (box.def / order.def)               # ※ 同上
      └── (各シンボリックリンク: .osu, 音源, 画像)

[出力先: output_path] (空欄時は osu_taiko_path の親フォルダー)
 ├── osuTaiko_Title/                          # [タイトル順カテゴリー (トグル OFF 時)]
 │    ├── category.def
 │    ├── order.def                           # [BoxInfo] リスト
 │    ├── Custom_01/                          # カスタムフォルダー（0個以上任意）
 │    └── TitleA_ArtistA_UUID/
 │         ├── box.def
 │         └── order.def                      # 該当曲の .osu への [SongInfo] リスト
 │
 ├── osuTaiko_Difficulty/                     # [難易度順カテゴリー]
 │    ├── category.def
 │    ├── order.def
 │    ├── Custom_01/                          # カスタムフォルダー （適正難易度など）
 │    │    ├── box.def
 │    │    └── order.def                      # 該当譜面の [SongInfo] リスト
 │    ├── Star_05/                            # ★5.00～★5.99 ボックス
 │    │    ├── box.def
 │    │    └── order.def
 │    └── ...
 │
 ├── osuTaiko_Status/                         # [ステータス別カテゴリー]
 │    ├── category.def
 │    ├── order.def
 │    ├── Custom_01/                          # カスタムフォルダー（任意個）
 │    ├── Status_Ranked/
 │    │    ├── box.def
 │    │    └── order.def
 │    └── ...
 │
 ├── osuTaiko_Collection/                     # [コレクションカテゴリー]
 │    ├── category.def
 │    ├── order.def
 │    ├── Custom_01/                          # カスタムフォルダー（任意個）
 │    └── Col_01/
 │         ├── box.def
 │         └── order.def
 │
 └── osuTaiko_Custom/                         # [カスタムカテゴリー（カスタムフォルダーのみ）]
      ├── category.def
      ├── order.def                           # 設定した順の [BoxInfo] リスト
      └── Custom_01/
           ├── box.def
           └── order.def
```

---

#### 2.2 GUI・全体設定およびカテゴリー構成仕様

##### ① 全体設定（`*` は必須項目）

| 項目名 | UIコンポーネント | 仕様・制約 |
| :--- | :--- | :--- |
| **Realm** `*` | `[入力欄]`（＋参照ボタン） | osu!lazer の `client.realm` パス。初期値：`%APPDATA%\osu\client.realm` |
| **osuTaiko** `*` | `[入力欄]`（＋参照ボタン） | `osuLazer2TaikoNauts` が生成した `osuTaiko` フォルダーのパス。 |
| **タイトル順にosuTaikoフォルダーを使う** | `[トグル]` | `osuTaiko` 入力欄の横に配置。ON でタイトル順のみ `osuTaiko` 内に直接生成、OFF ですべて `output` に生成。 |
| **output** | `[入力欄]`（＋参照ボタン） | 出力先ルートディレクトリ。空欄の場合は `osuTaiko` の親フォルダーを自動採用。 |
| **適正難易度** | `[入力欄]` | `0.00` 以上の数値。空欄時は **`5.2`** を採用。<br>※ 1つのビートマップセットに複数音源・複数背景画像が含まれる際、`StarRating` がこの値に最も近い譜面の素材を代表として採用する。 |
| **無効のカテゴリーを消去** | `[トグル]` | ON の場合、生成開始時に削除されるカテゴリー名一覧の確認ポップアップを表示し、承認後に無効カテゴリーの出力フォルダーを安全に削除する。 |

##### ② カテゴリーの設定（アコーディオン展開式）

以下の5カテゴリーについて、各項目名の横に **ON/OFF のトグル** を設ける。トグルが ON のときのみ、各カテゴリーの詳細設定（カテゴリー名、`order.def`、`box.def`、カスタムフォルダー一覧）が展開表示される。

1. **タイトル順**
2. **難易度順**
3. **ステータス別**
4. **コレクション**
5. **カスタム**（ユーザーが定義したカスタムフォルダーのみを収録するカテゴリー）

※ **カスタムフォルダー** は、上記 **すべてのカテゴリーにおいて 0 個以上の任意の数だけ追加・並べ替え・削除** できるようにする。

---

#### 2.3 `order.def` の並び順設定仕様

各カテゴリーで選択可能な `order.def` のソート仕様は以下の通りとする。
（※自動生成ボックスを設定した順に配置し、カスタムフォルダーを設定した順で末尾に配置する。）

| カテゴリー | box の並び順（カテゴリー直下 `order.def`） | box 内の並び順（各ボックス内 `order.def`） | 備考 |
| :--- | :--- | :--- | :--- |
| **タイトル順** | プルダウン：<br>・五十音昇順 / 五十音降順<br>・ローマ字昇順 / ローマ字降順 | プルダウン：<br>・難易度昇順 / 難易度降順 | 五十音順は記号→英数字→あ〜ん。ローマ字順は記号→数字→A-Z。<br>五十音は`TitleUnicode`を、ローマ字は`Title`を参照する。 |
| **難易度順** | プルダウン：<br>・難易度昇順 / 難易度降順 | プルダウン：<br>・難易度昇順 / 難易度降順 | box 内外問わず難易度（`StarRating`）順でソート。 |
| **ステータス別** | 固定順：<br>`Ranked` → `Qualified` → `Loved` → `Pending` → `WIP` → `Graveyard` → `Unknown` | プルダウン：<br>・五十音昇順 / 五十音降順<br>・ローマ字昇順 / ローマ字降順<br>・難易度昇順 / 難易度降順 |  |
| **コレクション** | プルダウン：<br>・五十音昇順 / 五十音降順<br>・ローマ字昇順 / ローマ字降順<br>・osu!コレクション順 | プルダウン：<br>・五十音昇順 / 五十音降順<br>・ローマ字昇順 / ローマ字降順<br>・難易度昇順 / 難易度降順<br>・osu!コレクション順 | osu! 側にコレクションが1つも存在せず、カスタムフォルダーも未設定の場合はカテゴリーを出力しない。 |
| **カスタム** | 固定（設定した順） | カスタムフォルダーごとにプルダウン：<br>・五十音昇順 / 五十音降順<br>・ローマ字昇順 / ローマ字降順<br>・難易度昇順 / 難易度降順 | カスタムフォルダーのみを含む。<br>カスタムフォルダーが未設定の場合はカテゴリーを出力しない。 |

##### `order.def` 出力フォーマット例

* **カテゴリー直下（box を並べる場合）**：
```ini
[BoxInfo]
TITLE:愛言葉IV

[BoxInfo]
TITLE:アンドロイドガール feat. 初音ミク
```
```ini
[BoxInfo]
TITLE:★5.00～★5.99

[BoxInfo]
TITLE:★6.00～★6.99
```
```ini
[BoxInfo]
TITLE:Ranked

[BoxInfo]
TITLE:Qualified
```

* **各ボックス内（個別譜面を並べる場合）**：
```ini
[SongInfo]
PATH:../../osuTaiko/TitleA_ArtistA_UUID/MIMI - Ai no Sukima (Cut Ver.) (Eriha) [Oni].osu

[SongInfo]
PATH:../../osuTaiko/TitleB_ArtistB_UUID/TUYU - Anoyo-iki no Bus ni Notte Saraba. (gaston_2199) [Oni].osu
```
※ `order.def` の配置場所から `osu_taiko_path` 内の該当 `.osu` ファイルへの相対パスを出力する。

---

#### 2.4 `box.def` の全12項目詳細仕様（カテゴリー別設定）

TaikøNauts（β2025.1214.0 以降）の `box.def` 全12項目について、各カテゴリー（タイトル順・難易度順・ステータス別・コレクション）での設定UIと出力仕様を以下に定義する。
※`category.def` はスタイル設定を持たないため `#TITLE:<カテゴリー名>` のみを出力する。

| 項目名 | UI・設定仕様 | 出力ルール・補足 |
| :--- | :--- | :--- |
| **`#TITLE`** | ・**タイトル順**：プルダウン（`TitleUnicode優先` / `Title (ローマ字)`）<br>・**その他カテゴリー**：設定不要（自動採用） | ・タイトル順：`TitleUnicode優先` の場合、`TitleUnicode` が空なら `Title` にフォールバック。もしも`Title`が空欄なら`TitleUnicode`を使用。<br>・難易度順：`★〇.00～★〇.99`（例：`★5.00～★5.99`）<br>・ステータス別：`Ranked`, `Qualified`, `Loved`, `Pending`, `WIP`, `Graveyard`, `Unknown`<br>・コレクション：コレクション名 |
| **`#GENRE`** | プルダウン：<br>・**タイトル順**：`ソース` / `タイトル` / `カスタム` / `なし`<br>・**その他**：`タイトル` / `カスタム` / `なし`<br>※`カスタム` 選択時のみプルダウン下に入力欄を表示。<br>※`ソース` 選択時のみソースが空の場合に`#TITLE`を代わりに使うトグルを表示。 | ・`ソース`：同一 box 内に複数ソースが混在する場合は「`、`」区切りで出力（すべて空なら `osu!taiko`）。<br>・`タイトル`：その box の `#TITLE` と同じ文字列。<br>・`なし`：項目行を非出力。 |
| **`#EXPLANATION`** | ・**タイトル順**：以下5項目の個別 ON/OFF トグル<br>　1. アーティスト（`ArtistUnicode優先` チェックボックス付き）<br>　2. 譜面製作者（`BeatmapMetadata.Author.Username`）<br>　3. ソース<br>　4. ステータス<br>　5. 難易度（トグル横に「1行あたりの最大文字数」入力欄。初期値 `40`）<br>・**その他**：設定不要（自動採用） | ・タイトル順：有効な項目のみ `\n` 区切りで出力。すべて OFF の場合は項目行を非出力。ソースが存在しない場合に空行にならないようにする。難易度は `★2.10 ★3.45 ★4.80` のように昇順で並べ、最大文字数を目安に均等割り改行するが`★○.○○`の途中で改行されないようにする。<br>・難易度順：`★〇.00～★△.99の曲をあつめたよ！`<br>・ステータス別：`osu!taikoの{status}の曲をあつめたよ！`<br>・コレクション：`osu!taikoの{コレクション名}の曲をあつめたよ！` |
| **`#BOXCOLOR`** | プルダウン：<br>・**タイトル順**：`ステータス` / `カスタム` / `なし`<br>・**難易度順**：`難易度の色` / `カスタム` / `なし`<br>・**ステータス別**：`ステータス` / `カスタム` / `なし`<br>・**コレクション**：`カスタム` / `なし`<br>※`カスタム` 選択時のみプルダウン下に入力欄を表示（初期値例：`#ff87c6`）。 | ・`box.def` は単色指定（グラデーション非対応）のため、難易度順では該当難易度帯の代表単色（2.5 参照）を出力する。<br>・`なし`：項目行を非出力。 |
| **`#FORECOLOR`** | プルダウン：`自動` / `カスタム` / `なし`<br>※`カスタム` 選択時のみプルダウン下に入力欄を表示。<br>難易度順のみ`osu!`の選択肢も用意。 | ・`自動`：`#BOXCOLOR` を HSL 変換した色相 $H$ を維持し、彩度 $S = 100\%$、明度 $L = 97\%$ としたカラーコードを出力。（`#BOXCOLOR` が `なし` の場合は `#FFFFFF`）<br>・`osu!`：下記「2.5 B. 難易度（Star Rating）カラー（単色算出ルール）」の「`#FORECOLOR` の値について」表の `#FORECOLOR` 値を参照。<br>・`なし`：項目行を非出力。 |
| **`#BACKCOLOR`** | プルダウン：`自動` / `カスタム` / `なし`<br>※`カスタム` 選択時のみプルダウン下に入力欄を表示。 | ・`自動`：`#BOXCOLOR` を HSL 変換した $H, S$ を維持し、明度 $L \ge 20\%$ なら $L - 20\%$、$L < 20\%$ なら $L + 20\%$ としたカラーコードを出力。（`#BOXCOLOR` が `なし` の場合は `#620616`）<br>・`なし`：項目行を非出力。 |
| **`#USEBARDECORATION`** | トグルのみ（ON / OFF） | ON なら `true`、OFF なら `false` を出力。 |
| **`#BOXFOCUSSOUND`** | プルダウン：<br>・**タイトル順**：`Beatmap` / `絶対パス` / `なし`<br>・**その他**：`絶対パス` / `なし`<br>※`絶対パス` 選択時のみプルダウン下に入力欄を表示。 | ・`Beatmap`：曲の音源への相対パスを出力。セット内に複数音源がある場合は、`StarRating` が全体設定の「適正難易度」に最も近い譜面の音源を採用。<br>・`絶対パス`：入力された絶対パスから出力先 `box.def` 基準の相対パスへ自動変換（相対パス入力は非対応）。<br>・`なし`：項目行を非出力。 |
| **`#BGIMAGE`** | プルダウン：<br>・**タイトル順**：`Beatmap` / `絶対パス` / `なし`<br>・**その他**：`絶対パス` / `なし`<br>※`絶対パス` 選択時のみプルダウン下に入力欄を表示。 | ・`Beatmap`：曲の背景画像への相対パスを出力。複数背景画像がある場合は、`StarRating` が全体設定の「適正難易度」に最も近い譜面の画像を採用。<br>・`絶対パス`：絶対パスから相対パスへ自動変換。<br>・`なし`：項目行を非出力。 |
| **`#BARIMAGE`** | 同上（タイトル順は `Beatmap` / `絶対パス` / `なし`、その他は `絶対パス` / `なし`） | 同上。 |
| **`#BOXPOPIMAGE`** | 同上（タイトル順は `Beatmap` / `絶対パス` / `なし`、その他は `絶対パス` / `なし`） | 同上。 |
| **`#DIFFBGIMAGE`** | 同上（タイトル順は `Beatmap` / `絶対パス` / `なし`、その他は `絶対パス` / `なし`） | 同上。 |

---

#### 2.5 カラーパレット定義（`#BOXCOLOR`）

##### A. ステータス カラー

| Status値 | 名称 | `#BOXCOLOR` | 色イメージ |
| :--- | :--- | :--- | :--- |
| `1` | `Ranked` | `#B0FC65` | 黄緑 |
| `3` | `Qualified` | `#64C9FB` | 水色 |
| `4` | `Loved` | `#FC65A9` | ピンク |
| `0` | `Pending` | `#FCD665` | 黄色 |
| `-1` | `WIP` | `#F69462` | オレンジ |
| `-2` | `Graveyard` | `#59666D` | グレー |
| `2` / その他 | `Approved` / `Unknown` | `#FFFFFF` | 白 |

##### B. 難易度（Star Rating）カラー（単色算出ルール）

任意の `StarRating` 値 $s$ に対応する単色 HEX コードを gamma(2.2)補間(`d3.interpolateRgb.gamma(2.2)` 相当) で算出する（`box.def` 自体は単色出力）。
* **難易度順カテゴリーの各帯ボックス**：各帯の基準値（例：★5帯 `5.00～5.99` なら `5.00`）に対応する単色を採用。
* **カスタムフォルダー（「難易度の平均値の色」選択時）**：最低難易度と最高難易度の平均値（例：`4.80` と `5.20` なら `5.00`）に対応する単色をgamma(2.2)補間で算出する。

| 基準 Star Rating | `#BOXCOLOR` |
| :--- | :--- |
| `0.00`以上 `0.10`未満 | `#AAAAAA` |
| `0.10`以上 `1.25`未満 | `#4290FB` ～ `#4FC0FF` |
| `1.25`以上 `2.00`未満 | `#4FC0FF` ～ `#4FFFD5` |
| `2.00`以上 `2.50`未満 | `#4FFFD5` ～ `#7CFF4F` |
| `2.50`以上 `3.30`未満 | `#7CFF4F` ～ `#F6F05C` |
| `3.30`以上 `4.20`未満 | `#F6F05C` ～ `#FF8068` |
| `4.20`以上 `4.90`未満 | `#FF8068` ～ `#FF4E6F` |
| `4.90`以上 `5.80`未満 | `#FF4E6F` ～ `#C645B8` |
| `5.80`以上 `6.70`未満 | `#C645B8` ～ `#6563DE` |
| `6.70`以上 `7.70`未満 | `#6563DE` ～ `#18158E` |
| `7.70`以上 `9.00`未満 | `#18158E` ～ `#000000` |
| `9.00`以上 | `#000000` |


* `#FORECOLOR` の値について (9.00未満はそれぞれの固定値、9.00以上12.40未満ではgamma(2.2) 補間)

| 基準 Star Rating| `#FORECOLOR` |
| :--- | :--- |
| `0.00`以上 `6.50`未満 | `#000000` |
| `6.50`以上 `9.00`未満 | `#F6F05C` |
| `9.00`以上 `9.90`未満 | `#F6F05C` ～ `#FF8068` |
| `9.90`以上 `10.60`未満 | `#FF8068` ～ `#FF4E6F` |
| `10.60`以上 `11.50`未満 | `#FF4E6F` ～ `#C645B8` |
| `11.50`以上 `12.40`未満 | `#C645B8` ～ `#6563DE` |
| `12.40`以上 | `#6563DE` |



---

#### 2.6 カスタムフォルダーの設定および検索クエリ仕様

すべてのカテゴリーに対して 0 個以上の任意の数だけ追加できる「カスタムフォルダー」の設定項目を以下に定義する。

##### ① 基本・フィルタリング設定

* **`#TITLE`**：入力欄。入力された文字列をそのまま出力する。
* **最低難易度 / 最高難易度**：数値入力欄。最低は `0.00` 以上または空欄、最高は 最低難易度以上の任意の実数 または空欄。
* **ワード入力欄（検索クエリ）**：
  * **フリーワード検索**：タイトル（`Title`, `TitleUnicode`）、アーティスト（`Artist`, `ArtistUnicode`）、ソース（`Source`）、タグ（`Tags`）、製作者（`Author.Username`）、難易度名（`DifficultyName`）のいずれかに含まれる譜面を抽出する（大文字小文字は区別しない）。
  * **条件指定検索**：`属性名 + 演算子 + 値` の形式で数値・文字列条件を指定可能とする。
    * 対応演算子：`=`, `!=`, `>`, `>=`, `<`, `<=`
    * 値は引用符あり（`BPM>="200"`）・なし（`BPM>=200`）の両方に対応する。
    * 属性名は **`client.realm` のプロパティ名そのまま**（例：`OverallDifficulty`, `StarRating`, `BPM`, `Length`, `Status`）と、**[osu!公式検索用語](https://osu.ppy.sh/wiki/en/Beatmap_search#client-(lazer))**（例：`od`, `star`/`stars`, `bpm`, `length`, `artist`, `title`, `creator`/`mapper`, `source`, `tag`, `status`）の両方に反応させる。
  * **区切り・フレーズ検索**：スペース区切りで複数条件を指定する。スペースを含む語句はダブルクォーテーション `""` で囲むことで連続文字列として検索する。また、ダブルクォーテーション内では条件指定検索を無効とする。
  * **論理演算（AND / OR の優先順位）**：
    * 基本はすべての条件を **AND** 結合とする。
    * 大文字 `OR` で挟まれた隣接条件は **OR** グループとして先に結合する（`OR` が `AND` より高い結合優先度を持つ）。
    * **解釈例**：
      `太鼓の達人 OR "Taiko no Tatsujin" かめりあ OR Camellia BPM>="200" StarRating>="4.80" StarRating<"5.30"`
      $\Rightarrow$ `(太鼓の達人 OR "Taiko no Tatsujin") AND (かめりあ OR Camellia) AND (BPM >= 200) AND (StarRating >= 4.80) AND (StarRating < 5.30)`
  * **難易度入力欄との競合ルール**：
    * 「最低難易度 / 最高難易度」入力欄と「ワード入力欄」の両方に難易度条件（`StarRating` / `stars` 等）が指定された場合、**ワード入力欄の指定を優先**する。

##### ② `box.def` スタイル設定（カスタムフォルダー）

* **`#GENRE`**：入力欄。入力された文字列をそのまま出力（空欄なら非出力）。
* **`#EXPLANATION`**：入力欄。入力された文字列をそのまま出力（空欄なら非出力）。
* **`#BOXCOLOR`**：プルダウン（`難易度の平均値の色` / `カスタム` / `なし`）。
  * `難易度の平均値の色` は **最低難易度と最高難易度の両方が入力されている場合のみ選択可能** とし、2値の平均値（例：`4.80` と `5.20` なら `5.00`）に対応する難易度カラーを出力する。<br>※フィルタリング条件がワード入力欄の難易度指定で上書きされた場合でも、色の算出にはUIの『最低難易度』と『最高難易度』の平均値を使用する。
* **`#FORECOLOR` / `#BACKCOLOR` / `#USEBARDECORATION`**：2.4 の `box.def` 設定に準ずる。
  * `#BOXCOLOR`で`難易度の平均値の色`が選択されている場合の`#FORECOLOR`においては、2.4 の `#FORECOLOR` での難易度順と同様に `osu!` に対応する。
* **`#BOXFOCUSSOUND` / `#BGIMAGE` / `#BARIMAGE` / `#BOXPOPIMAGE` / `#DIFFBGIMAGE`**：プルダウン（`絶対パス` / `なし`）。`Beatmap` には非対応。`絶対パス` 選択時のみ入力欄を表示する。

---

### 3. 設定ファイル（`config.json`）の構造設計


```json
{
  "paths": {
    "realm_path": "C:\\Users\\<User>\\AppData\\Roaming\\osu\\client.realm",
    "osu_taiko_path": "D:\\TaikoNauts\\Songs\\osuTaiko",
    "output_path": ""
  },
  "general": {
    "use_osu_taiko_folder_for_title": true,
    "target_star_rating": 5.2,
    "delete_disabled_categories": false
  },
  "categories": {
    "title": {
      "enabled": true,
      "category_name": "osu!(タイトル順)",
      "order": {
        "box_sort": "gojuon_asc",
        "song_sort": "difficulty_asc"
      },
      "box_style": {
        "title_mode": "unicode_prefer",
        "genre_mode": "source",
        "genre_custom": "",
        "explanation": {
          "show_artist": true,
          "artist_unicode_prefer": true,
          "show_creator": true,
          "show_source": true,
          "show_status": true,
          "show_star_rating": true,
          "star_rating_max_chars": 40
        },
        "box_color_mode": "ranked_status",
        "box_color_custom": "#ff87c6",
        "fore_color_mode": "auto",
        "fore_color_custom": "#FFFFFF",
        "back_color_mode": "auto",
        "back_color_custom": "#000000",
        "use_bar_decoration": false,
        "box_focus_sound": { "mode": "none", "path": "" },
        "bg_image": { "mode": "none", "path": "" },
        "bar_image": { "mode": "none", "path": "" },
        "box_pop_image": { "mode": "none", "path": "" },
        "diff_bg_image": { "mode": "none", "path": "" }
      },
      "custom_folders": []
    },
    "difficulty": {
      "enabled": true,
      "category_name": "osu!(難易度順)",
      "order": {
        "box_sort": "difficulty_asc",
        "song_sort": "difficulty_asc"
      },
      "box_style": {
        "genre_mode": "none",
        "genre_custom": "",
        "box_color_mode": "difficulty_color",
        "box_color_custom": "#ff87c6",
        "fore_color_mode": "osu",
        "fore_color_custom": "#FFFFFF",
        "back_color_mode": "auto",
        "back_color_custom": "#000000",
        "use_bar_decoration": false,
        "box_focus_sound": { "mode": "none", "path": "" },
        "bg_image": { "mode": "none", "path": "" },
        "bar_image": { "mode": "none", "path": "" },
        "box_pop_image": { "mode": "none", "path": "" },
        "diff_bg_image": { "mode": "none", "path": "" }
      },
      "custom_folders": [
        {
          "title": "適正難易度 (★4.80～5.20)",
          "min_star": 4.80,
          "max_star": 5.20,
          "search_query": "",
          "song_sort": "difficulty_asc",
          "genre": "osu!taiko",
          "explanation": "適正難易度帯の譜面を集めたよ！",
          "box_color_mode": "average_difficulty",
          "box_color_custom": "#ff87c6",
          "fore_color_mode": "auto",
          "fore_color_custom": "#FFFFFF",
          "back_color_mode": "auto",
          "back_color_custom": "#000000",
          "use_bar_decoration": false,
          "box_focus_sound": { "mode": "none", "path": "" },
          "bg_image": { "mode": "none", "path": "" },
          "bar_image": { "mode": "none", "path": "" },
          "box_pop_image": { "mode": "none", "path": "" },
          "diff_bg_image": { "mode": "none", "path": "" }
        }
      ]
    },
    "status": {
      "enabled": false,
      "category_name": "osu!(ステータス別)",
      "order": {
        "song_sort": "gojuon_asc"
      },
      "box_style": {
        "genre_mode": "none",
        "genre_custom": "",
        "box_color_mode": "status",
        "box_color_custom": "#ff87c6",
        "fore_color_mode": "auto",
        "fore_color_custom": "#FFFFFF",
        "back_color_mode": "auto",
        "back_color_custom": "#000000",
        "use_bar_decoration": false,
        "box_focus_sound": { "mode": "none", "path": "" },
        "bg_image": { "mode": "none", "path": "" },
        "bar_image": { "mode": "none", "path": "" },
        "box_pop_image": { "mode": "none", "path": "" },
        "diff_bg_image": { "mode": "none", "path": "" }
      },
      "custom_folders": []
    },
    "collection": {
      "enabled": true,
      "category_name": "osu!(コレクション)",
      "order": {
        "box_sort": "collection_order",
        "song_sort": "collection_order"
      },
      "box_style": {
        "genre_mode": "none",
        "genre_custom": "",
        "box_color_mode": "custom",
        "box_color_custom": "#ff87c6",
        "fore_color_mode": "auto",
        "fore_color_custom": "#FFFFFF",
        "back_color_mode": "auto",
        "back_color_custom": "#000000",
        "use_bar_decoration": false,
        "box_focus_sound": { "mode": "none", "path": "" },
        "bg_image": { "mode": "none", "path": "" },
        "bar_image": { "mode": "none", "path": "" },
        "box_pop_image": { "mode": "none", "path": "" },
        "diff_bg_image": { "mode": "none", "path": "" }
      },
      "custom_folders": []
    },
    "custom": {
      "enabled": false,
      "category_name": "osu!(カスタム)",
      "custom_folders": []
    }
  }
}
```

#### ソートキー定義（`box_sort` / `song_sort`）
* `gojuon_asc` / `gojuon_desc`（五十音昇順・降順）
* `romaji_asc` / `romaji_desc`（ローマ字昇順・降順）
* `difficulty_asc` / `difficulty_desc`（難易度昇順・降順）
* `collection_order`（osu!コレクション順）

#### 素材参照モード定義（`box_focus_sound` / `bg_image` / `bar_image` / `box_pop_image` / `diff_bg_image`）
* `mode`: `"none"` / `"beatmap"`（タイトル順のみ） / `"absolute_path"`
* `path`: `mode == "absolute_path"` のときのみ参照する絶対パス文字列

---

### 4. 内部設計（実装指針）

実装の細部は開発担当（または実装AI）に一任するが、以下の責務分離を推奨する。

1. **`ConfigManager`**：`config.json` の読み書き、空値フォールバック（`output_path` の親フォルダー自動解決、`target_star_rating = 5.2`）、CLI 起動時の未設定テンプレート出力とエラー終了。
2. **`RealmMetadataReader`**：`osu_taiko_path/manifest.json` と `client.realm`（一時コピー・読み取り専用）を照合し、譜面メタデータ・`StarRating`・`BPM`・`OverallDifficulty`・`Length`・`BeatmapCollection` 順序を抽出する。
3. **`QueryFilterEngine`**：カスタムフォルダー用の検索クエリパーサー（`""` フレーズ、`OR` 優先グルーピング、AND 結合、Realm プロパティ名および osu!公式検索エイリアス、比較演算子、難易度入力欄優先ルール）を実装する。
4. **`ColorAndTextFormatter`**：HSL 変換による `#FORECOLOR`（S=100%, L=97%）・`#BACKCOLOR`（L ± 20\%$）の自動計算、osu! 難易度スペクトルからの単色算出、`#EXPLANATION` の均等割り改行を行う。
5. **`SortEngine`**：五十音ソート、アルファベットソート、難易度ソート、コレクション順ソートを行う。
6. **`DefGenerator`**：各カテゴリーの `category.def`・`box.def`・`order.def` を差分生成し、素材・譜面パスをすべて相対パスへ変換する。無効カテゴリー削除時の対象リスト抽出と安全なクリーンアップを行う。

---

### 5. 参考

* [TaikøNauts公式サイト 譜面管理システム解説](https://taikonauts-docs.pages.dev/docs/SongSystem/CategoryAndOrder )
* [osu!公式 Wiki - Beatmap search (lazer)](https://osu.ppy.sh/wiki/en/Beatmap_search#client-(lazer) )
* [osu!公式 Wiki - 難易度カラー](https://osu.ppy.sh/wiki/en/Beatmap/Difficulty )
* [osuLazer2TaikoNauts](https://github.com/totoka3710/osuLazer2TaikoNauts )
  * [Antigravity版](https://github.com/totoka3710/osuLazer2TaikoNauts/blob/main/BL_Antigravity/README.md )
  * [Claude版](https://github.com/totoka3710/osuLazer2TaikoNauts/blob/main/BL_Claude/README.md )
  * [Codex版](https://github.com/totoka3710/osuLazer2TaikoNauts/blob/main/BL_Codex/README.md )