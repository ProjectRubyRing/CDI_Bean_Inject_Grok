# CDI の印があるのに @Inject が全部きかない理由

Bean アーカイブが無いと、コンテナはクラスを見に行かない。  
`@Inject` は「牛乳をください」と書いた付箋である。先生（CDI コンテナ）が教室を回らない限り、付箋は紙のまま残る。

- 基準日: 2026-10-08
- 仕様: Jakarta CDI 4.1（Jakarta EE 11）、CDI 5.0（Jakarta EE 12、2026-09-11 に Release Review 可決）
- 実装の確認元: CDI 4.1 仕様本文、`beans_4_1.xsd`、Weld 7.0.0.Final リファレンス、WildFly 38 の weld サブシステム、Open Liberty の cdi 設定、Payara の empty beans.xml 互換、Quarkus CDI リファレンス（2026-01 版の記述）、Apache OpenWebBeans の設定項目
- 同じ内容の表: `docs/cdi-bean-archive-inject.xlsx`（フォントは Meiryo UI、色は白・灰・黒）

この資料は、次の一文を分解する。

> CDI のアノテーションは付いている。それなのに Bean アーカイブが無いので、`@Inject` がまるごと動いていない。

「アノテーションがある」と「Bean アーカイブである」は、別の出来事である。ここを分けると、黙って null のままになる理由が説明できる。

---

## 1. 最初に結論

`@Inject` は Java のプログラムを動かす命令ではない。実行時まで残るメモである。読む人は CDI コンテナだけである。

コンテナは、最初に「どの教室を回るか」を決める。その名簿が Bean アーカイブである。名簿に入っていない教室では、次のことが全部起きない。

1. クラスを Bean として登録しない
2. `@Inject` の付いたフィールド、コンストラクタ、メソッドを注入点として登録しない
3. 注入先が足りない、という起動エラーも出さない
4. 実行時にフィールドへオブジェクトを入れない。参照型は null のまま

名簿に入る条件は、時代と置き場所で違う。2026 年の標準的な分け方は次のとおり。

| 状態 | Bean アーカイブか | 何が Bean になるか |
| --- | --- | --- |
| `beans.xml` が無く、Bean 定義アノテーションもセッション Bean も無い | ならない | 何もならない。`@Inject` は読まれない |
| `beans.xml` が無く、`@ApplicationScoped` などの Bean 定義アノテーションがある | 暗黙の Bean アーカイブになる（Jakarta EE サーバの既定） | 定義アノテーションの付いた型だけ |
| `beans.xml` の `bean-discovery-mode="none"` | ならない | 何もならない |
| 移植可能拡張、または Build Compatible Extension が入っていて `beans.xml` が無い | ならない | 拡張のサービス登録は別。同じ JAR の普通のクラスは Bean にならない |
| 中身が完全に空の `beans.xml`（CDI 4.0 以降） | 暗黙の Bean アーカイブ | `annotated`。定義アノテーションのある型だけ |
| 中身が完全に空の `beans.xml`（CDI 1.0 から 3.0） | 明示的な Bean アーカイブ | `all`。条件を満たすクラスはスコープ無しなら `@Dependent` |
| `bean-discovery-mode="annotated"` | 暗黙 | 定義アノテーションのある型だけ |
| `bean-discovery-mode="all"` | 明示 | アーカイブ内の対象クラス。スコープ無しは `@Dependent` |
| `bean-discovery-mode="all"` かつ `<trim/>` | 明示だが、Bean 化は狭い | 定義アノテーションか、何らかのスコープがある型だけが Bean。拡張向けの発見イベントは広い |

`@Inject`、`@Produces`、`@Named`、自作の `@Qualifier`、`@Alternative`、`@Specializes`、`@Priority`、`@Transactional`、`jakarta.inject.Singleton` は、クラスに付いていても Bean アーカイブを作らない。`@Inject` だけが付いたクラスは、「付箋はあるが、当番の机ではない」。

Weld をサーブレットコンテナや Java SE で単体利用する場合は、もう一段厳しい。Weld 7 のリファレンスでは、`beans.xml` の無い暗黙アーカイブは、Weld Servlet では発見対象から外れる。Weld SE でも、クラスパス走査を明示するまで、`beans.xml` の無い暗黙アーカイブは既定で外れる。

---

## 2. 小学生にもわかるたとえ

学校の給食を想像する。

各教室には机がある。机が Java のクラスである。  
机に貼られた付箋がアノテーションである。  
「牛乳をください」という付箋が `@Inject` である。

牛乳を運ぶのは建物ではない。給食の先生である。先生が CDI コンテナである。

先生は、全校の机を勘で回らない。当番表に載った教室だけを回る。当番表が Bean アーカイブである。

当番表の作り方は 2 つある。

1. 教室のドアに「この教室を回ってください」と書いた紙を貼る。この紙が `beans.xml` である。
2. 紙が無くても、机に「わたしは給食当番の机です」という特別な名札が付いていれば、Jakarta EE の学校では先生がその教室を見つける。特別な名札が Bean 定義アノテーションである。`@ApplicationScoped` や `@RequestScoped` や `@Dependent` がそれにあたる。

「牛乳をください」は特別な名札ではない。お願いの付箋である。  
お願いの付箋だけの教室に、ドアの紙も無いと、先生は廊下を素通りする。牛乳は来ない。放送で「牛乳が足りません」とも言わない。先生がその教室を担当表に入れてないから、不足にも気づけない。

これが、「アノテーションはあるのに、`@Inject` がまるごときいていない」の正体である。

先生が教室を担当しているときに、頼んだ牛乳そのものが学校に無い場合は話が変わる。給食が始まる前に「この牛乳は用意できません」とエラーになる。Weld なら `WELD-001408` のような起動エラーである。黙って null になる現象とは別である。

もう一つ。先生が運ぶのは、先生が作った給食である。自分で買ってきたパン（`new`）には、先生は牛乳を注がない。Bean アーカイブが正しくても、`new MyService()` で作ったオブジェクトの `@Inject` は空のままである。

---

## 3. 動作イメージ

時間の順に、コンテナの頭の中を追う。

### 3.1 コンパイルが終わった時点

クラスファイルの中に、アノテーションが属性として入っている。`@Inject` の保持期間は `RUNTIME` なので、起動後に反射で読める。

この時点で注入は起きない。JVM はアノテーションを見てコンストラクタ引数を埋めない。フレームワークが読むまで、フィールドの初期値は言語の既定値である。参照なら null、数なら 0、boolean なら false。

### 3.2 起動時、アーカイブを並べる

コンテナは「1 つのクラス」ではなく「アーカイブ」を単位に見る。アーカイブは、JAR、WAR の `WEB-INF/classes`、EAR の中の各モジュール、クラスパス上のディレクトリである。

WAR の中の `WEB-INF/lib/a.jar` と `WEB-INF/lib/b.jar` は、別々のアーカイブである。片方に `beans.xml` があっても、もう片方のクラスは自動では Bean にならない。

### 3.3 Bean アーカイブかどうかを決める

各アーカイブについて、記述子の有無、記述子のモード、Bean 定義アノテーション、セッション Bean、拡張の有無を見る。

ここで落ちると、そのアーカイブは以降の工程に出てこない。`@Inject` は存在していても、注入点の一覧に載らない。

### 3.4 発見モードで「見るクラス」を決める

アーカイブだと決まったあと、モードで範囲が変わる。

- `all`: そのアーカイブのクラスを候補にする。スコープが無ければ `@Dependent` の Bean にする。CDI Full の機能である。CDI Lite は `all` を要求しない。
- `annotated`: Bean 定義アノテーションのあるクラスだけを候補にする。
- `none`: アーカイブごと対象外。記述子はあっても、Bean アーカイブではない。

CDI 4.0 以降、モード属性を省略したときの既定は `annotated` である。中身が空のファイルも `annotated` として扱う。CDI 3.0 までは、空ファイルは `all` だった。

### 3.5 除外する

候補から、さらに次を外す。

- `beans.xml` の `<scan><exclude name="..."></exclude></scan>` に一致した型
- `@Vetoed` が付いた型、または `@Vetoed` の付いたパッケージ
- CDI Full の `<trim/>` により、定義アノテーションもスコープも無い型（`all` の明示アーカイブで使う）

除外は「見たうえで外す」である。Bean アーカイブに入っていない場合は、除外ルールまで到達しない。

### 3.6 Bean の設計図を作る

残ったクラスについて、コンテナは設計図を作る。設計図には、型、限定子、スコープ、名前、注入点、プロデューサ、オブザーバが入る。

`@Inject` が付いた場所は、この設計図の「注入点」になる。注入点は次の 3 か所である。

- コンストラクタが 1 つだけ `@Inject` されている。引数の型が依存先
- フィールドに `@Inject` がある
- イニシャライザメソッド（セットするメソッド）に `@Inject` がある

クラスに `@Inject` コンストラクタが無く、引数なしコンストラクタがある場合、コンテナは引数なしコンストラクタで作り、そのあとフィールドとイニシャライザを注入する。

### 3.7 起動が終わる前に、注入点を検査する

Bean アーカイブに入っていて、クラスが Bean になっていて、その注入点の相手が 1 つに定まらないとき、起動は失敗する。

- 相手が 0 個: 依存が満たされない。Weld は `WELD-001408`
- 相手が 2 個以上: 依存があいまい。Weld は `WELD-001409`

この検査は「担当している教室」だけに行う。担当外の教室の `@Inject` は検査されない。だからエラーが無く、実行時に null になる。

### 3.8 実行時、コンテナが作ったインスタンスだけ注入する

だれかが Bean を必要としたとき、コンテナは次の順で動く。

1. スコープの文脈から、既存のインスタンスを探す。`@ApplicationScoped` ならアプリケーションに 1 つ。`@RequestScoped` ならリクエストに 1 つ。`@Dependent` なら注入のたびに新しいインスタンス
2. 無ければ作る。`@Inject` コンストラクタがあれば、引数を先に解決してから呼ぶ
3. フィールド注入とイニシャライザメソッドを実行する
4. `@PostConstruct` を呼ぶ
5. 通常スコープなら、呼び出し側へは実体ではなくクライアントプロキシを渡すことが多い

`new` はこの列に入らない。言語の `new` はコンテナを呼ばない。

### 3.9 サーブレットなど、コンテナが別々に作る部品

サーブレット、フィルタ、リスナ、JAX-RS リソース、メッセージ駆動 Bean は、それぞれの仕様のコンテナがインスタンスを作る。CDI の Bean そのものではないものもある。

Jakarta EE では、そのモジュールが Bean アーカイブであるときに、CDI へ「このインスタンスの `@Inject` を埋めて」と頼む。モジュールが Bean アーカイブでなければ、この依頼が無い。サーブレットは生成される。`@Inject` のフィールドは null のまま、リクエストが動き始める。起動エラーは出ない。

これが、Web アプリケーションで「アノテーションはあるのに注入が全部死んでいる」と見える典型である。

---

## 4. なぜこの仕組みが必要になったか

### 4.1 2009 年、CDI 1.0。紙が無い教室は最初から担当外

Java EE 6 の CDI 1.0（JSR 299）は、Gavin King らの設計で、参照実装は Weld である。

当時の DI は、XML に配線を全部書くか、自前のファクトリを書くかが多かった。CDI は「型とアノテーションで配線し、XML は配備のときにだけ使う」という形にした。

その代わり、誤って全クラスパスを Bean にしないよう、参加の印を必須にした。印が `META-INF/beans.xml`、WAR では `WEB-INF/beans.xml` である。中身は空でよい。空でも「このアーカイブのクラスは Bean の候補」という意味だった。今日の `bean-discovery-mode="all"` にあたる。

紙が無い JAR は、CDI の世界の外である。`@Inject` が付いていても読まない。ライブラリ JAR を置いただけで、見知らぬクラスが Bean になり、起動が落ちるのを避けるためである。

### 4.2 2013 年、CDI 1.1。紙を忘れても動くようにした

Java EE 7 の CDI 1.1 で、`bean-discovery-mode` が生まれた。値は `all`、`annotated`、`none` の 3 つである。

同時に、暗黙の Bean アーカイブが生まれた。`beans.xml` が無くても、Bean 定義アノテーションのあるクラスがあれば、そのアーカイブを `annotated` として扱う。EE 7 は「CDI を標準の部品モデルにする」方針で、空の XML を置き忘れる事故を減らしたかった。

副作用がすぐ出た。コンテナはクラスパス上の JAR を広く見るようになった。JSR 330 のアノテーションを別目的で使うライブラリまで CDI の検査に巻き込まれ、起動が遅くなり、意図しない Bean が増え、あいまいな依存で配備が失敗した。Guice や Guava のようなライブラリが話題になった。

そこで仕様は、互換のためのスイッチを製品に義務づけた。「`beans.xml` が無いアーカイブは無視する」という、CDI 1.0 の動きに戻す選択肢である。WildFly の `require-bean-descriptor`、OpenWebBeans の `scanBeansXmlOnly`、Payara / GlassFish 系の `enable-implicit-cdi` は、この流れの具体的なつまみである。

### 4.3 2014 年、CDI 1.2。特別な名札を減らした

暗黙アーカイブが広すぎたので、Bean 定義アノテーションの集合を狭くした。`@Named` は定義アノテーションから外れた。`jakarta.inject.Singleton`（当時は `javax.inject.Singleton`）も定義アノテーションではない、と明確になった。

理由は、名前を付けただけのクラスや、別の DI のシングルトン印まで「CDI の机」にしてしまうと、また全 JAR が当番表に載るからである。

残った定義アノテーションは、通常スコープ、`@Dependent`、ステレオタイプ、`@Interceptor`、`@Decorator` である。お願いの `@Inject` は、最初からこの集合に入っていない。

### 4.4 2017 年、CDI 2.0。SE と trim

Java EE 8 の CDI 2.0 で、Java SE からコンテナを起動する `SeContainer` と `SeContainerInitializer` が標準になった。サーバが無いプログラムでも CDI を使える。ただし「どのクラスパスを見るか」は実装依存が大きく、Weld SE は安全側に倒して、`beans.xml` の無い暗黙アーカイブを既定では見ない。

同じ版で `<trim/>` が追加された。`all` は拡張が全クラスを観察できる一方、ユーティリティクラスまで `@Dependent` Bean になり、メモリとプロキシの問題を起こした。`trim` は「観察は広く、Bean にするのは印のある型だけ」という折衷である。

`@Priority` により、インターセプタ、デコレータ、オルタナティブを `beans.xml` に書かなくてもアプリケーション全体で有効にできるようになった。記述子の役割は「Bean を全部列挙する名簿」から「発見範囲と、アーカイブ単位の選択」へ寄っていった。

### 4.5 2020 年、CDI 3.0。パッケージ名が javax から jakarta へ

Jakarta EE 9 で、API のパッケージが `javax.*` から `jakarta.*` に変わった。発見ルール自体は CDI 2.0 と同じで、空の `beans.xml` は引き続き `all` である。

ここでありがちな事故が、import の取り残しである。ソースに `@Inject` と書いてあっても、型が `javax.inject.Inject` のままなら、CDI 3 以降のコンテナはそれを注入点とみなさない。アノテーションはクラスファイルに存在する。コンテナの知っている `@Inject` とは別の型である。Bean アーカイブがあっても、そのフィールドは普通のフィールドのまま残る。

### 4.6 2022 年、CDI 4.0。空の紙の意味が annotated に変わった

Jakarta EE 10 の CDI 4.0 は、仕様を Lite と Full に分けた。Lite はビルド時に配線を決める実行環境（Quarkus のような形）でも実装できるように、反射に依存する移植可能拡張を外したサブセットである。Full は従来の機能を含む。

ビルド時の発見と、実行時に全クラスを Bean にする `all` は相性が悪い。議論の結果、空の `beans.xml` の既定を `all` から `annotated` に変えた（仕様課題 jakartaee/cdi#500）。スキーマ 4.0 の `bean-discovery-mode` の既定値も `annotated` である。

これは破壊的変更である。空ファイルを置いて「印の無いクラスも `@Dependent` として注入する」ことに頼っていたアプリケーションは、CDI 4 以降ではそのクラスが Bean にならない。相手を注入しようとする側が Bean なら起動エラーになる。相手を `new` しているなら、前から null である。

Full の製品は、空ファイルを昔どおり明示アーカイブ（`all`）として扱うスイッチを持たなければならない。恒久の書き方は、ファイルに `bean-discovery-mode="all"` を明示することである。スイッチは移行用である。

Lite では、`beans.xml` から読むのは `bean-discovery-mode` だけである。`<alternatives>`、`<interceptors>`、`<decorators>`、`<scan>`、`<trim/>` は Lite の設定としては使われない。有効化は `@Priority` で行う。

### 4.7 2024 年から 2026 年。4.1 と 5.0 は発見ルールを維持した

CDI 4.1 は 2024-03-07 に Final となった。Jakarta EE 11（2025-06）の CDI である。EE との結合規則は CDI 仕様からプラットフォーム仕様へ移った。`beans_4_1.xsd` の実質的な変更は version の既定が 4.1 になったことである。発見モードの既定は `annotated` のままである。

Jakarta EE Web Profile 11 では、WAR の `beans.xml` の置き場所は次のどちらでもよい。

- `WEB-INF/beans.xml`
- `WEB-INF/classes/META-INF/beans.xml`

両方に置くと、移植できない動きになる。どちらか片方だけにする。

CDI 5.0 は Jakarta EE 12 向けで、2026-09-11 に Release Review が可決されている。最小 Java は 17。`@Reserve`、`@Eager`、`@AutoClose` などが追加された。`beans_5_0.xsd` は 4.1 と後方互換で、版数の更新が中心である。空ファイルの意味や、`@Inject` が Bean 定義アノテーションではないことは変わっていない。Maven 座標は `jakarta.cdi:jakarta.cdi-api` へ移り、5.0 では古い座標からのリロケーション成果物がある。

歴史を一文にすると、こうなる。

最初は「紙が無いと CDI は見ない」だった。紙の置き忘れが多かったので「特別な名札があれば見る」にした。見すぎて重く、ライブラリを壊したので名札を減らし、見ないスイッチを残した。ビルド時 CDI のために、空の紙の意味を「全部」から「名札のある机だけ」に変えた。`@Inject` は、どの時代でも名札ではない。

---

## 5. Bean 定義アノテーション

CDI 4.1 の本文と Weld のパッケージング説明を合わせた、クラスを Bean として発見するための印は次である。

| 印 | パッケージの例 | クラスに付くと annotated で Bean になるか | アーカイブを暗黙の Bean アーカイブにするか |
| --- | --- | --- | --- |
| `@ApplicationScoped` | `jakarta.enterprise.context` | する | する（EE で暗黙発見が有効なとき） |
| `@RequestScoped` | 同上 | する | する |
| `@SessionScoped` | 同上。通常スコープ | する | する |
| `@ConversationScoped` | 同上。CDI Full | する | する |
| 自作の `@NormalScope` | アプリのパッケージ | する | する |
| `@Dependent` | `jakarta.enterprise.context` | する | する |
| ステレオタイプ | `@Stereotype` が付いたアノテーション。中身が空でもよい | する | する |
| `@Model` | `jakarta.enterprise.inject`。`@Named` と `@RequestScoped` の組み込みステレオタイプ | する | する |
| `@Interceptor` | `jakarta.interceptor` | する（インターセプタとして） | する |
| `@Decorator` | `jakarta.decorator`。CDI Full | する（Weld は定義アノテーションに含める） | する |
| セッション Bean | `jakarta.ejb.Stateless` / `Stateful` / `Singleton` | EE では Bean として扱う | Web Profile 11 では、セッション Bean があると暗黙アーカイブになれる |

次は CDI の印に見えて、Bean 定義アノテーションではない。これしか無いアーカイブは、`beans.xml` も無ければ Bean アーカイブにならない。

| 印 | 実際の役割 |
| --- | --- |
| `@Inject` | 注入点。クラスを Bean にする印ではない |
| `@Produces` | プロデューサ。宣言しているクラス自身が Bean であることが先 |
| `@Disposes` | プロデューサの後始末 |
| `@Observes` / `@ObservesAsync` | イベントの受信。宣言クラスが Bean であることが先（Quarkus は例外。後述） |
| `@Named` | EL 名。CDI 1.2 以降、定義アノテーションではない |
| 自作 `@Qualifier` | 同じ型の Bean を区別する印 |
| `@Alternative` | 代替 Bean。選ばれるには `@Priority` か `beans.xml` の選択が別途必要。定義アノテーションではない |
| `@Specializes` | 特殊化。定義アノテーションではない |
| `@Priority` | 優先度。それ自体は定義アノテーションではない |
| `@Vetoed` | Bean にしない、という印 |
| `jakarta.inject.Singleton` | 疑似スコープ。仕様は「`@Dependent` 以外の疑似スコープは定義アノテーションではない」と書く |
| `@Transactional` | インターセプタバインディング。ステレオタイプでなければ定義アノテーションではない |
| JAX-RS の `@Path` | REST の印。CDI の定義アノテーションではない |

疑似スコープの注記は重要である。`@Scope` だけが付いた自作アノテーションは、`@NormalScope` ではない。`@Dependent` 以外は定義アノテーションにならない。

空のステレオタイプは定義アノテーションになる。スコープを書かないステレオタイプを付けたクラスは、`@Dependent` として発見される。

```java
@Stereotype
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface AppComponent {}
```

```java
@AppComponent
public class InvoiceService {
    @Inject
    Payment payment;
}
```

`InvoiceService` は annotated モードで Bean になる。`Payment` 側も Bean であることが必要である。

---

## 6. 処理の構成と、各段で指定できるもの

起動から注入までを段に分ける。各段で人間が指定できるものと、指定できる具体値を書く。

### 段 A. アーカイブの境界

指定するものは、クラスをどのアーカイブに入れるかである。

| モジュール | Bean のクラスの場所 | `beans.xml` の移植可能な場所 | Maven での置き場所 |
| --- | --- | --- | --- |
| JAR、EJB JAR | JAR のルート | `META-INF/beans.xml` | `src/main/resources/META-INF/beans.xml` |
| WAR のクラス | `WEB-INF/classes` | `WEB-INF/beans.xml` または `WEB-INF/classes/META-INF/beans.xml` の片方 | `src/main/webapp/WEB-INF/beans.xml` または `src/main/resources/META-INF/beans.xml` |
| WAR のライブラリ | `WEB-INF/lib/*.jar` の各 JAR | その JAR の `META-INF/beans.xml` | 依存 JAR 側の `src/main/resources/META-INF/beans.xml` |
| RAR | RAR 内 | `META-INF/beans.xml` | コネクタモジュールの resources |
| EAR | 各モジュールと `lib` の JAR を個別に見る | モジュールごとの上記 | EAR の `lib` に入れる JAR ごとに記述子 |
| クラスパスのディレクトリ | そのディレクトリ | `META-INF/beans.xml` | SE のクラスパス直下 |
| アプリケーションクライアント | クライアント JAR | `META-INF/beans.xml` | Web Profile 11 は、クライアント JAR の Bean アーカイブを必須にしていない |

WAR で両方に `beans.xml` を置くと非移植である。サーバによって、どちらを読むか、両方をどう合成するかが変わる。

`src/main/java` の横に XML を置いても、Maven はコンパイル出力へリソースとしてコピーしない。記述子は `resources` か `webapp` に置く。

### 段 B. 記述子の有無とモード

ファイル名は `beans.xml` 固定である。別名は仕様の記述子として読まれない。

`bean-discovery-mode` に書ける値は、スキーマ上ちょうど 3 つで、小文字である。

- `annotated`
- `all`
- `none`

`ALL` や `Annotated` は列挙に無い。製品が受けても、移植するなら小文字にする。

`version` はトークンで、XSD のパターンは `\.?[0-9]+(\.[0-9]+)*` である。実際に使う値は `1.0`、`1.1`、`2.0`、`3.0`、`4.0`、`4.1`、`5.0` である。4.1 のスキーマでは属性の既定が `4.1`、発見モードの既定が `annotated` である。

名前空間は版で分かれる。

| 仕様 | 名前空間 | スキーマの例 |
| --- | --- | --- |
| CDI 1.0 | `http://java.sun.com/xml/ns/javaee` | `beans_1_0.xsd` |
| CDI 1.1 / 2.0 | `http://xmlns.jcp.org/xml/ns/javaee` | `beans_1_1.xsd`、`beans_2_0.xsd` |
| CDI 3.0 以降 | `https://jakarta.ee/xml/ns/jakartaee` | `beans_3_0.xsd` から `beans_5_0.xsd` |

CDI 4.0 以降の、印のある型だけを見る最小の記述子は次である。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/beans_4_1.xsd"
       version="4.1"
       bean-discovery-mode="annotated">
</beans>
```

印の無いクラスも `@Dependent` として注入したいときの値は `all` である。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/beans_4_1.xsd"
       version="4.1"
       bean-discovery-mode="all">
</beans>
```

このアーカイブを CDI の対象から外す値は `none` である。ファイルはある。Bean アーカイブではない。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/beans_4_1.xsd"
       version="4.1"
       bean-discovery-mode="none">
</beans>
```

完全に空のファイル（0 バイト、または XML 要素が無い）は、CDI 4.0 以降では上の `annotated` と同じ扱いになり、CDI 3.0 までは `all` と同じ扱いだった。空ファイルは「何も書いていないから安全」ではない。版によって意味が反対側に動いた。

### 段 C. 発見後のフィルタ

`<scan>` は CDI Full で読まれる。子は `<exclude>` の繰り返しである。`<include>` はスキーマに無い。

`exclude` の `name` は必須で、XSD のパターンは次である。

```text
([a-zA-Z_$][a-zA-Z\d_$]*\.)*([a-zA-Z_$][a-zA-Z\d_$]*|\*|\*\*)
```

つまり、末尾だけが `*` または `**` になれる。`com.*.Foo` の形はスキーマに合わない。

| name の値 | 一致するもの |
| --- | --- |
| `com.example.shop.Cashier` | そのクラス |
| `com.example.shop.*` | `shop` パッケージ直下のクラス。`shop.internal` は含まない |
| `com.example.shop.**` | `shop` と、その配下のパッケージ |

条件要素は 0 個以上置ける。

| 要素 | 属性 | 意味 |
| --- | --- | --- |
| `if-class-available` | `name` 必須 | そのクラスをロードできるとき、この除外が有効 |
| `if-class-not-available` | `name` 必須 | そのクラスをロードできないとき、この除外が有効 |
| `if-system-property` | `name` 必須、`value` 任意 | `value` が無いときは、そのシステムプロパティが設定されていれば有効。`value` があるときは、値が一致すれば有効 |

条件を複数書いた exclude は、すべての条件が成り立つときに有効になる。条件が無ければ、名前が一致する型を除外する。

```xml
<scan>
    <exclude name="com.example.shop.internal.**"/>
    <exclude name="com.example.shop.faces.**">
        <if-class-not-available name="jakarta.faces.context.FacesContext"/>
    </exclude>
    <exclude name="com.example.shop.Mock**">
        <if-system-property name="app.env" value="prod"/>
    </exclude>
</scan>
```

Jakarta EE 8 以前の同じ条件は、クラス名が `javax.faces.context.FacesContext` である。

`<trim/>` は空要素である。XSD はテキストを空に固定している。属性は無い。明示アーカイブ（`all`）で使い、定義アノテーションもスコープアノテーションも無い型を、発見された型の集合から外す。`@Inject` だけのクラスはここで落ちる。`jakarta.inject.Singleton` はスコープアノテーションなので、trim の「スコープが無い」には当てはまらない。一方、annotated モードの定義アノテーションには当たらない。モードと trim で結果が分かれる。

### 段 D. アーカイブ単位で有効にする部品

CDI Full の `beans.xml` は、発見だけでなく「このアーカイブでどれを有効にするか」も書く。既定では、アーカイブに有効なインターセプタ、デコレータ、オルタナティブは無い。`@Priority` でアプリケーション全体に有効にしたものは別ルートである。

`<interceptors>` の子 `<class>` はインターセプタのクラス名。並びが呼び出し順で、先に書いたほうが先に呼ばれる。同じクラスを 2 回書くと配備エラー。

`<decorators>` も同様で、先に書いたほうが外側に近い呼び出し順になる。同じクラスの重複は配備エラー。

`<alternatives>` の子は `<class>` または `<stereotype>`。クラスは、代替の Bean クラス、またはプロデューサを宣言しているクラスの名前。ステレオタイプは `@Alternative` の付いたステレオタイプの名前。該当するクラスが無い、代替ではない、ステレオタイプが `@Alternative` ではない、重複している、は配備エラー。

`@Priority` の数値は `jakarta.interceptor.Interceptor.Priority` の定数を使う。

| 定数 | 値 |
| --- | --- |
| `PLATFORM_BEFORE` | 0 |
| `LIBRARY_BEFORE` | 1000 |
| `APPLICATION` | 2000 |
| `LIBRARY_AFTER` | 3000 |
| `PLATFORM_AFTER` | 4000 |

インターセプタは数値が小さいほうから先に呼ばれる。オルタナティブは数値の大きいほうが選ばれる。

Lite と Quarkus は、この 3 要素を `beans.xml` から読まない。Quarkus で代替を選ぶ例は、ビルド時設定 `quarkus.arc.selected-alternatives=com.example.shop.MockPayment` である。

### 段 E. クラス側で指定できるもの

Bean になったクラスで、注入に関係して指定できるものは次である。

| 場所 | 指定 | 具体例 |
| --- | --- | --- |
| 型 | スコープ | `@ApplicationScoped` |
| 型 | 限定子 | `@PaymentMethod("card")` |
| 型 | 名前 | `@Named("cashier")` または `@Named` で既定名 |
| 型 | 代替 | `@Alternative` と `@Priority(Interceptor.Priority.APPLICATION)` |
| 型 | 除外 | `@Vetoed` |
| パッケージ | パッケージごと除外 | `package-info.java` に `@Vetoed` |
| コンストラクタ | コンテナに呼ばせるコンストラクタは 1 つ | `@Inject public Cashier(Payment payment)` |
| フィールド | 注入 | `@Inject private Payment payment;` |
| フィールド | インスタンス選択 | `@Inject @Any Instance<Payment> payments;` |
| メソッド | イニシャライザ注入 | `@Inject void setPayment(Payment payment)` |
| メソッド | プロデューサ | `@Produces @RequestScoped Payment card()` |
| メソッド | オブザーバ | `void onPaid(@Observes Receipt receipt)` |

限定子アノテーション自体の定義は、`@Qualifier`、`@Retention(RUNTIME)`、`@Target` が必要である。メンバを持つ限定子は、注入点と Bean でメンバの値が一致する必要がある。`@Nonbinding` を付けたメンバは一致判定に使わない。

### 段 F. 実行時にインスタンスを受け取る経路

注入が実行される経路は限られる。

1. 別の Bean の注入点から、コンテナがこの Bean を要求する
2. `Instance<T>#get()`、`CDI.current().select(T.class).get()`
3. EL 名が付いていれば、ページの式から解決する
4. モジュールが Bean アーカイブなら、サーブレットなどの EE 部品へコンテナが注入する
5. SE のテストランナーや `SeContainer` がテストクラスへ注入する

次の経路では、アノテーションが正しくても注入されない。

- `new Cashier()`
- リフレクションの `getDeclaredConstructor().newInstance()` を自分で呼ぶ
- 別フレームワークが引数なしコンストラクタで作り、CDI に渡さない
- 静的フィールド初期化子の中で作る

---

## 7. パターン別の具体値

各パターンは「教室が当番表に載るか」と「載ったあと、その机が給食当番か」を分けて見る。

サンプルのクラスは共通である。

```java
package com.example.shop;

import jakarta.inject.Inject;

public class Cashier {
    @Inject
    Payment payment;

    public void checkout() {
        payment.charge();
    }
}
```

```java
package com.example.shop;

import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class CardPayment implements Payment {
    public void charge() { }
}
```

### パターン 1. 記述子が無く、定義アノテーションも無い

`Cashier` に `@Inject` だけがある。`CardPayment` のような定義アノテーションも、同じアーカイブに無い。`beans.xml` も無い。

Jakarta EE の既定では、このアーカイブは Bean アーカイブではない。`Cashier` をサーブレットが `new` しても、フィールドを CDI に渡しても、コンテナはこのアーカイブを担当していない。`payment` は null のまま。`WELD-001408` は出ない。

直すなら、次のどちらかにする。

- `Cashier` を注入される側にするなら、`@ApplicationScoped` か `@Dependent` を付ける
- 印の無いクラスも Bean にするなら、`bean-discovery-mode="all"` の `beans.xml` を、WAR なら `WEB-INF/beans.xml` に置く

`@Inject` を増やすだけではアーカイブにならない。

### パターン 2. 記述子は無いが、同じアーカイブに @ApplicationScoped がある

EE サーバで暗黙発見が有効なら、アーカイブは暗黙の Bean アーカイブ、モードは `annotated` になる。

`CardPayment` は Bean になる。`Cashier` は `@Inject` しか無いので Bean にならない。サーブレットが `CardPayment` を `@Inject` するなら、それは成功しうる。サーブレットが `Cashier` を `@Inject` するなら、依存が満たされず起動エラーになる。サーブレット自身のフィールドに `@Inject Cashier` があり、WAR が暗黙アーカイブなら、黙って null にはならず、配備が失敗する。

`Cashier` を Bean にする具体値は、クラスに `@Dependent` を付けることである。

### パターン 3. Weld Servlet（Tomcat や Jetty）で beans.xml が無い

Weld 7 リファレンス 19.3.5 は、`beans.xml` の無い暗黙 Bean アーカイブを Weld Servlet が発見しない、と書く。`@ApplicationScoped` があっても、記述子が無ければそのアーカイブは対象外である。

`@Inject` はサーブレット上でも Bean 上でも処理されない。起動ログに Weld の依存エラーが無いのに、フィールドが null になる。

直す具体値は、WAR の `WEB-INF/beans.xml` にパターン用の `annotated` または `all` を置くことである。Jandex をクラスパスに置くと走査が速くなる、と Weld は案内している。Jandex が無くても反射で走査する。Jandex は「記述子の代わり」ではない。

サーブレットコンテナの自動判定に失敗するときの初期化パラメタは、`org.jboss.weld.environment.container.class` で、値は次のいずれかである。

- `org.jboss.weld.environment.tomcat.TomcatContainer`
- `org.jboss.weld.environment.jetty.JettyContainer`
- `org.jboss.weld.environment.undertow.UndertowContainer`

### パターン 4. Weld SE の既定

Weld 7 リファレンス 19.4.7 では、`beans.xml` の無い暗黙アーカイブは既定で発見から外れる。クラスパス全体を見るには、次のどちらかを真にする。

- システムプロパティ `org.jboss.weld.se.scan.classpath.entries=true`
- システムプロパティ `jakarta.enterprise.inject.scan.implicit=true`

`Weld.property(...)` や `SeContainerInitializer.addProperty(...)` でも渡せる。

記述子が無い状態で、このプロパティも無いと、`@ApplicationScoped` があっても Weld SE はそのクラスパス項目を Bean アーカイブにしない。`@Inject` はきかない。

標準 API で発見そのものを止める書き方は次である。

```java
try (SeContainer container = SeContainerInitializer.newInstance()
        .disableDiscovery()
        .addBeanClasses(CardPayment.class, Cashier.class)
        .initialize()) {
    Cashier cashier = container.select(Cashier.class).get();
}
```

`disableDiscovery()` のあと `addBeanClasses` が無いと、コンテナは空に近い。`select` は依存を解決できない。`addPackages(Cashier.class)` はそのクラスのパッケージを加える。`addPackages(true, Cashier.class)` は再帰的にパッケージを加える。

Weld 独自に定義アノテーションを増やすメソッドは `Weld.addBeanDefiningAnnotations(...)` である。移植しない。

### パターン 5. bean-discovery-mode="none"

記述子はある。モードが `none` なので、仕様上このアーカイブは Bean アーカイブではない。定義アノテーションがあっても発見しない。`@Inject` は読まれない。

第三者ライブラリが CDI に巻き込まれるのを止めるときに使う。自分のアプリケーション WAR に `none` を置くと、アプリケーション自身の `@Inject` も止まる。

### パターン 6. 空の beans.xml を CDI 4 へ持ってきた

CDI 3.0 までの空ファイルは `all` だった。`Cashier` にスコープが無くても `@Dependent` Bean になり、コンテナが作れば `@Inject` は埋まる。

CDI 4.0 以降の空ファイルは `annotated` である。`Cashier` は Bean にならない。`Cashier` を注入している Bean があれば起動エラー。サーブレットが自分で `new Cashier()` しているなら null。

恒久の具体値は `bean-discovery-mode="all"` を書くことである。移行の間だけ昔の意味に戻すスイッチは、製品ごとに名前が違う。後の節に一覧がある。

### パターン 7. version="4.1" でモード属性を省略した

4.1 の XSD では `bean-discovery-mode` は省略可能で、既定は `annotated` である。省略は空ファイルと同じく「印のある型だけ」である。XSD の説明は、4.1 以降ではモードを明示するよう強く勧めている。

### パターン 8. all と trim

```xml
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/beans_4_1.xsd"
       version="4.1"
       bean-discovery-mode="all">
    <trim/>
</beans>
```

CDI Full では、全型について発見の観察が走る。Bean になるのは、定義アノテーションがある型と、何らかのスコープアノテーションがある型である。`@Inject` だけの `Cashier` は Bean にならない。移植可能拡張は `ProcessAnnotatedType` でそのクラスを観察できる。`annotated` モードでは、定義アノテーションの無い型に対してこのイベントは飛ばない。拡張が全クラスを見たいときに `all` と `trim` を組み合わせる、という経緯である。

Lite は `trim` を読まない。

### パターン 9. @Named だけ、または jakarta.inject.Singleton だけ

```java
@Named
public class Cashier {
    @Inject Payment payment;
}
```

```java
@Singleton // jakarta.inject.Singleton
public class Cashier {
    @Inject Payment payment;
}
```

annotated モードでは、どちらも Bean にならない。`beans.xml` が無ければアーカイブにもならないことがある。`jakarta.ejb.Singleton` は別物で、EE のセッション Bean である。

`all` なら、スコープ無しのクラスは `@Dependent` になる。`jakarta.inject.Singleton` は疑似スコープなので、`all` ではそのスコープの Bean として発見されうる。annotated では発見されない。この差で「サーバを上げたら急に null」が起きる。

### パターン 10. 拡張の JAR に beans.xml が無い

`META-INF/services/jakarta.enterprise.inject.spi.Extension` で拡張クラスを登録していても、`beans.xml` が無ければ、そのアーカイブは Bean アーカイブではない。拡張のコールバックは動く。同じ JAR の `@ApplicationScoped` サービスは発見されない。

拡張 JAR で Bean も置くなら、`META-INF/beans.xml` に `bean-discovery-mode="annotated"` を入れる。Build Compatible Extension（`jakarta.enterprise.inject.build.compatible.spi.BuildCompatibleExtension`）でも、記述子が無ければアーカイブではない、という同じ種類の規則がある。

### パターン 11. パッケージが javax のまま

```java
import javax.inject.Inject;
import javax.enterprise.context.ApplicationScoped;
```

このクラスを Jakarta EE 10 以降（CDI 4 以降）へデプロイすると、コンテナが見る定義アノテーションは `jakarta.enterprise.context.ApplicationScoped` である。`javax` 側の注釈は、別の型としてクラスファイルに残る。定義アノテーションが無いので annotated では Bean にならない。`javax.inject.Inject` は注入点にならない。記述子が `all` でも、コンテナは `jakarta.inject.Inject` の付いた場所を埋める。`javax.inject.Inject` のフィールドは普通のフィールドなので、null のまま残り、起動エラーにもならない。

直す値は import を `jakarta.inject.Inject` と `jakarta.enterprise.context.*` に変えることである。EE 8 以前へ `jakarta` の import を持っていくと、逆向きの同じ現象になる。

### パターン 12. exclude や @Vetoed で机を外した

アーカイブは存在する。`Cashier` が exclude の名前に一致する、または `@Vetoed` がある。Bean にならない。他の Bean が `Cashier` を注入するなら起動エラー。だれも注入せず、自分で `new` するなら null。

パッケージ全体を外す具体例は、`package-info.java` である。

```java
@Vetoed
package com.example.shop.internal;

import jakarta.enterprise.inject.Vetoed;
```

### パターン 13. アーカイブは正しいが new している

`beans.xml` は `annotated`、`Cashier` は `@ApplicationScoped`、`Payment` も Bean。サーブレットが次のように書く。

```java
Cashier cashier = new Cashier();
cashier.checkout();
```

`payment` は null である。`@PostConstruct` も走らない。正しい受け取り方の例は、サーブレットのフィールドである。

```java
public class CheckoutServlet extends HttpServlet {
    @Inject
    Cashier cashier;
}
```

このサーブレット注入が動く前提は、WAR が Bean アーカイブであることである。

### パターン 14. WAR の記述子が 2 箇所にある

`WEB-INF/beans.xml` が `annotated`、`WEB-INF/classes/META-INF/beans.xml` が `all`、のように両方ある。Web Profile 11 は、この状態を非移植と書く。サーバ A では `Cashier` が Bean になり、サーバ B ではならない、が起きうる。片方を削除する。

### パターン 15. WildFly が暗黙アーカイブを無視している

管理モデル（WildFly 38 のドキュメント、2026-08）の属性 `require-bean-descriptor` の既定は `false` である。`true` にすると、`beans.xml` の無い暗黙アーカイブを Weld が無視する。`@ApplicationScoped` があっても、その JAR は Bean アーカイブとして扱われない。`@Inject` はきかない。

サーバ全体:

```text
/subsystem=weld:write-attribute(name=require-bean-descriptor,value=true)
```

配備単位では、アプリケーションの `META-INF/jboss-all.xml` である。WAR の場合、アーカイブ直下の `META-INF` は `src/main/webapp/META-INF/jboss-all.xml` になる。`src/main/resources/META-INF` は `WEB-INF/classes/META-INF` であり、WAR 直下の `META-INF` とは別である。

```xml
<jboss xmlns="urn:jboss:1.0">
    <weld xmlns="urn:jboss:weld:1.0" require-bean-descriptor="true"/>
</jboss>
```

`false` に戻すと、記述子の無い暗黙アーカイブを再び見る。

### パターン 16. 空ファイルを all に戻す互換スイッチ

既定は、CDI 4 以降では戻さない（`annotated`）。

| 製品 | 範囲 | 設定名と、all に戻す値 |
| --- | --- | --- |
| Weld SE | そのコンテナ | `Weld.LEGACY_EMPTY_BEANS_XML_TREATMENT`。文字列は `org.jboss.weld.se.discovery.emptyBeansXmlModeAll`。値 `true` |
| Weld Servlet | その Web アプリ全体 | サーブレット初期化パラメタ `org.jboss.weld.environment.servlet.emptyBeansXmlModeAll` を `true`。発見より前に必要なので、Bean から設定することはできない |
| WildFly | サーバ。再起動が `all-services` | `/subsystem=weld:write-attribute(name=legacy-empty-beans-xml-treatment,value=true)`。既定 `false` |
| Open Liberty | サーバ設定。CDI 4.0 以降だけ意味がある | `<cdi emptyBeansXmlCDI3Compatibility="true"/>`。既定は false |
| Payara | 1 つの配備 | `asadmin deploy --properties emptyBeansXmlModeALL=true myapplication.war` |
| Payara | 構成全体 | `asadmin create-system-properties fish.payara.deployment.emptyBeansXmlModeALL=true` |

Payara のドキュメントは、Eclipse Transformer で変換した Jakarta EE 8 アプリケーションでは、このレガシーモードが既定で有効になる、と書いている。

このスイッチは空ファイルの解釈だけを変える。`bean-discovery-mode="annotated"` と明示したファイルは `annotated` のままである。

### パターン 17. Open Liberty が暗黙アーカイブを見ない

```xml
<cdi enableImplicitBeanArchives="false" emptyBeansXmlCDI3Compatibility="false"/>
```

`enableImplicitBeanArchives` の既定は `true` で、CDI 1.2 以降に適用される、と Open Liberty の設定リファレンスは書く。`false` にすると、記述子の無いアーカイブを走査しない。`beans.xml` が無い JAR の `@Inject` はきかない。

### パターン 18. OpenWebBeans が記述子の無い JAR を見ない

TomEE などが使う OpenWebBeans のプロパティファイルは `META-INF/openwebbeans/openwebbeans.properties` である。`configuration.ordinal` が大きいファイルが勝つ。実装の内部値は 1 から 100 で、自分のファイルの既定 ordinal も 100 である。

```properties
configuration.ordinal=101
org.apache.webbeans.scanBeansXmlOnly=true
```

`scanBeansXmlOnly` の既定は `false` である。`true` は「`META-INF/beans.xml` がある JAR だけを解析する」で、CDI 1.0 の走査に戻す、とプロジェクトの説明にある。`true` のとき、記述子の無いアーカイブの `@Inject` は読まれない。

関連して `org.apache.webbeans.scanExclusionPaths` は、既定スキャナが Bean 走査しない JAR やパスの一覧である。中身は `openwebbeans-impl` のプロパティに列がある。

### パターン 19. Payara / GlassFish 系で EAR の暗黙 CDI を止めた

`glassfish-application.xml` の `enable-implicit-cdi` の既定は `true` である。`false` にすると、EAR 内モジュールの暗黙走査を止める。

```xml
<glassfish-application>
    <enable-implicit-cdi>false</enable-implicit-cdi>
</glassfish-application>
```

WAR 単体で同じことをする移植可能な方法は、その WAR の `beans.xml` を `bean-discovery-mode="none"` にすることである。`none` は「この WAR も Bean アーカイブではない」なので、アプリケーション本体の `@Inject` も止まる。ライブラリだけ止めたいなら、ライブラリ JAR 側に `none` を置く。

古い Payara 4 / 5 のデプロイ例には `asadmin deploy --properties=implicitCdiEnabled=false myapp.war` がある。管理コンソールの Implicit CDI チェックは、記述子側の指定があると無視される、と Payara の説明にある。

### パターン 20. Quarkus は単一アーカイブで、依存の beans.xml の中身を見ない

Quarkus の CDI リファレンス（2026-01 時点のガイド）は、発見を次のように説明している。

- Bean アーカイブは 1 つだけで、発見モードは `annotated`
- そのアーカイブは、アプリケーションクラス、`beans.xml` がある依存、`META-INF/jandex.idx` がある依存、`quarkus.index-dependency` で指名した依存、Quarkus の統合コードから合成される
- 依存の `beans.xml` は、ファイルがあることが印で、中身は無視される
- Bean 定義アノテーションの無いクラスは発見されない
- ただしプロデューサメソッド、プロデューサフィールド、オブザーバは、宣言クラスに定義アノテーションが無くても発見し、そのクラスを `@Dependent` とみなす。これは仕様と違う、とガイドが明記している

依存 JAR に `@ApplicationScoped` と `@Inject` があっても、`beans.xml` も Jandex 索引も `quarkus.index-dependency` も無いと、合成アーカイブに入らない。アプリケーションがそれを注入すると、ビルドが失敗する。黙って null になるのは、そのクラスを `new` している場合である。

索引を依存側で作る例（ガイドに出ているプラグイン版）:

```xml
<plugin>
    <groupId>io.smallrye</groupId>
    <artifactId>jandex-maven-plugin</artifactId>
    <version>3.5.2</version>
    <executions>
        <execution>
            <id>make-index</id>
            <goals><goal>jandex</goal></goals>
        </execution>
    </executions>
</plugin>
```

依存を直せないときの具体値:

```properties
quarkus.index-dependency.mylib.group-id=com.example
quarkus.index-dependency.mylib.artifact-id=shop-lib
```

`classifier` は任意である。

未使用 Bean の除去はビルド時に行われる。注入している側が Bean として見えていないと、依存先は「使われていない」と判定されうる。設定 `quarkus.arc.remove-unused-beans` の値は `all`、`true`、`fwk`、`none` が案内されている。`fwk` はアプリケーションの Bean を残す側、`none` は除去しない。残したい型は `quarkus.arc.unremovable-types` に書く。クラスに `@io.quarkus.arc.Unremovable` を付ける方法もある。

`beans.xml` に `<alternatives>` を書いても、Quarkus はその中身を読まない。代替の選択は `@Priority` か `quarkus.arc.selected-alternatives` で行う。

### パターン 21. CDI SE で発見を切り、クラスを足していない

`SeContainerInitializer.disableDiscovery()` はクラスパス走査を止める。`addBeanClasses`、`addPackages`、有効化メソッドが空なら、`@Inject` の付いたクラスはコンテナに存在しない。

`select` する側のテストが JUnit 拡張（Weld JUnit、OpenWebBeans の `@Cdi` など）に注入を任せている場合、拡張の設定が `disableDiscovery = true` で `classes` が空だと、テストフィールドの `@Inject` も埋まらない。OpenWebBeans JUnit 5 の具体例は `@Cdi(disableDiscovery = true, classes = Cashier.class)` である。`classes` に依存先の `CardPayment.class` も含める。

### パターン 22. インターセプタを beans.xml に書き忘れ、Lite で読まれない

`@Inject` そのものが null になる現象とは違う。Bean は生成される。`@Transactional` などの横断処理だけが走らない。CDI Full で `@Priority` も `<interceptors><class>` も無いインターセプタは、アーカイブで有効になっていない。Lite では `<interceptors>` を書いても読まれないので、`@Priority(Interceptor.Priority.APPLICATION)` をインターセプタクラスに付ける。

「注入がきいていない」ように見える処理の一部は、注入は成功していて、インターセプタが有効になっていない、という段の違いである。

---

## 8. 症状の見分け

同じ「null」に見えて、段が違う。直し方も違う。

| 観察 | 段 | まず見ること |
| --- | --- | --- |
| 起動は成功する。`@Inject` フィールドが null。Weld の 001408 がログに無い | Bean アーカイブになっていない、注入点がコンテナの `@Inject` ではない、または `new` である | 展開した WAR/JAR の `beans.xml` の実パスと中身。import が `jakarta` か。インスタンスの生成箇所 |
| 起動が `WELD-001408` で落ちる | アーカイブはあり、注入点はある。相手の Bean が 0 個 | 相手クラスに定義アノテーションがあるか。相手の JAR に記述子があるか。exclude と `@Vetoed` と `@Alternative` の未選択 |
| 起動が `WELD-001409` で落ちる | 相手が 2 個以上 | 限定子を分けるか、オルタナティブを 1 つ選ぶ |
| Quarkus のビルドが未満足の注入で落ちる | 合成アーカイブに相手が入っていない | 依存の `beans.xml`、`jandex.idx`、`quarkus.index-dependency` |
| フィールドは埋まるが、期待した実装ではない | 発見は成功。選択が違う | `@Priority`、`<alternatives>`、Quarkus の `selected-alternatives`。Lite は `beans.xml` の alternatives を無視する |
| CDI 3 へ上げたあと、空の `beans.xml` のままで印の無いクラスだけ注入されない | 空ファイルの既定が `all` から `annotated` に変わった | `bean-discovery-mode="all"` を明示する |

展開後に見るコマンドのイメージは、アーカイブの中のパスを列挙することである。

```text
WEB-INF/beans.xml
WEB-INF/classes/META-INF/beans.xml
WEB-INF/classes/com/example/shop/Cashier.class
WEB-INF/lib/shop-lib.jar の中の META-INF/beans.xml
```

`beans.xml` が `src` 上にあっても、パッケージ成果物に入っていなければコンテナは見ない。テストの `src/test/resources/META-INF/beans.xml` は、テストクラスパスにだけ入り、WAR には入らない。

---

## 9. コンテナ別のつまみ一覧

値が boolean のものは、書ける具体値が `true` と `false` である。

### 仕様が全 Full 製品に求めるもの

| 目的 | 方向 | 備考 |
| --- | --- | --- |
| `beans.xml` が無いアーカイブを無視し、CDI 1.0 の参加方法に戻す | 製品ごとのオプション。名前は仕様が固定していない | CDI 1.1 からの互換要求 |
| 空の `beans.xml` を明示アーカイブ（`all`）とみなす | 製品ごとのオプション | CDI 4.0 からの互換要求。恒久対応はモードの明示 |

### WildFly の weld サブシステム

WildFly 38 の管理モデルで確認した属性である。変更は `all-services` の再起動を要する。

| 属性 | 型 | 既定 | true のときの動作 |
| --- | --- | --- | --- |
| `require-bean-descriptor` | BOOLEAN | `false` | 記述子の無い暗黙アーカイブを無視する |
| `legacy-empty-beans-xml-treatment` | BOOLEAN | `false` | 空の `beans.xml` を発見モード `all` とみなす |
| `non-portable-mode` | BOOLEAN | `false` | CDI 1.1 で厳しくなった拡張 SPI の検査を緩める。注入の有無そのもののスイッチではない |
| `development-mode` | BOOLEAN | `false` | Weld の開発用ツール。WildFly のこのモデルでは 5.0.0 以降非推奨で、現行サーバではサポートされない、と説明されている |
| `thread-pool-size` | INT | 無し（nillable） | 1 以上 2147483647 以下。配備をまたぐ Weld のスレッドプール。発見の成否ではない |

`non-portable-mode` を `true` にしても、Bean アーカイブが無いアーカイブの `@Inject` は読まれない。役割が違う。

### Weld のプロパティ

| キー | 既定 | 具体値 | 効果 |
| --- | --- | --- | --- |
| `org.jboss.weld.se.scan.classpath.entries` | 走査しない側 | `true` | SE で、記述子の無い暗黙アーカイブもクラスパスから発見する |
| `jakarta.enterprise.inject.scan.implicit` | 同上 | `true` | 上と同じ目的の CDI 側のプロパティ名として Weld が案内 |
| `org.jboss.weld.se.discovery.emptyBeansXmlModeAll` | `false` | `true` | SE で空 `beans.xml` を `all` とみなす |
| `org.jboss.weld.environment.servlet.emptyBeansXmlModeAll` | `false` | `true` | Servlet 統合で空 `beans.xml` を `all` とみなす。アプリ全体に効く |
| `org.jboss.weld.environment.servlet.archive.isolation` | 分離する（`true` が既定の動作） | `false` | `false` でフラット配備。全 `beans.xml` を 1 つのアーカイブに合成する。Weld 2.2 より前の動き。代替やインターセプタの有効範囲が変わる |
| `org.jboss.weld.se.archive.isolation` | 分離する | `false` | SE 版のフラット配備 |
| `org.jboss.weld.bootstrap.vetoTypesWithoutBeanDefiningAnnotation` | 空 | 正規表現 | `all` で発見した型のうち、名前が一致し、かつ定義アノテーションが無い型を veto する。`annotated` では、その型は最初から発見されない |
| `org.jboss.weld.bootstrap.unusedBeans.excludeType` | `ALL` | 正規表現、または `ALL`、`NONE` | 未使用 Bean 除去の対象外。`ALL` は除去しない側。`NONE` はパターンによる除外をしない |
| `org.jboss.weld.bootstrap.unusedBeans.excludeAnnotation` | `jakarta\\.ws\\.rs.*` | 正規表現 | そのアノテーションがある Bean を除去対象から外す |
| `org.jboss.weld.nonPortableMode` | `false` | `true` | 非移植モード |
| `org.jboss.weld.environment.container.class` | 自動検出 | 上記 Tomcat / Jetty / Undertow のクラス名 | Servlet コンテナの実装クラス |

フラット配備は「アーカイブが無い問題」を直すスイッチではない。記述子の選択をアプリ全体で混ぜるスイッチである。

### Open Liberty

`server.xml` の要素名は `cdi`。

| 属性 | 既定 | 適用 | 具体値 |
| --- | --- | --- | --- |
| `enableImplicitBeanArchives` | `true` | CDI 1.2 以降 | `false` で、記述子の無い暗黙アーカイブを走査しない |
| `emptyBeansXmlCDI3Compatibility` | `false` | CDI 4.0 以降 | `true` で、空 `beans.xml` を CDI 3.0 と同じ `all` にする |

### Payara

| 設定 | 具体値 |
| --- | --- |
| 配備プロパティ `emptyBeansXmlModeALL` | `true` で、その配備の空ファイルを `all` |
| システムプロパティ `fish.payara.deployment.emptyBeansXmlModeALL` | `true` で構成全体 |
| `glassfish-application.xml` の `enable-implicit-cdi` | `true` が既定。`false` で EAR の暗黙 CDI を止める |
| 開発モードの配備プロパティ `cdiDevModeEnabled` | `true`。注入をオンにするスイッチではない |
| 開発モードのシステムプロパティ `org.jboss.weld.development` | `true`。全アプリケーションの Weld 開発モード |

### Quarkus

ビルド時に決まる。実行中に `beans.xml` を置いても、すでに合成が終わっていれば効かない。

| 設定 | 具体値の形 |
| --- | --- |
| `quarkus.index-dependency.<名前>.group-id` | Maven の groupId |
| `quarkus.index-dependency.<名前>.artifact-id` | artifactId。ガイドでは省略可能とされている |
| `quarkus.index-dependency.<名前>.classifier` | 任意 |
| `quarkus.arc.remove-unused-beans` | `all`、`true`、`fwk`、`none` |
| `quarkus.arc.unremovable-types` | 型名やパターンの一覧 |
| `quarkus.arc.selected-alternatives` | 代替クラスの完全修飾名 |
| `quarkus.arc.exclude-types` | 発見から外す型 |

### 標準の SeContainerInitializer

移植可能なメソッドと、渡す値の意味は次である。

| メソッド | 渡すもの | 効果 |
| --- | --- | --- |
| `disableDiscovery()` | 引数なし | クラスパスの自動発見を止める |
| `addBeanClasses(Class<?>...)` | `Cashier.class` のようにクラスリテラル | そのクラスを Bean として加える |
| `addPackages(Class<?>...)` | そのクラスが属するパッケージ | パッケージを加える |
| `addPackages(boolean, Class<?>...)` | 第 1 引数 `true` で再帰 | サブパッケージも含める |
| `addPackages(Package...)` | `Package` | パッケージを加える |
| `addExtensions(...)` | 拡張のクラスまたはインスタンス | 移植可能拡張を加える |
| `enableInterceptors(Class<?>...)` | インターセプタクラス | 有効化する |
| `enableDecorators(Class<?>...)` | デコレータクラス | 有効化する |
| `selectAlternatives(Class<?>...)` | 代替のクラス | 選択する |
| `selectAlternativeStereotypes(Class<? extends Annotation>...)` | 代替ステレオタイプ | 選択する |
| `addProperty(String, Object)` | 実装のキーと値 | Weld の空ファイル互換などに使う |
| `setClassLoader(ClassLoader)` | 走査に使うローダ | 別ローダのクラスを見る |
| `initialize()` | なし | `SeContainer` を返す。try-with-resources で閉じる |

発見を止めたまま `addBeanClasses(Cashier.class)` だけにすると、`Cashier` の `@Inject Payment` の相手がコンテナに無い。`CardPayment.class` も加える。

---

## 10. 動作をもう一度、給食で確認する

1. ドアに紙（`beans.xml`）が無い。机の名札も「当番」（定義アノテーション）ではない。先生は教室に入らない。`@Inject` の付箋は読まれない。
2. 紙が `none` と書いてある。紙はあるが、「この教室は回らなくてよい」という意味である。やはり読まれない。
3. 紙が空である。2022 年より前の学校では「全部の机を回る」だった。2022 年以降の学校では「当番の名札の机だけ」である。付箋だけの机は回らない。
4. 紙が `all` である。付箋だけの机も当番になる。ただし先生が配膳した給食だけに牛乳が入る。自分で買ったパン（`new`）には入らない。
5. 先生が Tomcat に出張している（Weld Servlet）、または Java SE の保健室にいる（Weld SE の既定）。紙の無い教室は、名札があっても回らない。紙を貼る。
6. 付箋の文字が古い国語（`javax.inject.Inject`）で、先生は新しい国語（`jakarta.inject.Inject`）しか読まない。付箋はある。注文としては扱わない。

どの段で落ちたかを先に決めると、設定値は 1 つに絞れる。アーカイブが無いのに、限定子やインターセプタの優先度を調整しても、先生は教室に入らない。

---

## 11. 参照した一次情報

- Jakarta CDI 4.1 仕様（Final、2024-03-07）の Beans 定義アノテーション、Bean アーカイブ、空の `beans.xml` を明示アーカイブとして扱う互換オプション  
  https://jakarta.ee/specifications/cdi/4.1/jakarta-cdi-spec-4.1
- CDI 4.1 の `beans_4_1.xsd`（`bean-discovery-mode` の既定 `annotated`、列挙 `annotated` / `all` / `none`、`scan` / `exclude` / `trim` / `interceptors` / `decorators` / `alternatives`）
- Jakarta EE Web Profile 11 の Bean アーカイブ。WAR では `WEB-INF/beans.xml` または `WEB-INF/classes/META-INF/beans.xml`。両方は非移植  
  https://jakarta.ee/specifications/webprofile/11/jakarta-webprofile-spec-11.0
- CDI 4.0 の空 `beans.xml` を `annotated` に変える課題 jakartaee/cdi#500
- CDI 5.0 リリース記録。スキーマ変更は版数が中心で 4.1 と後方互換。Release Review は 2026-09-11  
  https://jakarta.ee/specifications/cdi/5.0/
- Weld 7.0.0.Final リファレンス。Weld Servlet は記述子の無い暗黙アーカイブを発見しない。Weld SE の `org.jboss.weld.se.scan.classpath.entries` と `jakarta.enterprise.inject.scan.implicit`。空ファイル互換キー  
  https://docs.jboss.org/weld/reference/latest/en-US/html_single/
- WildFly 38 weld サブシステム。`require-bean-descriptor`、`legacy-empty-beans-xml-treatment`、`non-portable-mode`、`thread-pool-size`  
  https://docs.wildfly.org/38/feature-pack/doc/reference/subsystem/weld/index.html
- Open Liberty の `cdi` 設定。`enableImplicitBeanArchives`、`emptyBeansXmlCDI3Compatibility`  
  https://openliberty.io/docs/latest/reference/config/cdi.html
- Payara の empty beans.xml レガシーモード。`emptyBeansXmlModeALL`、`fish.payara.deployment.emptyBeansXmlModeALL`
- Apache OpenWebBeans の `org.apache.webbeans.scanBeansXmlOnly`（既定 false）  
  https://openwebbeans.apache.org/owbconfig.html
- Quarkus CDI リファレンス。単一アーカイブ、モード `annotated`、依存の `beans.xml` は中身を無視、プロデューサ発見の仕様との差  
  https://quarkus.io/guides/cdi-reference
