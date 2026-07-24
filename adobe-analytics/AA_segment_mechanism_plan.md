# AA セグメント仕組み理解ドキュメント：設計書

> このファイルは別セッションで執筆を再開するための計画書。
> 最終成果物は `AA_segment_mechanism.md`（Confluence markup）として出力する。

---

## プロジェクト目標

Adobe Analytics のセグメントを「データ抽出装置」として理解するための包括ドキュメントを作る。
読者が「このセグメントは何を返すか」を自分で逆算できるようになることがゴール。

対象読者：AA の基本操作は知っているが、除外や順次の挙動で意図しない結果に直面したことがある実務者。

---

## 既存リソース

| ファイル | 内容 | 本ドキュメントとの関係 |
|---|---|---|
| `AA_sequential_segment_exclude.md` | 順次セグメントの EXCLUDE 位置別挙動（Confluence markup） | 7 節の EXCLUDE サブ節で参照・統合 |
| `順次セグメント：EXCLUDEの位置による挙動… - Noriaki Inoue - Wiki.pdf` | 上記の Wiki 公開版 PDF | 同上 |

---

## 合意した章構成と各節の執筆指針

### 1節　AA のデータモデル：セグメントが操作する対象

**ゴール：** 「セグメントとは識別子の集合を選ぶ操作である」という定義を確立する。

**カバーする内容：**
- hit・visit・visitor の親子階層
- 各レコードが持つ識別子（hit ID / visit ID / visitor ID）
- 同一 visit に属する hit は visit ID を共有する、という事実
- 「取得」= 「特定の識別子を持つ行を result set に含める」という操作モデル

**執筆ポイント：**
- 図（テキスト表で代替可）で階層を示す：visitor > visit > hit
- この節が後続節の基盤になるので、「識別子の共有」という概念を明確に

---

### 2節　コンテナの本質：評価スコープと返却スコープの一体性

**ゴール：** コンテナが「何を判定するか」と「何を返すか」を同時に定義するという本質を理解させる。

**カバーする内容：**
- コンテナの 2 つの役割を定義：
  - **評価スコープ**：条件をどの粒度で判定するか
  - **返却スコープ**：条件が通った時、何を返すか
- 評価スコープ ≡ 返却スコープ（常に同じ）
- 3 コンテナの動作をこのモデルで説明：

| コンテナ | 評価スコープ | 返却スコープ |
|---|---|---|
| ヒット | 個々の hit | 条件に一致した hit のみ |
| 訪問 | 訪問内の全 hit 群 | 条件が通った訪問の全 hit |
| 訪問者 | 訪問者の全 hit 群 | 条件が通った訪問者の全 hit |

**執筆ポイント：**
- 訪問コンテナで「条件に一致しない hit も返る」理由は、返却スコープが visit 全体だから、と明示
- 具体例：`[Visit: page=product]` を適用した時、cart hit や checkout hit も返ってくる理由

---

### 3節　複合条件の評価メカニズム：AND/OR とスコープの関係

**ゴール：** 複数条件の AND/OR が「どのスコープで判定されるか」を正確に理解させる。

**カバーする内容：**

**3-1. 単一コンテナ内の AND/OR**
- AND：同じスコープの同じ粒度で両方 true であること
- 例：`[Hit: page=home AND page=product]` → 1 hit に両方は入らない → 必ず 0 件
- 例：`[Visit: page=home AND page=product]` → 同じ visit 内のどこかに両方の hit が存在すれば OK

**3-2. ネストしたコンテナの AND**
- 子コンテナは「その条件が成立するスコープが存在するか」を真偽値として親に返す
- 例：`[Visit: [Hit: page=home] AND [Hit: page=product]]` と `[Visit: page=home AND page=product]` は等価
- 例：`[Visitor: [Visit: page=checkout]]` = 過去に checkout visit を持つ訪問者の全データ

**3-3. 異なるスコープを AND で組み合わせた場合の評価フロー**
- 外側コンテナが全体の返却スコープを決める
- 内側条件は「そのスコープ内で条件が成立するか」の真偽値を提供するだけ

**執筆ポイント：**
- `[Visitor: [Visit: page=cart] AND [Visit: page=checkout]]` の例で「異なる visit にまたがっても OK」を示す
- これが後の「スコープミスマッチ」理解の前提になる

---

### 4節　除外コンテナのメカニズム

**ゴール：** 除外が「物理的な行削除」ではなく「スコープ単位の否定条件」であることを理解させる。除外コンテナのスコープが結果を決定的に変えることを理解させる。

**カバーする内容：**

**4-1. 除外コンテナの本質**
- `[Exclude Visit: page=plp]` = 「page=plp を含む visit を結果から除く」
- 「除外」は行を削除するのではなく、スコープ単位で false を返す条件
- 親コンテナが visit なら、子の Exclude Visit は visit 単位で否定

**4-2. スコープ別の除外コンテナの意味**

| 除外コンテナ | 否定の単位 | 意味 |
|---|---|---|
| Exclude Hit | hit | この hit が条件を満たす → この hit を除く |
| Exclude Visit | visit | この visit が条件を満たす → この visit を除く |
| Exclude Visitor | visitor | この visitor が条件を満たす → この visitor を除く |

**4-3. ネストした時の評価フロー（これが核心）**

パターン A（スコープ一致・正しい）：
```
[Visit:
  page=home
  AND [Exclude Visit: page=plp]
]
```
評価フロー：
1. この visit に page=home の hit が存在するか → true/false
2. この visit に page=plp の hit が存在するか → true の場合、Exclude Visit が visit を否定
3. 両方 true の visit のみを返す
→ 結果：home あり、かつ plp なし の visit ✅

パターン B（スコープミスマッチ・意図しない結果）：
```
[Visit:
  page=home
  AND [Exclude Hit: page=plp]
]
```
評価フロー：
1. Exclude Hit は hit 単位で評価：「page=plp でない hit が存在するか」を visit 内で探す
2. AND の結合は hit レベルで行われる：「page=home かつ page≠plp の hit が存在するか」
3. page=home と page=plp は同一 hit に共存しない → 実質的に `[Visit: page=home]` と等価
→ 結果：plp あり の visit も通過してしまう ❌

**メカニズムの説明：**
Exclude Hit が訪問コンテナの AND に組み込まれると、AND は最内スコープ（hit レベル）で評価される。
hit レベルで「page=home AND page≠plp」は、ページが 1 hit に 1 値しか取れない場合、「page=home」と等価。
除外の效果が失われる。

**4-4. スコープ合わせの原則**
- 「何を除外したいか」のスコープに合わせて Exclude コンテナを選ぶ
- visit 全体を除外したいなら Exclude Visit
- 特定 hit だけ除外したいなら Exclude Hit（ただし親コンテナが visit 以上なら効果に注意）

**4-5. 他のパターン例（よくある誤りと正解）**

| やりたいこと | 誤ったセグメント | 正しいセグメント |
|---|---|---|
| home あり、plp なし の訪問 | `[Visit: home AND [Excl.Hit: plp]]` | `[Visit: home AND [Excl.Visit: plp]]` |
| 購入した訪問者から購入訪問を除いたデータ | `[Visitor: [Excl.Hit: order]]` | `[Visitor: exists AND [Excl.Visit: order]]` |
| 特定 hit を含む visit を除いたすべてのデータ | `[Visit: [Excl.Hit: error]]` | `[Excl.Visit: error]`（トップレベル） |

---

### 5節　順次セグメントのメカニズム

**ゴール：** THEN・After・Within が「hit ストリームのスキャンモデル」として動くことを理解させる。

**カバーする内容：**

**5-1. THEN 演算子：スキャンモデルの基本**
- hit ストリームを時系列に左から右へスキャン
- チェックポイント A に一致する hit を発見 → そこを「通過点」として記録
- 通過点以降の hit ストリームを再スキャンしてチェックポイント B を探す
- B が見つかればシーケンス成立、B が見つからなければ次の A 候補を探して繰り返す
- 「greedy（欲張り）」な動作：成立するなら最大データ量になる組み合わせを選ぶ

具体例（`A THEN B`）：

| ストリーム | 結果 | 理由 |
|---|---|---|
| A → B | 成立 | A の後に B あり |
| B → A | 不成立 | A の後に B がない |
| A → C → B | 成立 | A の後に B あり（C は無視） |
| A → B → A → B | 最初の A→B で成立（greedy） | |

**5-2. After 制約：スキャン開始位置の遅延**
- `A THEN [After X] B` = A の通過点から X の時間/回数/訪問を経過してから B を探し始める
- X 以内に B が出現しても無効（「少なくとも X 経過後でなければ B とみなさない」）
- 単位は time（秒・分・時・日・週）/ hit 数 / visit 数

例：`Page A THEN [After 1 Day] Page B`
- A 閲覧から 24 時間以内の B は対象外
- 24 時間経過後に B を閲覧した訪問者のみ対象

**5-3. Within 制約：スキャン終了位置の設定**
- `A THEN [Within X] B` = A の通過点から X の範囲内に B が存在しなければ不成立
- X を超えても B が現れなかった場合、このシーケンスは不成立
- 「B を探す窓の上限」

例：`Page A THEN [Within 30 Days] Page B`
- A から 30 日以内に B が出現した場合のみ成立

**5-4. After と Within の組み合わせ：スキャン窓の定義**
- `A THEN [After X, Within Y] B` = A の後 X〜Y の窓の中に B が存在すること
- X = 窓の下限、Y = 窓の上限（X < Y）

```
時間軸：
A ---|---X---|--------Y---|---->
          ↑ここから        ↑ここまで
          B を探す窓
```

例：`Visited App [After 1 Day, Within 30 Days] Purchased`
= アプリ訪問の 1 日後〜30 日以内に購入

**5-5. EXCLUDE チェックポイント：順次の中での役割**
- 既存ドキュメント `AA_sequential_segment_exclude.md` の内容を参照・要約して組み込む
- 位置（最初・途中・最後）による役割の違い（qualifying 条件 vs. データ収集終端）
- スキャンモデルで説明：EXCLUDE はスキャンを「止める」か「通過の条件を加える」かが位置で変わる
- After/Within と EXCLUDE の組み合わせ：窓の中での EXCLUDE の扱い

---

### 6節　Only Before / Only After Sequence のメカニズム

**ゴール：** シーケンス条件に加えて「どの範囲のデータを返すか」を制御する仕組みを理解させる。

**カバーする内容：**

**6-1. Include Everyone との違い**
- Include Everyone：シーケンスを「qualifying 条件」として使い、訪問者/訪問の全データを返す
- Only After Sequence：シーケンスの「起点」以降のデータのみを返す
- Only Before Sequence：シーケンスの「終点」以前のデータのみを返す

**6-2. Only After Sequence の起点決定**
- 起点 = 「シーケンス内の最後の INCLUDE チェックポイントに一致した hit」
- EXCLUDE チェックポイントは起点の判定に使われない（webinar 引用で確認済み）
- シーケンスが複数回成立する場合：最初の成立における最後の INCLUDE hit が起点

**6-3. Only Before Sequence の終点決定**
- 終点 = シーケンス内の最初の INCLUDE チェックポイントに一致した hit
- Only After と対称的な構造（時系列を逆方向にスキャンするイメージ）

**6-4. EXCLUDE との組み合わせ**
- Only After + 最後の EXCLUDE = データ収集の終端として機能（5節の EXCLUDE 最後の位置）
- Only Before + 最初の EXCLUDE = データ収集の終端として機能（対称）

**調査で確認済みの公式引用：**
```
"As soon as all those THEN checkpoints that are includes, not excludes, are met,
then it starts to match the data."
— Adobe Customer Success Webinar (Mastering Sequential Logic: Starts and Stops)

"It'll continue until it reaches the end of the person/visit data set, or it hits an exclude."
— 同上
```
URL: https://experienceleague.adobe.com/en/docs/events/adobe-customer-success-webinar-recordings/2025/multisolution2025/sequential-logic-start-stop

---

### 7節　複合設計の読み方

**ゴール：** コンテナ・除外・順次を組み合わせたセグメントを「これは何を返すか」と自分で読み解けるようにする。

**カバーする内容：**

**7-1. 評価の処理順序**
1. 最外コンテナのスコープを確認する（= 返却スコープ）
2. 子コンテナ・条件を内側から外側に向けて評価する
3. 除外コンテナが見つかったら、そのスコープで否定を適用する
4. 順次条件が含まれる場合、スキャンモデルで通過点を特定する
5. Only After/Before が指定されていれば、起点/終点でデータを切り出す

**7-2. 逆算して読む練習例**
- 複合セグメントを示して「何を返すか」を推論するパターンを 3 例程度掲載
- 例：`[Visitor: [Visit: page=product] THEN [Within 7 Days] [Visit: event=purchase] AND [Exclude Visit: page=cart]]`
  - 逆算の手順をステップで示す

**7-3. 設計の検証アプローチ**
- セグメントを「意図した結果」から逆算して設計する手順
- 除外スコープのチェック（「何を除外したいか」→「そのスコープで Exclude を書けているか」）
- 順次条件の greedy 性の確認（「意図しない早期マッチが起きないか」）

---

## 執筆フォーマット仕様

- 出力形式：Confluence wiki markup（`.md` ファイルに書いて Confluence にペースト可能な形式）
- 既存ドキュメント `AA_sequential_segment_exclude.md` と同じ markup を使う
- 各節に `h2.` / `h3.` を使用、コード例は `{{` `}}` で inline、ブロックは `{code}` タグ
- 表は `|| ヘッダ ||` / `| セル |` 形式
- 重要概念は `*太字*` で強調
- 引用は `{quote}` タグ

---

## 未調査・要確認事項

| 項目 | 状況 | 調査方針 |
|---|---|---|
| Exclude Hit inside Visit の正確な評価メカニズム | 推論で説明（直接的公式文書未確認） | ExL 公式ドキュメントの Exclude 節を直接取得して確認 |
| After 制約の単位（hit数・visit数）の正確な動作 | 概念は理解、詳細未確認 | ExL seg-sequential-build の After/Within 節を確認 |
| Within の「シーケンス開始点」基準（A の hit から？A の visit 開始から？） | 未確認 | ExL + webinar で確認 |
| Only Before Sequence の EXCLUDE が「終端」になる動作 | 推論（公式の直接記述未確認） | 要確認 |

---

## セッション間の引き継ぎメモ

- この計画書に沿って `AA_segment_mechanism.md` を作成する（Confluence markup）
- 既存の `AA_sequential_segment_exclude.md` の内容は 5-5 節・6節に統合する
- 未調査事項は執筆前に fj（fluffyjaws）で調査してから書く
- 公式引用には必ず URL を添付する（ExL URL が確認済みのものは上記に記載済み）
