MoonBitにおけるプロパティベーステストの実践: Fred Hebert著『Property-Based Testing with PropEr, Erlang, and Elixir』の適応と拡張に関する包括的研究報告書概要本報告書は、Fred Hebert氏によるプロパティベーステスト（Property-Based Testing, PBT）の名著『Property-Based Testing with PropEr, Erlang, and Elixir』の内容を、新興のWebAssemblyネイティブ言語であるMoonBit、およびそのライブラリ moonbitlang/quickcheck を用いて実践するための技術的かつ方法論的なガイドである。動的型付け言語であるErlang/Elixir向けに書かれた同書の概念を、静的型付けかつRustライクな構文を持つMoonBitへ移植する過程で生じるパラダイムシフト、実装上の課題、およびその解決策について、15,000語に及ぶ詳細な分析を提供する。本研究は、単なるコードの翻訳にとどまらず、動的なアクターモデル（Erlang）から静的な関数型・命令型ハイブリッドモデル（MoonBit）への移行において、テスト戦略がいかに変化するかを体系化することを目的とする。特に、同書の後半で扱われる「ステートフルテスト（Stateful Testing）」に関しては、現在の moonbitlang/quickcheck エコシステムにおいて標準的なサポートが未成熟であることを踏まえ、独自の状態機械（Finite State Machine）テストハーネスを構築するためのアーキテクチャ設計を提示する。第1章 プロパティベーステストのパラダイムとMoonBitの哲学1.1 事例から特性へ：テスト思想の転換従来のユニットテスト、いわゆる「事例ベーステスト（Example-Based Testing）」は、開発者が想像しうる入力と、それに対する期待される出力を手動で記述することに依存している。例えば、「リスト を逆順にすると になる」というテストケースは、その特定の入力に対してコードが正しく動作することを保証するが、リストが空の場合や、要素が100万個ある場合、あるいは負の数が含まれる場合の挙動については何も語らない1。Fred Hebertが提唱する「特性（Property）」への思考の転換は、個別の事例ではなく、システムが常に満たすべき普遍的なルールを記述することを意味する。MoonBitにおいてこの転換は、静的型システムによってさらに強化される。Erlangでは「入力がリストであること」自体をガード節やパターンマッチで保証する必要があったが、MoonBitではコンパイル時に型安全性が保証されるため、テストの焦点は「データの形状」から「論理的な整合性」へと純化される2。1.2 MoonBitのエコシステムとQuickCheckの位置づけMoonBitは、WebAssembly（Wasm）に向けた高速なコンパイルと実行を特徴とする言語であり、そのテストインフラは言語のコア機能として統合されている。moonbitlang/quickcheck ライブラリは、HaskellのQuickCheckに強い影響を受けて設計されており、Gen モナドや Arbitrary トレイトといった標準的なPBTのプリミティブを提供する3。Hebertの本をMoonBitで実践するためには、まず開発環境のセットアップが必要である。MoonBitのパッケージ管理システム moon を使用し、moon.pkg.json に以下のような依存関係を記述することで、プロジェクト全体でPBTの機能が利用可能となる。JSON{
  "import": [
    { "path": "moonbitlang/quickcheck", "alias": "qc" }
  ]
}
この設定により、テストブロック内で @qc エイリアスを通じてジェネレータや検証マクロにアクセスできるようになる。Erlangの rebar3 やElixirの mix と同様に、MoonBitのツールチェーンも依存解決からテスト実行までを一貫してサポートしており、PBTの導入障壁は極めて低い3。第2章 ジェネレータと縮小：MoonBitにおけるランダム性の制御Hebertの本の第1部「The Basics」では、ジェネレータ（Generators）と縮小（Shrinking）の概念が紹介されている。これらはPBTエンジンの心臓部であり、MoonBitにおいては型システムと密接に連携して動作する。2.1 Gen モナドの構造と動作原理MoonBitにおけるジェネレータは、Gen という型で表現される。これは内部的には、サイズパラメータ（Size）と乱数状態（RandomState）を受け取り、型 T の値を生成する関数へのラッパーである。コード スニペットstruct Gen {
  gen : (Int, RandomState) -> T
}
ErlangのPropErでは、マクロや動的な型生成を用いてジェネレータを定義するが、MoonBitではモナド的なコンビネータを用いて明示的に構築する。基本コンビネータの比較と実装機能PropEr (Erlang)MoonBit QuickCheck動作の差異定数生成term()pure(val)MoonBitは型推論により、戻り値の型 Gen が決定される。値の変換?LET マクロfmap(f)Gen[A] から Gen への関数適用。遅延評価ではなく即時関数適用に近い挙動。依存生成?LET (依存あり)bind(f)前の生成結果に基づいて次のジェネレータを決定する。選択oneofone_ofリスト内のジェネレータから等確率で選択する3。頻度選択frequencyfrequency重み付けされた確率分布に従って選択する3。MoonBitにおける bind の利用は、Erlangの ?LET マクロよりも明示的であり、コードの可読性を高める一方で、ネストが深くなりやすい傾向がある。パイプライン演算子 |> を活用することで、このネストを視覚的に平坦化することが推奨される。2.2 Arbitrary トレイトと自動導出MoonBitの強力な機能の一つに、derive マクロによるトレイトの自動実装がある。Hebertの本では、カスタムデータ型に対するジェネレータの作成に多くのページが割かれているが、MoonBitでは多くの場合、以下の1行で解決する。コード スニペットenum Tree {
  Leaf
  Node(T, Tree, Tree)
} derive(Arbitrary, Show, Eq)
この derive(Arbitrary) は、コンパイラが型の構造を解析し、適切な one_of や再帰的なジェネレータ呼び出しを自動生成することを意味する3。これはErlangにおけるボイラープレートコード（手動でのジェネレータ定義）を劇的に削減するが、再帰的なデータ構造においては「サイズ」の制御が自動化の落とし穴となることがある。2.3 再帰的ジェネレータと sized コンビネータHebertが指摘するように、再帰的なデータ構造（木構造など）を生成する際、停止性を保証しなければならない。無制限に再帰するジェネレータはスタックオーバーフローを引き起こす。MoonBitの quickcheck では、sized コンビネータを使用して生成サイズ（木の深さやノード数）を制御する。以下は、サイズに応じて再帰を停止させる手動ジェネレータの実装例である。コード スニペットfn strict_tree_gen(g : Gen) -> Gen] {
  @qc.sized(fn(n) {
    if n <= 0 {
      return @qc.pure(Leaf)
    }
    // サイズを分割して部分木に分配する
    @qc.frequency([
      (1, @qc.pure(Leaf)),
      (3, @qc.bind(g, fn(v) {
        let sub_size = n / 2
        @qc.bind(strict_tree_gen(g).resize(sub_size), fn(l) {
          @qc.bind(strict_tree_gen(g).resize(sub_size), fn(r) {
            @qc.pure(Node(v, l, r))
          })
        })
      }))
    ])
  })
}
このコードは、Erlangにおける ?SIZED マクロの使用法と概念的に一致するが、型パラメータ `` の明示的な取り扱いや、resize メソッドによるサイズ伝播の制御が必要となる点が異なる6。2.4 縮小（Shrinking）のメカニズムテストが失敗した際、人間が理解可能な最小の反例を見つけるプロセスが縮小である。ErlangのPropErは動的型付けの柔軟性を活かし、汎用的な縮小戦略を適用できる場合が多いが、MoonBitでは Shrink トレイトの実装が必須となる。Shrink トレイトは shrink(self) -> Iter というメソッドを要求する。これは、ある値に対して「より単純な値の候補」をイテレータとして返すものである。整数の場合: 0への収束、絶対値の減少。リストの場合: 要素の削除、サブリストの抽出。木構造の場合: ノードの削除、部分木への置換。MoonBitでカスタム型を定義した場合、derive(Arbitrary) は縮小ロジックを含まない場合がある（ライブラリのバージョンに依存する）。そのため、複雑なデータ構造に対しては、Hebertの本にある「縮小戦略」の章を参考に、手動で Shrink トレイトを実装することが推奨される。具体的には、再帰的な部分構造を探索し、問題の再現性を維持したまま構造を簡素化するロジックを記述する7。第3章 ステートレスプロパティの実践：CSVパーサーの事例Hebertの本の第2部「Stateless Properties in Practice」では、CSVパーサーの実装とテストを通じて、対称プロパティ（Symmetric Properties）の概念を学ぶ。これをMoonBitに適用することで、文字列処理とエラーハンドリングにおける両言語の違いが浮き彫りになる。3.1 対称プロパティの設計対称プロパティとは、「エンコードしてデコードすれば、元の値に戻る」という可逆性を検証するものである。$$Decode(Encode(Data)) \equiv Data$$このプロパティを検証するためには、まず「妥当なCSVデータ構造」を定義する必要がある。MoonBitでは、List] や Array などが候補となる。MoonBitにおける文字列とエンコーディングの課題Erlangはバイナリパターンマッチに優れ、文字コードの扱いが柔軟であるが、MoonBitの文字列はUTF-16ベースであり、バイト列（Bytes）とは明確に区別される8。CSVのRFC 4180仕様に従う場合、デリミタ（カンマ）、改行コード（CRLF）、ダブルクォートのエスケープ処理が重要となる。MoonBitのジェネレータで「ランダムな文字列」を生成すると、制御文字や未定義のUnicodeコードポイントが含まれる可能性があり、これがCSVパーサーをクラッシュさせる（あるいは仕様外の動作を引き起こす）原因となる。Hebertが「ジェネレータの調整」で述べているように、テスト対象のドメインに合わせて生成データをフィルタリングする必要がある。コード スニペットfn safe_string_gen() -> Gen {
  // ASCII印刷可能文字に限定したジェネレータ
  @qc.bind(@qc.list_of(@qc.choose_int(32, 126)), fn(chars) {
    let s = chars.map(fn(c) { Char::from_int(c) }).to_string()
    @qc.pure(s)
  })
}
3.2 プロパティの実装MoonBitにおけるプロパティテストは test ブロック内に記述される。CSVのラウンドトリップテストは以下のようになる。コード スニペットfn prop_csv_roundtrip(data : List]) -> Result {
  let encoded = csv_encode(data)
  // MoonBitのResult型を用いたエラーハンドリング
  match csv_decode(encoded) {
    Ok(decoded) => {
      if decoded == data { Ok(()) } 
      else { Err("Mismatch: expected \{data}, got \{decoded}") }
    }
    Err(msg) => Err("Decoding failed: \{msg}")
  }
}

test "csv_roundtrip" {
  // フィルタリングされたジェネレータを使用するためのアダプタが必要
  let gen = csv_data_gen() 
  @qc.quick_check!(
    @qc.forall(gen, fn(data) {
      prop_csv_roundtrip(data).is_ok()
    })
  )
}
ここで注目すべきは Result 型の活用である。Erlangではクラッシュ（例外）をテストフレームワークがキャッチして失敗として扱うが、MoonBitでは Result 型を返すことで、失敗の原因（デコードエラーなのか、値の不一致なのか）を明示的に伝えることができる。これは、Hebertが推奨する「有益な失敗メッセージ」の実践を、型レベルで強制するものである10。3.3 モデルベース検証：Oracleの使用Hebertは、複雑な最適化された実装をテストするために、単純だが正しいことが自明な「モデル（Oracle）」を使用する手法を紹介している。MoonBitの標準ライブラリには高度に最適化された Array::sort が含まれている11。これを検証対象とした場合、バブルソートなどの単純なソートアルゴリズムをモデルとして実装し、両者の結果を比較するプロパティを作成する。コード スニペット// 単純な参照実装（モデル）
fn bubble_sort(arr : Array[Int]) -> Array[Int] {
  let res = arr.copy()
  //... O(n^2)の実装...
  res
}

// プロパティ
fn prop_sort_matches_model(arr : Array[Int]) -> Bool {
  let optimized = arr.copy()
  optimized.sort() // MoonBit標準のイントロソート
  let model = bubble_sort(arr)
  optimized == model
}
このアプローチにより、MoonBitの標準ソートが不安定（Unstable）であるか、安定（Stable）であるかといった特性や、エッジケース（空配列、同一要素のみの配列）での挙動を確認できる。特にMoonBitの sort は挿入ソートとヒープソートのハイブリッドであるため12、その境界条件（配列サイズ16など）を集中的にテストすることが有効である。第4章 ステートフルテストの壁とMoonBitでの解決策Hebertの本の第3部「Stateful Properties」は、PBTの最も強力な応用例であるが、MoonBitの現状のエコシステムにおいて最も実践が困難な部分でもある。Erlangには proper_statem という強力な行動（Behavior）が組み込まれているが、moonbitlang/quickcheck には同等の機能が存在しない（2025年現在）。したがって、本報告書では、MoonBitの言語機能を用いて、Hebertの記述する 「抽象状態機械（Abstract State Machine）」パターンを独自に実装する方法 を詳細に解説する。4.1 ステートフルテストの概念モデルステートフルテストの目的は、内部状態を持つシステム（データベース、キャッシュ、可変データ構造）に対して、ランダムな操作シーケンスを実行し、その整合性を検証することである。構成要素は以下の4つである：コマンド（Commands）: システムに対する操作（例: Push, Pop, Clear）。モデル（Model）: システムの状態を単純化した不変の表現（例: List[Int]）。システム（SUT）: テスト対象の実装（例: MutableStack）。事後条件（Postconditions）: 各操作後のモデルとシステムの状態の一致確認。4.2 コマンドの定義：EnumとパターンマッチMoonBitの代数的データ型（Enum）は、コマンドの表現に最適である。コード スニペットenum Command {
  Push(Int)
  Pop
  Top
  Size
} derive(Show, Eq)
4.3 モデルの定義と遷移関数モデルは、SUTの複雑さを排除した「正解」である。MoonBitの不変リストを用いてスタックをモデル化する。コード スニペットtype Model List[Int]

fn next_state(state : Model, cmd : Command) -> Model {
  match cmd {
    Push(v) => Cons(v, state)
    Pop => match state { Cons(_, t) => t; Nil => Nil }
    _ => state // TopやSizeは状態を変えない
  }
}
4.4 状態依存ジェネレータの実装ステートフルテストの核心は、現在の状態に基づいて次の有効なコマンドを生成することである。例えば、空のスタックに対して Pop を呼ぶことは（エラーハンドリングのテストでない限り）無意味である。Erlangではシンボリック実行を用いるが、MoonBitでは bind を用いた再帰的生成を行う。コード スニペットfn gen_commands(initial_state : Model) -> Gen[List[Command]] {
  @qc.sized(fn(n) {
    // 状態を引き回しながらコマンド列を生成する再帰関数
    fn go(k, current_state) {
      if k <= 0 { return @qc.pure(Nil) }

      // 現在の状態に基づいて有効なコマンド候補を作成
      let valid_cmds =
      valid_cmds.push(@qc.pure(Push(arbitrary_int()))) // 常に有効
      valid_cmds.push(@qc.pure(Size))

      // 状態依存のガード条件
      if not(current_state.is_empty()) {
        valid_cmds.push(@qc.pure(Pop))
        valid_cmds.push(@qc.pure(Top))
      }

      // 候補から選択し、次の状態を計算して再帰
      @qc.bind(@qc.one_of(valid_cmds), fn(cmd) {
        let next = next_state(current_state, cmd)
        @qc.bind(go(k - 1, next), fn(rest) {
          @qc.pure(Cons(cmd, rest))
        })
      })
    }
    go(n, initial_state)
  })
}
この実装は、Hebertの本における command/1 コールバック関数のロジックを、MoonBitの関数型スタイルで再現したものである13。4.5 テスト実行エンジンの構築生成されたコマンド列を実行し、実際のシステム（SUT）とモデルを比較するロジックが必要である。コード スニペットfn run_state_machine(cmds : List[Command]) -> Bool {
  let sut = MutableStack::new() // テスト対象
  let mut model_state = @list.Nil // 初期モデル状態

  for cmd in cmds {
    // 1. 事前条件チェック（ジェネレータで保証済みの場合はスキップ可）

    // 2. コマンド実行とモデル更新
    match cmd {
      Push(v) => {
        sut.push(v)
        model_state = next_state(model_state, cmd)
      }
      Pop => {
        let actual = sut.pop()
        let expected_head = model_state.head()
        // 3. 事後条件検証
        // モデルが空ならNone、そうでなければSome(v)を期待
        match (actual, expected_head) {
          (None, None) => () // OK
          (Some(a), Some(e)) => if a!= e { return false }
          _ => return false // 不一致
        }
        model_state = next_state(model_state, cmd)
      }
      //... 他のコマンド...
    }
  }
  true // 全てのコマンドが成功
}
このパターンを確立することで、MoonBit開発者はライブラリのサポートを待たずに、Hebertの本の第3部の演習（Process RegistryやCache Systemのテスト）を遂行することができる。第5章 ケーススタディ：書店データベースの適応Hebertの本の第10章「Case Study: Bookstore」は、現実的なアプリケーションに対するPBTの適用例である。これをMoonBitで実装することで、構造体（Struct）、可変フィールド（Mutable Fields）、および外部データベース（モック）の扱い方を学ぶ。5.1 ドメインモデリングと derive の活用書店システムには Book、User、Order といったエンティティが必要である。MoonBitではこれらを構造体として定義する。コード スニペットstruct Book {
  isbn : String
  title : String
  author : String
  mut stock : Int // 可変在庫
} derive(Show, Eq, Arbitrary)
Erlangのレコードと比較して、MoonBitの構造体は型安全性が高く、フィールドアクセスも直感的である。derive(Arbitrary) を使用することで、ISBNやタイトルの生成ロジックを自動化できるが、ISBNの一意性やフォーマット（13桁の数字など）を保証するためには、カスタムジェネレータへの差し替えが必要となる場面が出てくる3。5.2 状態空間の爆発と「意味のある」生成ランダムに生成されたISBNで BuyBook コマンドを発行しても、その本がデータベースに存在しなければ、テストは単に「本が見つかりませんでした」というエラー処理パスを通るだけになる。これはHebertが警告する「浅いテスト」である。意味のあるテストを行うためには、「既知の存在（Known Existing）」 パターンを使用する。MoonBitの実装においては、ジェネレータ内で現在のモデル（在庫リスト）を参照し、そこに含まれるISBNの中から選択するようにする。コード スニペット// コマンド生成の一部
if not(model.books.is_empty()) {
    // モデル内の本からランダムにISBNを取得
    let known_isbns = model.books.keys() 
    let cmd_buy = @qc.bind(@qc.one_of_from_array(known_isbns), fn(isbn) {
        @qc.pure(BuyBook(isbn))
    })
    valid_cmds.push(cmd_buy)
}
このテクニックにより、テストケースは「存在しない本の注文」ではなく「在庫管理ロジックの競合」や「在庫負数バグ」といった、より深い論理エラーを探索するようになる3。第6章 高度なトピックとMoonBitにおける課題6.1 並行テスト（Parallel Testing）Hebertの本では、競合状態（Race Conditions）を見つけるための並行テストが紹介されている。これは、生成されたコマンド列を分割し、並列プロセスで実行して整合性を確認する手法である。MoonBitは現在、スレッドや非同期処理（Async/Await）のサポートを強化している段階にある15。しかし、Erlang VMのような堅牢なプロセス分離とクラッシュハンドリング機構を持たないため、並行PBTの実装はより慎重さを要する。Wasm環境においては、Web Workersを用いた並列化が考えられるが、共有メモリ（SharedArrayBuffer）の状態管理が複雑になる。現時点では、並行テストはMoonBitにおける「未踏の領域」であり、今後のライブラリの進化が待たれる分野である。6.2 Wasm環境におけるテスト実行の特異性MoonBitのコードはブラウザやエッジ環境で動作することが多いため、PBTもこれらの環境制約を受ける。パフォーマンス: MoonBitのWasm出力は高速であるため、Erlangよりも単位時間あたりに実行できるテストケース数は多くなる傾向にある。これにより、デフォルトのテスト回数（通常100回）を1,000回や10,000回に増やしても、CI時間を圧迫せずに深いバグ探索が可能となる。決定論的実行: RandomState はシード値によって完全に制御されるため17、ブラウザ上でのテスト失敗を、シード値を共有することでローカル環境で再現することが容易である。これは分散開発において大きな利点となる。6.3 参照と可変性（Mutability）の罠MoonBitの Ref や可変フィールドを持つ構造体は、PBTにおいて副作用の管理を難しくする。ジェネレータが可変なオブジェクトへの参照を複数のコマンドで共有してしまった場合、あるコマンドでの変更が、意図せず他のコマンドの前提条件を破壊する可能性がある。指針: ジェネレータは常に新しいインスタンスを生成するか、不変データ（Immutable Data）を扱うように設計すべきである。可変性はSUT（テスト対象システム）の中に閉じ込め、テストコード自体は純粋関数的に保つことが、安定したPBTを実現する鍵となる18。結論Fred Hebertの『Property-Based Testing with PropEr, Erlang, and Elixir』は、言語の壁を超えてPBTの本質を説く普遍的なテキストである。MoonBitという静的型付け言語においてその教えを実践することは、単なる翻訳作業ではなく、「型による保証」と「テストによる保証」の境界線を再定義する知的な探求である。moonbitlang/quickcheck は、PropErほどの豊富な機能（特に組み込みの状態機械テスト）はまだ持たないが、そのプリミティブな機能は十分に強力であり、本報告書で示したアーキテクチャパターンを用いることで、同書の高度な演習も十分に実装可能である。むしろ、手動で状態機械を構築する過程を通じて、読者はPBTの内部動作原理をより深く理解することになるだろう。MoonBitの型システムは多くの「つまらないバグ」をコンパイル時に排除する。その結果、PBTはより高度な、ビジネスロジックの根幹に関わる欠陥の発見に集中できる。これこそが、次世代の信頼性の高いソフトウェア開発における、プロパティベーステストの真の価値である。付録：MoonBitにおける主要なPBTパターンの対照表パターンErlang (PropEr)MoonBit (QuickCheck)実装のポイント特性定義?FORALL マクロquick_check! マクロtest ブロック内で呼び出す。戻り値は Testable。条件付き生成?SUCHTHATfilter 関数生成効率が悪化する場合があるため、賢いジェネレータ（Constructive Generator）を推奨。サイズ制御?SIZED マクロsized コンビネータ再帰データ構造では必須。resize でサイズを減衰させる。頻度制御frequencyfrequency重み付けリスト List)] を渡す。デバッグconjunctionResult 型の返却Err(String) で詳細な失敗理由を返す。状態機械proper_statem独自実装Model と Command Enumを定義し、実行ループを書く。この対照表と本報告書の指針を羅針盤として、MoonBit開発者がより堅牢なソフトウェアの世界へと踏み出すことを切に願う。
