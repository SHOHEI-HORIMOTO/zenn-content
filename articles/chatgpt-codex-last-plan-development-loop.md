---
title: "ChatGPTをPlanner、CodexをExecutorにして開発ループを回す"
emoji: "🔁"
type: "tech"
topics: ["chatgpt", "codex", "python", "unity", "ai"]
published: true
---

## はじめに

私は現在、Unity上でAIが自律的に動作・発話する3Dアバターを個人開発しています。

この開発では、ChatGPTに大まかな要望を伝え、ChatGPTが実装計画を作り、その計画をCodexへ渡して実装とテストを行う方法を採用しています。

単に「AIにコードを書いてもらう」のではありません。

```text
人間が目的を伝える
  ↓
ChatGPTが実装計画を作る
  ↓
last_plan.mdへ保存する
  ↓
Codexが実装する
  ↓
自動テストとUnity実機評価を行う
  ↓
評価結果を次の計画へ戻す
```

本記事では、この開発方法を採用した理由と、開発ループを制御するPythonスクリプトの仕組みを紹介します。

## なぜこの方法を選んだのか

AIコーディング環境にはCodex以外にも、AnthropicのClaude Codeなど有力な選択肢があります。機能やモデル性能だけを比べれば、用途によって別のサービスが適している場合もあると思います。

それでも今回ChatGPTとCodexを選んだ最大の理由は、個人開発の費用を予測しやすいことです。

私の構成では、ChatGPTとCodexの間をAPIで直接接続していません。ChatGPT Plus上で計画を作り、Markdownファイルを介してローカルのCodexへ渡しています。

ChatGPT Plusに含まれる利用枠の範囲であれば、OpenAI APIを呼び出すたびに発生する従量制のトークン料金を追加で支払わずに開発ループを回せます。

```text
APIで完全自動化する場合

Planner API → トークン課金
Executor API → トークン課金
Reviewer API → トークン課金
```

```text
今回の方法

ChatGPT Plus → 計画作成
Markdown     → 計画の受け渡し
Codex        → 実装・テスト
ローカル処理 → 評価・比較
```

もちろん「完全無料」という意味ではありません。ChatGPT Plusの月額料金は必要で、利用上限もあります。OpenAI APIや外部のレビューAPIを追加すれば、別途料金が発生します。料金や提供条件も将来変わる可能性があります。

正確には、**Plusの利用枠内なら、追加の従量制API料金を極力発生させずに運用できる**という点をメリットとして捉えています。

## 役割を分ける

この方法では、人間、ChatGPT、Codex、自動評価に異なる役割を持たせています。

|担当|役割|
|---|---|
|人間|目的、優先順位、見た目の品質を判断する|
|ChatGPT|曖昧な要望を実装可能な計画へ変換する|
|Codex|リポジトリを調査し、実装とテストを行う|
|自動評価|変更前後の結果を比較し、問題を検出する|
|Git|変更履歴と復帰点を管理する|

人間が実装方法を一行ずつ指定するのではなく、最終的に実現したい状態を伝えます。

例えば、アバター開発では次のような要望です。

> 複数のジェスチャーが指定した順序で実行されたことを、自動テストで確認できるようにしたい。

ChatGPTは、現在の実装状況や直前のテスト結果を踏まえ、この要望をCodexが実行できる単位に分解します。

## `last_plan.md`を作業指示書にする

ChatGPTが作った計画は、`last_plan.md`というMarkdownファイルに保存します。

実際には、次のような構造です。

```markdown
## Task Title

複合ジェスチャーシーケンスの自動評価を追加する

## Why This Task

単一ジェスチャーだけでなく、一連の動作順序を検証するため。

## Files To Touch

- Assets/Scripts/RuntimeScenarioRunner.cs
- orchestrator/evaluate_session.py
- scripts/tests/test_evaluate_session.py

## Codex Instruction

複数のジェスチャーが指定順序で実行されたことを、
Unity側の実行ログとPython側の評価処理で検証できるようにする。
既存の単一ジェスチャー評価との互換性を維持すること。

## Validation

- Pythonのユニットテストを実行する
- Unityシナリオの評価結果を確認する

## Abort Conditions

- 指定外の主要ファイルを変更する必要がある場合は停止する
```

これは単なるプロンプトではなく、AI向けの小さな作業指示書です。

特に重要なのは次の点です。

- 何を実現するか
- なぜ今それを行うのか
- 変更予定のファイル
- 維持すべき既存動作
- 完了を判断するテスト
- 作業を中止する条件

実装方法を細かく固定しすぎず、変更範囲と完了条件を明確にします。これにより、Codexの調査・実装能力を活かしつつ、意図しない大規模変更を抑えられます。

## `run_loop.py`は何をしているのか

この開発ループの中心にあるのが、自作した`run_loop.py`です。

このスクリプトはCodexを起動するだけではありません。Planner、Executor、テスト、Reviewer、Gitの間をつなぐオーケストレーターとして動きます。

プロジェクト固有の情報を除いて要約すると、次の処理を行っています。

1. Gitの状態や前回の評価結果からPlanner向け資料を生成する
2. `last_plan.md`の必須項目を検証する
3. 計画をCodex用プロンプトへ変換する
4. Codexをローカルリポジトリ上で実行する
5. 実行前後のGitワークツリーを比較する
6. 予定したファイルと実際に変更されたファイルを照合する
7. PythonテストとUnityテストを実行する
8. Unityの変更前と変更後を同じシナリオで評価する
9. 評価結果をReviewer用の入力へまとめる
10. コミット、ロールバック、手動確認のどれに進むか判定する
11. 必要に応じて次の計画候補を生成する
12. 同じ失敗が繰り返された場合はループを停止する

処理のイメージを擬似コードにすると、次のようになります。

```python
def run_loop():
    policy = load_policy()

    if baseline_first:
        baseline = run_unity_scenario()
        save(evaluate(baseline))

    planner_input = collect_context_from_git_and_reports()
    save("planner_input.md", planner_input)

    for iteration in range(max_iterations):
        plan = load_and_validate("last_plan.md")

        before = snapshot_worktree()
        codex_result = run_codex(build_prompt(plan))
        after = snapshot_worktree()

        changed_files = compare(before, after)
        check_expected_files(plan, changed_files)

        python_result = run_python_tests()
        unity_result = run_unity_tests()

        candidate = run_unity_scenario()
        comparison = compare_with_baseline(candidate)

        review = review_results(
            codex_result,
            changed_files,
            python_result,
            unity_result,
            comparison,
        )

        if review.can_accept:
            commit_when_allowed()
            break

        if review.must_rollback:
            rollback_when_allowed()

        if same_failure_repeated():
            stop_loop()

        create_next_plan_candidate(review)
```

実際のコードは3,000行を超えていますが、記事では内部の評価閾値やプロジェクト固有のルールを公開せず、責務が分かる粒度に抽象化しています。

## Codexはどのように呼び出しているか

`run_loop.py`は、計画をテンプレートへ埋め込んだ後、標準入力からCodexへ渡します。

概念的には次のようなコマンドです。

```powershell
codex exec --cd <PROJECT_ROOT> --sandbox <SANDBOX_MODE> --json -
```

Pythonでは`subprocess.run()`を使用し、終了コード、標準出力、標準エラーを回収します。

```python
proc = subprocess.run(
    command,
    input=codex_prompt,
    capture_output=True,
    text=True,
    cwd=project_root,
    timeout=timeout_seconds,
)
```

Codexの終了コードだけではなく、コンパイラエラーや例外を示す出力も確認します。AIが最後に「完了しました」と回答していても、機械的な検証結果を優先します。

## 変更予定ファイルと実際の差分を照合する

AIへリポジトリを変更させるとき、テスト成功と同じくらい重要なのが変更範囲の監視です。

ループ実行前後でGitワークツリーの状態とファイルハッシュを取得し、差分を比較しています。

```text
last_plan.mdで指定したファイル
  ├─ 実際に変更された → expected
  ├─ 変更されなかった → missing
  └─ 指定外が変更された → unexpected
```

指定外のファイルが変更された場合、内容が正しく見えても自動承認しません。Unityが自動生成する一部のファイルなどは、明示したポリシーに基づいて別扱いにします。

この仕組みにより、「テストを通すためにAIが想定外の場所まで変更した」という問題を検出できます。

## baselineとcandidateを比較する

Unityアバターのような実行時挙動を持つソフトウェアでは、コンパイル成功だけでは品質を判断できません。

そこで、変更前を`baseline`、変更後を`candidate`として、同じシナリオを実行します。

```powershell
python .\orchestrator\run_loop.py --run-unity-eval --baseline-first
```

評価対象には、例えば次のような情報があります。

- 意図したアクションが実行された割合
- 同じ動作の過剰な繰り返し
- 動作の失敗理由
- 移動や回転の異常
- 緊急停止の発生
- 変更前後の評価差

結果はJSONとMarkdownで保存し、`accept`、`review`などの判定材料にします。

数値評価が良くても、人間から見て動作が不自然な場合があります。そのため、最終的な見た目の確認は人間が担当します。

## 自動化に停止条件を入れる

AIによる改善ループは、回数を増やせば必ず良くなるわけではありません。同じ問題に対して似た修正を繰り返すこともあります。

そのため、次のような場合は自動的に停止します。

- 同じ失敗理由が連続した
- 変更なしの実行が連続した
- 手動レビューが必要な状態から改善しない
- 最大反復回数に到達した
- 必須テストが失敗した
- 計画に必要なセクションがない

停止は失敗ではなく、人間へ判断を戻すための安全機構です。

また、保護対象のブランチでは自動コミットや自動ロールバックを行わないようにしています。自動化の範囲を、ブランチとポリシーによって変えられる設計です。

## 実際に起きた失敗

この仕組みは、最初からきれいに動いたわけではありません。

開発中には次のような問題がありました。

- Unityシナリオがタイムアウトした
- Unityのコンパイルエラーを正常終了として扱いかけた
- 予定していない補助ファイルが変更された
- 自動評価スコアは高いのに、実機では歩行へ遷移しなかった
- 視線制御と別のコンポーネントが競合した
- 同じ失敗に対する修正が繰り返された

これらを受けて、コンパイラ出力の解析、変更ファイルの照合、変更前後の比較、反復停止条件などを追加しました。

重要だったのは、失敗を人間だけが読むログで終わらせず、次の計画に再利用できる構造化データとして残すことでした。

## 完全自動化しないことにも意味がある

現時点では、ChatGPTが作った計画を人間が確認し、`last_plan.md`へ渡す工程を残しています。

APIで接続すれば、計画生成から実装まで完全自動化できます。しかし、手動の受け渡しには次の利点があります。

- 追加のAPI利用料を抑えられる
- 実装前に計画と変更範囲を確認できる
- 大規模な変更を止められる
- 人間が目的を修正するタイミングを持てる
- AI同士が誤った前提を強化し続けるのを防げる

目標は人間をループから完全に排除することではなく、人間がコードの細部ではなく、目的と品質の判断へ集中できる状態を作ることです。

## この方法が向いているケース

この方式は、次のような個人開発や検証プロジェクトと相性がよいと思います。

- ChatGPT PlusとCodexをすでに利用している
- APIの従量課金を抑えたい
- ローカルで自動テストを実行できる
- 一度の変更を小さく分割できる
- AIの変更範囲をGitで監視したい
- 最終品質に人間の目視確認が必要

一方、大量のタスクを無人で常時処理したい場合や、複数人で承認フローを運用する場合は、API、CI/CD、専用のエージェント基盤を使った方が適している可能性があります。

## まとめ

この開発方法で重視しているのは、優れたプロンプトを一度だけ作ることではありません。

```text
要望
→ 計画
→ 実装
→ テスト
→ 評価
→ 次の計画
```

この循環を、失敗時の停止条件とGitによる変更管理を含めて再現可能にすることです。

ChatGPTをPlanner、CodexをExecutorとして役割分担し、Markdownを境界にすることで、追加の従量制API料金を抑えながら、個人でも継続可能なAI開発ループを構築できました。

今後は、Reviewerの精度向上、計画候補の安全な自動昇格、Unity上の見た目を評価する仕組みを改善していく予定です。
