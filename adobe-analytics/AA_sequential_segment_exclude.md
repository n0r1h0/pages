h1. Adobe Analytics 順次セグメント：EXCLUDEの位置による挙動の違い

h2. 概要

Adobe Analytics の順次セグメントでは、{{THEN}} 演算子で繋いだチェックポイントに *EXCLUDE（除外）* を設定できる。
EXCLUDEの *位置（最初・途中・最後）* によって、その役割が根本的に異なる。

{info:title=参考ドキュメント}
* [Sequential segments（AA）|https://experienceleague.adobe.com/en/docs/analytics/components/segmentation/segmentation-workflow/seg-sequential-build]
* [Sequential segments（CJA）|https://experienceleague.adobe.com/en/docs/analytics-platform/using/cja-components/segments/seg-sequential-build]
* [Mastering Sequential Logic: Starts and Stops（ウェビナー）|https://experienceleague.adobe.com/en/docs/events/adobe-customer-success-webinar-recordings/2025/multisolution2025/sequential-logic-start-stop]
{info}

----

h2. Only After Sequence の基本原則

{{Only After Sequence}} を設定した場合、*データ取得の起点は「最後のINCLUDEチェックポイントに一致したhit」* で決まる。

{quote}
"As soon as all those THEN checkpoints that are includes, not excludes, are met, then it starts to match the data."
— Adobe Customer Success Webinar
{quote}

EXCLUDEチェックポイントは起点の判断に使われない。ただし後述のとおり、*位置によって機能が異なる*。

----

h2. EXCLUDEの位置別の役割

h3. 1. 最初が EXCLUDE：(EXCLUDE A) THEN B

*意味*：「Bに到達した時点で、それより前にAが存在しないこと」

* EXCLUDEは *qualifying条件（不在の証明）* として機能する
* Only After Sequence の起点：*B*（唯一のINCLUDE）

*hitストリーム例*

||パターン||一致||理由||
|A→B|不一致|BよりAが先に出現|
|C→B|一致|Bより前にAは出現していない。起点＝B|
|A→C→B|不一致|BよりAが先に出現（Cが間にあっても同様）|
|C→B→A→B|最初のBで一致|最初のB到達時点ではAなし。起点＝最初のB|

----

h3. 2. 途中が EXCLUDE：A THEN (EXCLUDE B) THEN C

*意味*：「AのあとCに到達したが、AとCの間にBが出現しなかった」

* EXCLUDEは *qualifying条件（区間内の不在）* として機能する
* Only After Sequence の起点：*C*（最後のINCLUDE）

{info:title=参考}
[Excluding between checkpoints（Tips & Tricks）|https://experienceleague.adobe.com/en/docs/events/the-skill-exchange-recordings/analytics/may2022/tips-and-tricks]
{info}

*hitストリーム例*

||パターン||一致||理由||
|A→C|一致|AとCの間にBなし。起点＝C|
|A→X→C|一致|間にXはあるがBなし。起点＝C|
|A→B→C|不一致|AとCの間にBあり|
|A→B→C→A→C|2つ目のA→Cで一致|2つ目のA以降にBなし。起点＝2つ目のC|

----

h3. 3. 最後が EXCLUDE：A THEN B THEN (EXCLUDE C)

*意味*：「AのあとBに到達し、Cが現れるまでのデータを収集する」

* EXCLUDEは *データ収集の終端（範囲の境界）* として機能する
* Only After Sequence の起点：*B*（最後のINCLUDE）
* Cのhit自体は収集されない

{quote}
"It'll continue until it reaches the end of the person/visit data set, or it hits an exclude."
— Adobe Customer Success Webinar
{quote}

*hitストリーム例*

||パターン||収集されるデータ||
|A→B→C|Bのhitのみ（CはEXCLUDE）|
|A→B→X→C|B・X（CはEXCLUDE）|
|A→B→X→Y|B・X・Y（Cなし → データ末尾まで）|

----

h2. まとめ：EXCLUDEの位置と役割

||位置||構成例||EXCLUDEの役割||Only After Sequence の起点||
|*最初*|(EXCLUDE A) THEN B|qualifying条件（Bより前にAが存在しない）|B|
|*途中*|A THEN (EXCLUDE B) THEN C|qualifying条件（AとCの間にBが存在しない）|C|
|*最後*|A THEN B THEN (EXCLUDE C)|データ収集の終端（Cが現れるまで）|B|

{warning:title=重要}
最初・途中のEXCLUDEは「シーケンスが成立するための条件」として機能する。
最後のEXCLUDEだけは「データ収集がどこまで続くか」を定義する点で根本的に役割が異なる。
{warning}

----

h2. 補足：Include Everyone / Only Before Sequence との関係

上記はすべて {{Only After Sequence}} を前提とした説明。

* *Include Everyone*（デフォルト）の場合、起点・終端の概念はなく、EXCLUDEはシーケンスの qualifying条件のみとして機能する
* *Only Before Sequence* では評価が逆方向（時系列の末尾から）になるため、最後のEXCLUDEが「終端」の役割を担う（Only After と対称の関係）

----

_作成日：2026-04-22 / 参照：Adobe Experience League ドキュメント・Customer Success Webinar_
