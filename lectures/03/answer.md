# 第3回 解答

## 演習

- 第1回（素のLLM）とエージェントの回答の違い：
    - 第1回の `app.ask`：
    AIE2026の宿題にある「3レベル」について、次の3つの可能性を挙げています。
    1. 難易度の3段階：初級・中級・上級の課題。
    2. AI開発の3段階：API利用・既存モデルの活用・独自開発やファインチューニング。
    3. プロンプトの3段階：単純な質問・条件付きの指示・高度なプロンプト。
    ただし、どれが正しいのかは確認できておらず、推測に基づく回答です。
    - エージェント：`search_glossary` で用語集を検索してから答え、「基礎（講義内容の確認）・実装（自分でコードを書く）・発展（調査や試行）の3段階で、どれか1つに取り組めば完了、原則として採点なし」と、用語集の `homework-levels` と同じ内容を答えた。最後に（第1回）と関係する回も示した

- `--trace` で見えたツール呼び出し（ツール名と引数）：
モデルに見えているツール: 3 個

[search_glossary]
  講座の用語集から用語を検索し、定義と、どの回で扱うかを返す。
  
  AIエンジニアリングの用語や、この講座（AIE2026）に固有の言葉について
  答える前に使うこと。query には調べたい用語を1つだけ入れる（例: "LoRA"）。
  引数: {"query": {"type": "string"}}

[get_lecture_summary]
  指定した回（1始まりの整数）の講義の要点を返す。
  
  「第3回では何を学ぶ？」のように、回の内容や関係を聞かれたときに使う。
  引数: {"lecture": {"type": "integer"}}

[make_quiz]
  指定した回（第N回）の用語集から確認問題を1問作る。
  
  利用者が「第2回の確認問題を出して」「第N回の内容でクイズして」のように、
  特定の回の確認問題・クイズ・理解度チェックを求めたときに使う。
  lecture には回の番号（例：第2回なら 2）を入れる。
  問題（question）と答え（answer）を JSON 文字列で返す。
  引数: {"lecture": {"type": "integer"}}

- 1つの質問での LLM 呼び出し回数（`requests`）：
    どの質問も `requests=2`。1回目でモデルがツール呼び出しを決め、2回目でツールの結果を見て回答を作る。LoRA/QLoRA では2つの検索を1回のリクエストでまとめて呼んでいた（`tool_calls=2`）

- 追加したツールと、docstring に書いた説明：
    - `make_quiz(lecture: int) -> str`：指定した回の用語を用語集から `random.choice` で1つ選び、`{"question": "「○○」の意味は？", "answer": 定義}` を JSON 文字列で返す。用語がない回（第0回など）では「第N回の用語は用語集にありません。」を返す
    - docstring：「指定した回（第N回）の用語集から確認問題を1問作る。利用者が「第2回の確認問題を出して」「第N回の内容でクイズして」のように、特定の回の確認問題・クイズ・理解度チェックを求めたときに使う。lecture には回の番号（例：第2回なら 2）を入れる。問題（question）と答え（answer）を JSON 文字列で返す。」
    - 結果：`make_quiz({'lecture': 2})` が呼ばれ、「エポック」の問題が出た。`uv run pytest -q` は 11 passed

## 宿題

- 取り組んだレベル：発展(a) ADKで作り直す
- やったこと：
    - `app/` は変えず、`adk_agent/glossary_agent/__init__.py`（`from . import agent`）と `adk_agent/glossary_agent/agent.py` を作った。`agent.py` では `Agent(model="gemini-3.5-flash-lite", name="glossary_agent", instruction=INSTRUCTIONS, tools=[tools.search_glossary, tools.get_lecture_summary, tools.make_quiz])` とした
    - `PYTHONPATH=. uv run --with google-adk adk run adk_agent/glossary_agent` で動かし、演習と同じ3つの質問を Pydantic AI 版と ADK 版の両方に投げて比べた
- 結果：
    - 書き方の違い：
    - 指示：Pydantic AI は `instructions=`、ADK は `instruction=`
    - ツール：Pydantic AI はエージェントを作った後に `agent.tool_plain(...)` で1つずつ登録、ADK はコンストラクタで `tools=[...]` とリストで渡す
    - モデル：Pydantic AI は実行時に `run_sync(..., model=...)` で渡す（テストで TestModel に差し替えられる）、ADK は定義の中で固定
    - ADK は `name` が必須。実行は `adk run` が対話ループを用意してくれるので CLI を書かなくてよい
    - どちらも関数の docstring と型ヒントからツールの説明を作るので、`tools.py` を変えずに使えた
　- 回答の違い：
    - 3レベル：内容・構成はほぼ同じで、どちらも最後に（第1回）。Pydantic AI 版は「興味や理解度に合わせて選んで」という一文を追加していた
    - LoRA/QLoRA：内容はほぼ同じ。ADK 版は PEFT（Parameter-Efficient Fine-Tuning）や量子化（Quantization）の正式名を補っていた。どちらも最後に（第2回）
    - 確認問題：Pydantic AI 版は「エポック」、ADK 版は「量子化」。ADK 版の問題も「「○○」の意味は？」という make_quiz の返り値と同じ形なので、make_quiz が呼ばれたと考えられる。用語が違うのは `random.choice` のためで、フレームワークの違いではない
- 気づき・疑問：
  - モデル・指示・ツールが同じなら、フレームワークを変えても回答の中身はほとんど変わらなかった。違いは言い回しや補足の程度
  - ADK の `adk run` は会話を覚えている。確認問題に "does it mean to tokenize?" と答えると、直前の問題が「量子化」だったことを踏まえて「トークナイズとは異なる概念」と説明した。`app.agent` は1問答えると終了するので、答え合わせはできない
  - 英語で答えても日本語で返ってきた。INSTRUCTIONS が日本語だからだと思われる
  - `adk run` ではツール呼び出しが見えず、回答の内容から推測するしかなかった。Pydantic AI 版は `--trace` で確認できたので、自分で書いた CLI の方が中で何が起きているかを確かめやすい
  - 疑問：ADK でツール呼び出しを確認するには、`adk web` やログ（`/tmp/agents_log/`）を見ればよいのか