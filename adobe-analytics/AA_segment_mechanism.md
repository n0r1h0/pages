h1. Adobe Analytics セグメント：データ抽出装置として理解する

h2. このドキュメントについて

Adobe Analytics のセグメントを「データ抽出装置」として理解するための包括ガイドです。セグメントで条件を指定した時に「何が起こるか」を仕組みのレベルで解説し、条件の複合・除外・順次それぞれの挙動を自分で逆算できるようになることを目標とします。

対象読者：AA の基本操作は知っているが、除外や順次の挙動で意図しない結果に直面したことがある実務者。

{info:title=AA と CJA の用語対応}
このドキュメントは AA の用語（訪問者・訪問・ヒット）で統一しています。CJA では人物（Person）・セッション（Session）・イベント（Event）に相当します。仕組みは同じです。
{info}

----

h2. 1. AA のデータモデル：セグメントが操作する対象

h3. 1-1. 3 層の階層構造

AA のデータは以下の 3 層で管理される。

{code}
訪問者（Visitor）
  └─ 訪問（Visit）
       └─ ヒット（Hit）
{code}

||レベル||意味||識別子||
|*訪問者*|特定デバイスまたは人物のライフタイム全体|visitor ID|
|*訪問*|一連の連続したセッション（30 分無操作で終了）|visit ID|
|*ヒット*|個々のページビュー・リンククリック・カスタムイベント|hit ID|

h3. 1-2. 識別子の共有関係

*同一訪問に属するヒットは、同じ visit ID を共有する。*
*同一訪問者に属する訪問・ヒットは、同じ visitor ID を共有する。*

||dateTime||page||visit_id||visitor_id||
|1/1 10:00|home|S1|V1|
|1/1 10:05|product|S1|V1|
|1/1 10:12|cart|S1|V1|
|1/3 14:00|checkout|S2|V1|
|1/3 14:08|order|S2|V1|

h3. 1-3. 「取得」とは何か

セグメントの「取得」とは、*特定の識別子を持つ行を結果セットに含める操作* である。セグメントが「一致した」と判定した場合、AA はその識別子（hit ID / visit ID / visitor ID）に等しい行をすべてレポートに返す。


----

h2. 2. コンテナの本質：評価スコープと返却スコープの一体性

h3. 2-1. コンテナの 2 つの役割

コンテナは以下の 2 つの役割を同時に担う。

||役割||定義||
|*評価スコープ*|条件をどの粒度で判定するか|
|*返却スコープ*|条件が通った時、何を返すか|

*評価スコープと返却スコープは常に同じ（≡）である。*

h3. 2-2. 3 コンテナの動作比較

||コンテナ||評価の対象||条件が通った時に返すもの||
|ヒット|個々のヒット 1 件ずつ|条件に一致した *そのヒットのみ*|
|訪問|訪問内の全ヒット群|条件が通った *訪問の全ヒット*|
|訪問者|訪問者の全ヒット群|条件が通った *訪問者の全ヒット*|

h3. 2-3. 具体例：コンテナ別の返却範囲

同じデータに同じ条件 *page=product* を適用し、コンテナを変えると何が返るかを比較する。

データ（○ = 返却される、- = 返却されない）：

||dateTime||page||visit_id||visitor_id||Hit||Visit||Visitor||
|1/1 10:00|home|S1|V1|-|○|○|
|1/1 10:05|*product*|S1|V1|○|○|○|
|1/1 10:12|cart|S1|V1|-|○|○|
|1/3 14:00|checkout|S2|V1|-|-|○|
|1/3 14:08|order|S2|V1|-|-|○|
|1/2 09:00|home|S1|V2|-|○|○|
|1/2 09:04|*product*|S1|V2|○|○|○|
|1/2 09:10|plp|S1|V2|-|○|○|

* *ヒットコンテナ*：条件に一致した 2 ヒットのみ
* *訪問コンテナ*：product ヒットを含む訪問（S1/V1・S1/V2）の全ヒット。V1/S2 は product がないため除外
* *訪問者コンテナ*：product ヒットを含む訪問者の全ヒット。V1/S2 の checkout・order も返ってくる

{warning:title=訪問・訪問者コンテナの 1:N 返却}
条件に一致したヒットだけでなく、そのスコープに属する *すべてのヒット* が返ってくる。
{warning}

----

h2. 3. 複合条件の評価メカニズム：AND/OR とスコープの関係

h3. 3-1. 単一コンテナ内の AND/OR

AND・OR の評価はコンテナのスコープ内で行われる。

*ヒットコンテナ内の AND：*
{code}
[Hit]
    page=home AND page=product
{code}
1 つのヒットに page=home と page=product の両方が存在しなければならない。
1 ヒットに page は 1 値しか持てないため、*結果は常に 0 件*。

*訪問コンテナ内の AND：*
{code}
[Visit]
    page=home AND page=product
{code}
同一訪問の中のどこかに page=home のヒットと page=product のヒットの両方が存在すること。
別々のヒットに分かれていても OK。

○ = 返却される、- = 返却されない：

||dateTime||page||visit_id||visitor_id||[Hit] home AND product||[Visit] home AND product||
|1/1 10:00|home|S1|V1|-|○|
|1/1 10:05|product|S1|V1|-|○|
|1/2 09:00|home|S2|V2|-|-|
|1/2 09:10|contact|S2|V2|-|-|

* Hit コンテナ：1 ヒットに page=home と page=product は共存できないため全 0 件
* Visit コンテナ：S1 は home・product の両ヒットを持つ → S1 全ヒット返却。S2 は product なし → 除外

h3. 3-2. ネストしたコンテナの AND

子コンテナは「その条件が成立するスコープが存在するか」を *真偽値として親に返す*。

{code}
[Visitor]
    [Visit]
        page=cart
    AND
    [Visit]
        page=checkout
{code}

* 「page=cart を含む訪問が存在するか」→ true/false
* 「page=checkout を含む訪問が存在するか」→ true/false
* 両方 true の訪問者の全データを返す
* cart 訪問と checkout 訪問は *異なる訪問でも OK*

○ = 返却される、- = 返却されない：

||dateTime||page||visit_id||visitor_id||返却||
|1/1 10:00|home|S1|V1|○|
|1/1 10:05|cart|S1|V1|○|
|1/3 14:00|product|S2|V1|○|
|1/3 14:10|checkout|S2|V1|○|
|1/2 09:00|home|S3|V2|-|
|1/2 09:08|cart|S3|V2|-|

* V1：S1（home/cart）と S2（product/checkout）の 2 訪問が存在 → cart 訪問 ✓・checkout 訪問 ✓ → V1 全データ返却
* V2：S3 に cart はあるが checkout 訪問なし → {{[Visit] page=checkout}} が false → V2 は除外

以下の 2 つは等価である：

{code}
[Visit]
    [Hit]
        page=home
    AND
    [Hit]
        page=product
{code}

{code}
[Visit]
    page=home AND page=product
{code}

h3. 3-3. 異なるスコープを AND で組み合わせた場合

外側コンテナが全体の返却スコープを決める。内側の条件はその範囲内で真偽を返すだけである。

{code}
[Visit]
    page=product
    AND
    [Visitor]
        lifetime_purchases > 0
{code}

返却スコープ：訪問（Visit コンテナが最外）
評価：この訪問に page=product があり、かつこの訪問者として過去購入実績がある

○ = 返却される、- = 返却されない：

||dateTime||page||visit_id||visitor_id||purchases||返却||
|1/1 10:00|home|S1|V1|0|-|
|1/1 10:05|*product*|S1|V1|0|-|
|1/2 09:00|*product*|S2|V2|2|○|
|1/2 09:08|cart|S2|V2|2|○|
|1/3 14:00|home|S3|V2|2|-|
|1/3 14:05|contact|S3|V2|2|-|

* S1（V1）：product ヒットあり、しかし V1 は purchases=0 → {{[Visitor] purchases > 0}} 不成立 → 訪問ごと除外
* S2（V2）：product ヒットあり・purchases=2 → 両条件成立 → S2 の全ヒット（cart を含む）を返却
* S3（V2）：purchases=2 だが product ヒットなし → {{[Visit] page=product}} 不成立 → 除外（V2 が購入者でも無関係）

{warning:title=返却スコープは Visit のまま}
V2 は purchases > 0 を満たすが、返却されるのは V2 の全データではなく *条件を満たした訪問（S2）のデータのみ*。返却スコープは最外コンテナ（Visit）が決める。
{warning}

*逆の入れ子構造：外側が小・内側が大のケース*

コンテナは Visitor → Visit → Hit の順（外が大・内が小）に限らない。逆順——外側が Hit・内側が Visitor——でも設定できる。原則は変わらない：*外側コンテナが返却スコープを決め、内側コンテナは条件の真偽を返す。*

{code}
[Hit]
    page=checkout
    AND
    [Visitor]
        lifetime_purchases > 0
{code}

返却スコープ：ヒット（Hit が最外）
評価：このヒットが page=checkout であり、かつこのヒットを行った訪問者が past purchases > 0 である

○ = 返却される、- = 返却されない：

||dateTime||page||visit_id||visitor_id||purchases||返却: ヒット単位||返却: 訪問者全体||
|1/1 10:00|home|S1|V1|0|-|-|
|1/1 10:05|checkout|S1|V1|0|-|-|
|1/1 10:10|product|S1|V1|0|-|-|
|1/2 09:00|home|S2|V2|2|-|○|
|1/2 09:05|checkout|S2|V2|2|○|○|
|1/2 09:08|cart|S2|V2|2|-|○|

左列（{{[Hit] checkout AND [Visitor] purchases>0}}）、右列（{{[Visitor] checkout AND purchases>0}}）：

* V1：purchases=0 → {{[Visitor] purchases > 0}} 不成立 → 両パターンとも全行 -
* V2 / ヒット単位：checkout ヒットのみ返却。home・cart は page≠checkout のため -
* V2 / 訪問者全体：checkout あり・purchases=2 → Visitor 条件成立 → home・checkout・cart すべて返却

{info:title=逆の入れ子の使いどころ}
「訪問者属性で絞り込みたいが、返却は特定ヒットだけにしたい」場面で有効。外側を Visitor にすると対象訪問者の全データが返ってくるが、外側を Hit にすれば条件を満たすヒットだけに絞れる。
{info}

----

h2. 4. 否定条件の意味：「存在しない」ではなく「存在する」

否定条件（{{!=}}・{{does not contain}}・{{does not exist}} 等）はコンテナのスコープ内で *条件を満たすヒットが存在するか* として評価される。AND/OR と同じ評価モデルが適用されるため、直感と外れた結果になりやすい。

{code}
[Visit]
    page != plp
{code}

これは「この訪問の中に page が plp でないヒットが存在するか」を評価する。home・product・cart など plp 以外のページを 1 件でも含む訪問なら true になる。

||ストリーム||{{[Visit] page != plp}}||意図と一致するか||
|home → product → cart|成立（home など plp 以外のヒットあり）|✅|
|home → plp → product|成立（home が plp 以外のヒットとして存在）|❌ 意図は「plp なし」なのに通過する|
|plp のみ|不成立（すべてのヒットが plp）|✅|

plp のみの訪問は除外されるが、plp と他ページが混在する訪問は通過してしまう。

*スコープ別の動作比較：*

||条件||評価の意味||
|{{[Hit] page != plp}}|そのヒットが plp でないか（直感と一致する）|
|{{[Visit] page != plp}}|訪問内に「plp でないヒット」が *存在するか*（他のページが 1 件でもあれば true）|
|{{[Visitor] page != plp}}|訪問者のヒット全体に「plp でないヒット」が *存在するか*（同上）|

ヒットコンテナでは否定条件が直感通り機能する。訪問・訪問者コンテナで「この訪問に plp が一切ない」を表現したい場合は否定条件では届かず、Exclude コンテナが必要になる（5 節で詳説）。

----

h2. 5. 除外コンテナのメカニズム

h3. 5-1. 除外コンテナの本質

除外コンテナは「行を物理的に削除する」のではなく、*スコープ単位で否定条件を返す* 仕組みである。

||除外コンテナ||否定の単位||意味||
|Exclude Hit|ヒット|このヒットが条件を満たす → *このヒットを除外する*|
|Exclude Visit|訪問|この訪問が条件を満たす → *この訪問を除外する*|
|Exclude Visitor|訪問者|この訪問者が条件を満たす → *この訪問者を除外する*|

h3. 5-2. スコープ一致が正しい除外の鍵

「plp を含まない訪問を取得したい」という要件に対して、3 通りのアプローチを比較する。

*パターン A：否定条件*
{code}
[Visit]
    page=home
    AND
    page != plp
{code}

評価：訪問内に「page=home かつ page≠plp のヒット」が存在するか。
page=home のヒットは常に page≠plp なので、*実質的に {{[Visit] page=home}} と等価*。
→ plp あり の訪問も通過 ❌

*パターン B：Exclude Hit（スコープミスマッチ）*
{code}
[Visit]
    page=home
    AND
    [Exclude Hit]
        page=plp
{code}

評価：Exclude Hit は各ヒットを個別に評価し、page=plp のヒットを除外（false）、それ以外を通過（true）とする。
AND はヒットレベルで結合されるため「page=home かつ {{[Exclude Hit]}} を通過したヒット」を探す。
page=home のヒットは plp ではないので {{[Exclude Hit]}} を必ず通過する。
→ こちらも*実質的に {{[Visit] page=home}} と等価*。plp あり の訪問も通過 ❌

*パターン C：Exclude Visit（正しい）*
{code}
[Visit]
    page=home
    AND
    [Exclude Visit]
        page=plp
{code}

評価フロー：
# この訪問に page=home のヒットが存在するか → true/false
# この訪問に page=plp のヒットが存在するか → true ならば *訪問全体を除外*（Exclude Visit）
# 両条件が成立する訪問のみを返す

→ home あり・plp なし の訪問のみ返る ✅

*同一データでの比較（○ = 返却、- = 返却されない）：*

||dateTime||page||visit_id||visitor_id||A: {{page!=plp}}||B: {{[Excl.Hit]plp}}||C: {{[Excl.Visit]plp}}||
|1/1 10:00|home|S1|V1|○|○|○|
|1/1 10:05|product|S1|V1|○|○|○|
|1/2 09:00|home|S2|V1|○|○|-|
|1/2 09:05|plp|S2|V1|○|○|-|
|1/2 09:10|product|S2|V1|○|○|-|
|1/3 11:00|home|S3|V2|○|○|○|
|1/3 11:08|contact|S3|V2|○|○|○|

* S1・S3（plp なし）：A・B・C すべて条件成立 → 全ヒット返却
* S2（plp あり）：A・B は visit が通過 → *plp ヒット自体も含む* S2 全ヒット返却 ❌ / C は訪問全体を除外 → すべて - ✅

*3 パターンの違いまとめ：*

||パターン||評価の単位||「plp なし」を表現できるか||等価な意味||
|否定条件 {{page != plp}}|ヒット|❌|「plp でないヒットが存在する訪問」|
|{{[Exclude Hit] page=plp}}|ヒット|❌|plp ヒットを除外するが AND がヒットレベルで結合 → 「page=home のヒットがある訪問」|
|{{[Exclude Visit] page=plp}}|訪問|✅|「plp ヒットを含む訪問全体を除外した訪問」|

{warning:title=スコープミスマッチの原則}
否定条件と Exclude Hit はどちらもヒットレベルで評価される。*訪問・訪問者スコープで「この条件を含まない」を表現するには、スコープが一致した Exclude コンテナが必要。*
{warning}

h3. 5-3. よくある誤りと正解パターン

||やりたいこと||誤ったセグメント||正しいセグメント||
|home あり、plp なし の訪問|{{[Visit] > home AND [Excl.Hit] > plp}}|{{[Visit] > home AND [Excl.Visit] > plp}}|
|購入した訪問者から購入訪問を除いたデータ|{{[Visitor] > [Excl.Hit] > order}}|{{[Visitor] > [Visit] > exists AND [Excl.Visit] > order}}|
|特定ヒットを含む訪問全体を除外したデータ|{{[Visit] > [Excl.Hit] > error_page}}|{{[Excl.Visit] > error_page}}（トップレベル）|

----

h2. 6. 順次セグメントのメカニズム

h3. 6-1. THEN 演算子：スキャンモデルの基本

順次セグメントは hit ストリームを *時系列に左から右へスキャン* するモデルで動作する。

動作原理：
# ヒットストリームを先頭からスキャンし、チェックポイント A に一致するヒットを探す
# A が見つかった地点を「通過点」として記録する
# 通過点以降のストリームを再スキャンしてチェックポイント B を探す
# B が見つかればシーケンス成立。見つからなければ次の A 候補から繰り返す

*greedy（欲張り）な動作：* シーケンスが複数の方法で成立する場合、AA は *最大データ量を返す組み合わせ* を選ぶ。

*{{Page A THEN Page B}} のヒットストリーム例：*

||パターン||結果||理由||
|A → B|成立|A の後に B あり|
|B → A|不成立|B の後に A があるが、A の後に B がない|
|A → C → B|成立|A の後に B あり（C は無視）|
|A → B → A → B|成立（greedy）|最初の A → B で成立。最大データを返す|

h3. 6-2. After 制約：スキャン開始位置の遅延

{quote}
The After operator is used to specify a minimum limit on the amount of time between two checkpoints. When setting the After values, the time limit begins when the segment is applied. For example, if the After operator is set on a container to identify visitors who visit page A, but don't return to visit page B until after one day, then that day will start when the visitor leaves page A.
— Adobe Analytics Documentation (Sequential segments)
URL: https://experienceleague.adobe.com/en/docs/analytics/components/segmentation/segmentation-workflow/seg-sequential-build
{quote}

*意味：* A の通過点（A を離れた時点）から、After 値が経過するまで B を探し始めない。

||単位||意味||
|時間（秒・分・時・日・週）|A を離れた時刻から X 時間以上経過後に B を探す|
|Hit(s)|A 以降 X ヒット以上経過後に B を探す|
|Visit(s)|A 以降 X 訪問以上経過後に B を探す|

*例：{{Page A THEN [After 1 Day] Page B}}*
A を閲覧してから 24 時間以内に発生した B は対象外。
24 時間経過後に B を閲覧したユーザーのみが対象。

○ = 返却される、- = 返却されない：

||dateTime||page||visit_id||visitor_id||A THEN [After 1 Day] B||
|1/1 10:00|A|S1|V1|-|
|1/1 15:00|B|S1|V1|-|
|1/1 10:00|A|S2|V2|○|
|1/3 11:00|B|S3|V2|○|
|1/3 11:10|other|S3|V2|○|

* V1：A (1/1 10:00) → B (同日 15:00)、5時間後 → After 1 Day 未満 → 不一致 → 全行 -
* V2：A (1/1 10:00) → B (1/3 11:00)、49時間後 → After 1 Day 以上 → 一致 → V2 全データ返却

h3. 6-3. Within 制約：スキャン窓の上限設定

{quote}
The Within operator is used to specify a maximum limit on the amount of time between two checkpoints. (...) that day begins when the visitor leaves page A. To be included in the segment, the visitor has a maximum time of one day before opening page B.
— Adobe Analytics Documentation (Sequential segments)
{quote}

*意味：* A の通過点（A を離れた時点）から、Within 値の範囲内に B が見つからなければ不成立。

||単位||意味||
|時間（秒・分・時・日・週）|A を離れた時刻から X 時間以内に B が現れること|
|Hit(s)|A 以降 X ヒット以内に B が現れること|
|Visit(s)|A 以降 X 訪問以内に B が現れること|

*例：{{Page A THEN [Within 30 Days] Page B}}*
A から 30 日以内に B が出現した場合のみ成立。

○ = 返却される、- = 返却されない：

||dateTime||page||visit_id||visitor_id||A THEN [Within 1 Day] B||
|1/1 10:00|A|S1|V1|○|
|1/1 20:00|B|S1|V1|○|
|1/1 09:00|A|S2|V2|-|
|1/3 15:00|B|S3|V2|-|

* V1：A (1/1 10:00) → B (1/1 20:00)、10時間後 → Within 1 Day 内 → 一致 → V1 全データ返却
* V2：A (1/1 09:00) → B (1/3 15:00)、30時間後 → Within 1 Day 超過 → 不一致 → -

h3. 6-4. After と Within の組み合わせ：スキャン窓の定義

{quote}
After but Within: When using both the After and Within operators, both operators start and end in parallel, not sequentially. (...) Both conditions are enforced from the time of the first page view.
— Adobe Analytics Documentation (Sequential segments)
{quote}

After と Within を同時に設定すると、*両制約が同じ基点（A の通過点）から同時に適用される。*

{code}
時間軸：
A ---|---After---|--------Within---|---->
              ↑ここから        ↑ここまで
              B を探す窓 [After, Within]
{code}

*例：{{Visited App THEN [After 1 Day, Within 30 Days] Purchased}}*
= アプリ訪問の 1 日後〜30 日以内に購入

||After||Within||B を探す窓||
|1 Day|30 Days|A から 1 日後〜30 日以内|
|1 Hit|5 Hits|A から 1 ヒット後〜5 ヒット以内|
|なし|7 Days|A の直後〜7 日以内（下限なし）|
|1 Day|なし|A から 1 日後以降（上限なし）|

h3. 6-5. EXCLUDE チェックポイント：順次での役割

順次セグメント内の EXCLUDE チェックポイントは、*位置（最初・途中・最後）によって役割が根本的に異なる。*

h4. 6-5-1. 最初が EXCLUDE：(EXCLUDE A) THEN B

*意味：* 「B に到達した時点で、それより前に A が存在しないこと」

EXCLUDE は *qualifying 条件（不在の証明）* として機能する。シーケンスの成立条件を定義するが、Only After Sequence の起点の判断には使われない。

||パターン||一致||理由||
|A → B|不一致|B より A が先に出現|
|C → B|一致|B より前に A なし。起点 ＝ B|
|A → C → B|不一致|B より A が先に出現（C が間にあっても同様）|
|C → B → A → B|最初の B で一致|最初の B 到達時点では A なし。起点 ＝ 最初の B|

h4. 6-5-2. 途中が EXCLUDE：A THEN (EXCLUDE B) THEN C

*意味：* 「A の後 C に到達したが、A と C の間に B が出現しなかった」

EXCLUDE は *qualifying 条件（区間内の不在）* として機能する。Only After Sequence の起点は C（最後の INCLUDE）。

{info:title=参考}
[Excluding between checkpoints（Tips & Tricks）|https://experienceleague.adobe.com/en/docs/events/the-skill-exchange-recordings/analytics/may2022/tips-and-tricks]
{info}

||パターン||一致||理由||
|A → C|一致|A と C の間に B なし。起点 ＝ C|
|A → X → C|一致|間に X はあるが B なし。起点 ＝ C|
|A → B → C|不一致|A と C の間に B あり|
|A → B → C → A → C|2 つ目の A → C で一致|2 つ目の A 以降に B なし。起点 ＝ 2 つ目の C|

h4. 6-5-3. 最後が EXCLUDE：A THEN B THEN (EXCLUDE C)

*意味：* 「A の後 B に到達し、C が現れるまでのデータを収集する」

EXCLUDE は *データ収集の終端（範囲の境界）* として機能する。C のヒット自体は収集されない。Only After Sequence の起点は B（最後の INCLUDE）。

{quote}
"It'll continue until it reaches the end of the person/visit data set, or it hits an exclude."
— Adobe Customer Success Webinar (Mastering Sequential Logic: Starts and Stops)
URL: https://experienceleague.adobe.com/en/docs/events/adobe-customer-success-webinar-recordings/2025/multisolution2025/sequential-logic-start-stop
{quote}

||パターン||収集されるデータ||
|A → B → C|B のヒットのみ（C は EXCLUDE）|
|A → B → X → C|B・X（C は EXCLUDE）|
|A → B → X → Y|B・X・Y（C なし → データ末尾まで）|

h4. まとめ：EXCLUDE の位置と役割

||位置||構成例||EXCLUDE の役割||Only After Sequence の起点||
|*最初*|(EXCLUDE A) THEN B|qualifying 条件（B より前に A が存在しない）|B|
|*途中*|A THEN (EXCLUDE B) THEN C|qualifying 条件（A と C の間に B が存在しない）|C|
|*最後*|A THEN B THEN (EXCLUDE C)|データ収集の終端（C が現れるまで）|B|

{warning:title=重要}
最初・途中の EXCLUDE は「シーケンスが成立するための条件」として機能する。
最後の EXCLUDE だけは「データ収集がどこまで続くか」を定義する点で根本的に役割が異なる。
{warning}

----

h2. 7. Only Before / Only After Sequence のメカニズム

h3. 7-1. Include Everyone との違い

3 つのモードで「*どの範囲のデータを返すか*」が変わる。

||モード||データの返却範囲||
|*Include Everyone*|シーケンスを qualifying 条件として使い、条件を満たした訪問者/訪問の全データを返す|
|*Only After Sequence*|シーケンスの「起点」*以降*のデータのみを返す|
|*Only Before Sequence*|シーケンスの「終点」*以前*のデータのみを返す|

h3. 7-2. Only After Sequence の起点決定

{quote}
"As soon as all those THEN checkpoints that are includes, not excludes, are met, then it starts to match the data."
— Adobe Customer Success Webinar (Mastering Sequential Logic: Starts and Stops)
URL: https://experienceleague.adobe.com/en/docs/events/adobe-customer-success-webinar-recordings/2025/multisolution2025/sequential-logic-start-stop
{quote}

*起点 = シーケンス内の最後の INCLUDE チェックポイントに一致したヒット*

* EXCLUDE チェックポイントは起点の判定に使われない
* シーケンスが複数回成立する場合：greedy の原則により、最大データを返す成立が採用される

*例：{{A THEN B THEN (EXCLUDE C)}} + Only After Sequence*
→ 起点：B（最後の INCLUDE）のヒット以降
→ 収集：B 以降のすべてのデータ。ただし C に達した時点で収集終了

*Include Everyone と Only After Sequence の返却範囲の違い（{{A THEN B}} の場合）：*

○ = 返却される、- = 返却されない：

||dateTime||page||visit_id||visitor_id||Include Everyone||Only After Sequence||
|1/1 10:00|A|S1|V1|○|-|
|1/1 10:05|C|S1|V1|○|-|
|1/3 14:00|B|S2|V1|○|○|
|1/3 14:10|D|S2|V1|○|○|

* Include Everyone：シーケンス成立 → 訪問者の *全データ* を返す（A・C を含む）
* Only After Sequence：起点（B）以降のデータのみ返す。B 以前の A・C は含まれない

h3. 7-3. Only Before Sequence の終点決定

Only After の鏡像（対称的な構造）。時系列を遡るようにイメージすると理解しやすい。

*終点 = シーケンス内の最初の INCLUDE チェックポイントに一致したヒット*

* 収集されるのは終点 *以前* のデータ
* EXCLUDE を最初に置くことで「逆方向の終端境界」を設定できる

*例：{{(EXCLUDE C) THEN A}} + Only Before Sequence*
→ 終点：A（最初の INCLUDE）のヒット以前
→ 収集：A より前のすべてのデータ。ただし逆方向に C に達した時点で収集終了

*Include Everyone と Only Before Sequence の返却範囲の違い（{{A THEN B}} の場合）：*

○ = 返却される、- = 返却されない：

||dateTime||page||visit_id||visitor_id||Include Everyone||Only Before Sequence||
|1/1 09:00|pre|S1|V1|○|○|
|1/1 10:00|A|S1|V1|○|-|
|1/1 10:05|C|S1|V1|○|-|
|1/3 14:00|B|S2|V1|○|-|
|1/3 14:10|post|S2|V1|○|-|

* Include Everyone：シーケンス成立 → 訪問者の *全データ* を返す
* Only Before Sequence：終点（A = 最初の INCLUDE）より前のデータのみ返す。A 自体・C・B・post は含まれない

h3. 7-4. EXCLUDE と Only Before/After の組み合わせ

{quote}
"It's going to continue in the direction you selected based on only after / only before until it may see the exclude checkpoint condition triggered."
— Adobe Customer Success Webinar (Mastering Sequential Logic: Starts and Stops)
{quote}

||構成||EXCLUDE の役割||
|Only After + 最後の EXCLUDE|データ収集の *前方終端*（C が現れたら収集終了）|
|Only Before + 最初の EXCLUDE|データ収集の *後方終端*（逆方向で C が現れたら収集終了）|

この 2 つを組み合わせることで、*特定のイベント間のデータ切り出し* が可能になる。

*活用例：アプリ申し込み後の「直後 1 セッション」だけを取得する*
{code}
[Only After Sequence]
    [Visit]
        App submit event        # 起点候補の前提条件
    THEN
    [Visit]
        Day exists              # 起点（この訪問の開始ヒット以降）
    THEN
    [Exclude Visit]
        Day exists              # 終端（次の訪問が始まったら収集終了）
{code}

この構成で「申し込み後の最初の 1 セッション」に限定されたデータを返す。

*簡易例：{{[Only After Sequence] A THEN B THEN (EXCLUDE C)}}*

○ = 返却される、- = 返却されない：

||dateTime||page||visit_id||visitor_id||返却||
|1/1 10:00|A|S1|V1|-|
|1/1 10:08|X|S1|V1|-|
|1/2 09:00|B|S2|V1|○|
|1/2 09:10|Y|S2|V1|○|
|1/3 14:00|Z|S3|V1|○|
|1/3 14:20|C|S3|V1|-|

* A → B のシーケンス成立。Only After 起点 ＝ B → B 以前の A・X は除外
* B 以降（Y・Z）は収集。C に達した時点で収集終了（C 自体は除外）

----

h2. 8. 複合設計の読み方

h3. 8-1. セグメントを「読む」処理手順

複雑なセグメントに直面した時は、以下の手順で評価を追う。

# *最外コンテナのスコープを確認する*（= 返却スコープ）
# *子コンテナ・条件を内側から外側に向けて評価する*
# *除外コンテナが見つかったら、そのスコープで否定を適用する*
# *順次条件が含まれる場合、スキャンモデルで通過点を特定する*
# *Only After/Before が指定されていれば、起点/終点でデータを切り出す*

h3. 8-2. 読み解き例

*例 1（複合除外）：*
{code}
[Visitor]
    [Visit]
        page=product
    AND
    [Exclude Visit]
        event=purchase
{code}
* 返却スコープ：訪問者（Visitor コンテナ）
* 条件 1：product ページを含む訪問が存在する（Visit コンテナ → 真偽値）
* 条件 2：purchase イベントを含む訪問が存在しない（Exclude Visit → 訪問単位で否定 → 真偽値）
* 結果：product を見たが購入はしていない訪問者の全データ ✅

○ = 返却される、- = 返却されない：

||dateTime||page||event||visit_id||visitor_id||返却||
|1/1 10:00|product|-|S1|V1|-|
|1/1 10:08|cart|-|S1|V1|-|
|1/2 09:00|checkout|purchase|S2|V1|-|
|1/1 11:00|product|-|S3|V2|○|
|1/1 11:10|cart|-|S3|V2|○|
|1/3 14:00|info|-|S4|V2|○|

* V1：product 訪問あり（S1）、しかし purchase 訪問も存在（S2）→ Exclude Visit 発動 → V1 全体除外
* V2：product 訪問あり（S3）、purchase 訪問なし → 条件成立 → V2 全データ返却

*例 2（順次 + 時間制約）：*
{code}
[Visitor]
    [Visit]
        page=product
    THEN [After 1 Day, Within 30 Days]
    [Visit]
        event=purchase
{code}
* 返却スコープ：訪問者
* シーケンス：product ページ閲覧訪問の後、1 日後〜30 日以内に purchase イベント訪問
* 時間制約の基点：product 閲覧訪問の終了時点（A を離れた時点）
* 結果：product 閲覧から 1 日後〜30 日以内に購入した訪問者の全データ

○ = 返却される、- = 返却されない：

||dateTime||page||event||visit_id||visitor_id||返却||
|1/1 10:00|product|-|S1|V1|○|
|1/1 10:08|cart|-|S1|V1|○|
|1/2 11:00|checkout|purchase|S2|V1|○|
|1/1 09:00|product|-|S3|V2|-|
|1/1 14:00|checkout|purchase|S3|V2|-|

* V1：product 訪問（1/1 10:00）→ purchase 訪問（1/2 11:00）、25時間後 → After 1 Day 以上・Within 30 Days 以内 → 一致 → V1 全データ返却
* V2：product 訪問（1/1 09:00）→ purchase 訪問（同日 14:00）、5時間後 → After 1 Day 未満 → 不一致 → -

*例 3（Only After + 終端 EXCLUDE）：*
{code}
[Only After Sequence]
    [Visit]
        page=checkout
    THEN
    [Hit]
        event=order_confirm
    THEN
    [Exclude Hit]
        page=homepage
{code}
* Only After 起点：order_confirm ヒット（最後の INCLUDE）以降
* 終端：homepage ヒットが出現した時点で収集終了
* 結果：購入完了後から次にホームを訪問するまでの行動データ

○ = 返却される、- = 返却されない：

||dateTime||page||event||visit_id||visitor_id||返却||
|1/1 10:00|checkout|-|S1|V1|-|
|1/1 10:10|-|order_confirm|S1|V1|○|
|1/1 10:12|thankyou|-|S1|V1|○|
|1/2 09:00|product|-|S2|V1|○|
|1/2 09:08|homepage|-|S2|V1|-|

* checkout ヒットは order_confirm（起点）より前 → Only After により除外
* order_confirm・thankyou・product は起点以降かつ homepage 前 → 収集
* homepage は Exclude Hit → 収集終了（homepage 自体は除外）

h3. 8-3. 設計時のチェックリスト

*除外を設計する時：*
* 「何を除外したいか」の粒度は？ヒット・訪問・訪問者のいずれか？
* Exclude コンテナのレベルはその粒度と一致しているか？
* Exclude Hit を訪問コンテナの AND に入れていないか？（スコープミスマッチに注意）

*順次を設計する時：*
* greedy の影響で意図しない早期マッチが起きないか？
* After/Within の基点は「A を離れた時点」であることを確認しているか？
* EXCLUDE の位置は最初・途中・最後のどこか？それぞれの役割を把握しているか？

*Only After/Before を設計する時：*
* 起点/終点は何のヒットに相当するか？
* データ収集を途中で止めたい場合、終端 EXCLUDE を設定しているか？
* 複数回シーケンスが成立する場合、greedy でどの成立が選ばれるか確認しているか？

----

{info:title=参考ドキュメント}
* [Sequential segments（AA）|https://experienceleague.adobe.com/en/docs/analytics/components/segmentation/segmentation-workflow/seg-sequential-build]
* [Sequential segments（CJA）|https://experienceleague.adobe.com/en/docs/analytics-platform/using/cja-components/segments/seg-sequential-build]
* [Mastering Sequential Logic: Starts and Stops（ウェビナー）|https://experienceleague.adobe.com/en/docs/events/adobe-customer-success-webinar-recordings/2025/multisolution2025/sequential-logic-start-stop]
* [Mastering Sequential Logic: A Visual Framework（ウェビナー）|https://experienceleague.adobe.com/en/docs/events/adobe-customer-success-webinar-recordings/2025/multisolution2025/mastering-sequential-logic]
* [Excluding between checkpoints（Tips & Tricks）|https://experienceleague.adobe.com/en/docs/events/the-skill-exchange-recordings/analytics/may2022/tips-and-tricks]
{info}

_作成日：2026-04-23_
