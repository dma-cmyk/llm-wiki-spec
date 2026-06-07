# 関連メモリ自動リンク埋め込み仕様 (Auto-Relations Embedder)

本ドキュメントは、**Venus Memory Vault (LLM Wiki)** において、カード間の関係性（`relations`）を本文末尾にWikiリンクとして自動埋め込みし、Web UI側の表示とスマートに協調させる仕組みを他の環境でも再現するための仕様書です。

---

## 💠 1. 背景と目的
従来、カード間のリレーション（`relations`）はJSON上のメタデータとして保持され、Web UI側でグレーのリストボタンとして自動生成されていました。
しかし、これには以下の課題がありました。
1. **ポータビリティの欠如**: ローカルのマークダウンファイルを直接VS Codeなどのエディタで閲覧した際、関連カードへのリンクが表示されず、情報が孤立する。
2. **デザインの統一性の欠如**: Web UIのマークダウンパーサーがレンダリングする美しい紫色のインラインリンクと、下部に独立して並ぶグレーのリストボタンのデザインが混在し、ノイズになる。

本仕様は、**「自動最適化スクリプトで本文末尾に関連リンクを自動追記する」**機能と、**「Web UI側で重複描画を自動的に防ぐ」**フォールバックを組み合わせることで、この課題をクリーンに解決します。

---

## 🛠️ 2. 再現のための実装手順

### ステップ 1: 最適化スクリプト (`optimize_graph.py`) の書き換え

自動最適化スクリプト内の `merge_relations` 関数を以下のように拡張し、最適化実行時に各カードの本文末尾を自動クリーンアップ・追記します。

#### 【前提条件】
* スクリプト先頭付近に `import re` が存在すること。
* `[[タイトル]]` を抽出するための `extract_wikilinks(text)` 関数が定義されていること。

#### 【移植コード】
```python
def merge_relations(
    db: list, new_relations: dict[str, set[str]]
) -> tuple[list, dict]:
    """
    既存のrelationsを維持しつつ、推論結果をマージする。
    また、各カードの content 末尾にある「### 関連するメモリ」セクションを自動更新する。
    """
    valid_ids = {card["id"] for card in db}
    id_to_card = {card["id"]: card for card in db}
    stats = {"added": 0, "cleaned": 0, "orphans_fixed": 0}

    for card in db:
        cid = card["id"]
        existing = set(card.get("relations", []))
        inferred = new_relations.get(cid, set())

        # 1. 存在しないIDへの参照を除去（ゴミ掃除）
        cleaned = existing & valid_ids
        stats["cleaned"] += len(existing) - len(cleaned)

        # 2. 推論結果をマージ（既存 + 新規、自分自身は除外）
        merged = (cleaned | inferred) - {cid}

        # 追加されたリンクをカウント
        newly_added = merged - cleaned
        stats["added"] += len(newly_added)

        # 孤立ノード（リレーションゼロ）を検知
        if len(existing) == 0 and len(merged) > 0:
            stats["orphans_fixed"] += 1

        card["relations"] = sorted(list(merged))

        # ── 3. 本文末尾の「関連するメモリ」セクションの自動更新 ──
        content = card.get("content", "")
        
        # 過去に手動または別仕様で作られた類似セクション（関連Wikiカード等）を一括クリーンアップ
        cleaned_content = re.sub(
            r"\s*### (?:🔗\s*|💠\s*)?(?:関連Wikiカード|関連するメモリ|関連カード|関連リンク)[\s\S]*$",
            "",
            content
        ).strip()

        # 既に本文中に存在する明示的/暗黙的なWikiリンクを取得
        already_linked_titles = set(extract_wikilinks(cleaned_content))

        # 本文中にまだ一度もリンクとして登場していない関連カードのタイトルだけを抽出
        rel_titles = []
        for rel_id in merged:
            rel_card = id_to_card.get(rel_id)
            if rel_card:
                title = rel_card.get("title")
                if title and title not in already_linked_titles:
                    rel_titles.append(title)

        # 関連するカードがあれば、末尾に「### 関連するメモリ」としてWikiリンクをリスト追記
        if rel_titles:
            links = [f"- [[{t}]]" for t in sorted(rel_titles)]
            links_block = "\n\n### 関連するメモリ\n" + "\n".join(links)
            card["content"] = cleaned_content + links_block
        else:
            card["content"] = cleaned_content

    return db, stats
```

---

### ステップ 2: Web UI 側 (`src/app/page.tsx`) のレンダリング条件の変更

マークダウン内に自動埋め込みされたリンクと、UIが描画する下部関連カードUI（`filter-item`）の重複（2重表示）を防ぐため、UI側のNext.jsコードのレンダリング条件式に `!content.includes` の条件を追加します。

#### 【移植コード】
`src/app/page.tsx` 内の関連カード描画ブロックを探索し、条件式を以下のように書き換えます。

* **Before**:
  ```tsx
  {activeCard.relations && activeCard.relations.length > 0 && (
    <div style={{ marginTop: '20px' }}>
      <h4 className="form-label" style={{ marginBottom: '8px' }}>関連するメモリ</h4>
      {/* 関連カードのループ描画 */}
    </div>
  )}
  ```

* **After**:
  ```tsx
  {activeCard.relations && activeCard.relations.length > 0 && !activeCard.content.includes('### 関連するメモリ') && (
    <div style={{ marginTop: '20px' }}>
      <h4 className="form-label" style={{ marginBottom: '8px' }}>関連するメモリ</h4>
      {/* 関連カードのループ描画 */}
    </div>
  )}
  ```

---

## 🚀 3. 再現性の検証と運用方法
移植完了後、他の環境で以下の手順を実行し、挙動を確認します。

1. **最適化の実行**:
   ```bash
   python3 ~/.gemini/skills/venus-memory-vault/scripts/llm_wiki.py optimize --sync
   ```
2. **確認項目**:
   - データベースJSON上のカード `content` 末尾に、自動的に `### 関連するメモリ` セクションが作成され、双方向のWikiリンク（`[[タイトル]]`）が追加されていること。
   - `### 🔗 関連Wikiカード` などの古い類似セクションが存在していたカードについては、古い見出しが削除され、新しい `### 関連するメモリ` に一本化されていること。
   - Web UIを表示した際、マークダウン本文の中に関連リンクが表示され、画面下部にはグレーの重複リンクUIが表示されていないこと。

---

## 🤖 4. AIへの指示用コピペプロンプト (AI Prompts)
他の環境でこの仕様を再現する際、作業を担当するAIアシスタントに以下のテキストをそのままコピペして渡してください。

```markdown
### 指示
Venus Memory Vault (LLM Wiki) に、関連メモリを本文末尾に自動で埋め込む機能を追加し、UI側の2重表示を防止してください。
仕様書（AUTO_RELATIONS_SPEC.md）に記載されている以下の修正を行ってください：

1. `~/.gemini/skills/venus-memory-vault/scripts/optimize_graph.py` の `merge_relations` 関数を仕様書のコードで丸ごと置き換えてください。
   * 前提条件として、ファイル先頭付近で `import re` が行われていること、および `extract_wikilinks` 関数が定義されていることを確認してください。もし無ければ追加してください。
2. `~/プロジェクト/venus-memory-vault-ui/src/app/page.tsx` 内の関連カード描画条件式に `!activeCard.content.includes('### 関連するメモリ')` を追加し、重複表示を防止してください。

修正完了後、`llm_wiki.py optimize --sync` を実行してエラーなく動作し、カード末尾に関連リンクが綺麗に埋め込まれ、UI側のグレーのボタンリストが重複表示されなくなったことを確認してください。
```
