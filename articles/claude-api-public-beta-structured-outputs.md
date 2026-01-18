---
title: "Claude APIのStructured Outputs(パブリックベータ)が出たので実装を見てみる"
emoji: "🧱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Claude", "Anthropic", "TypeScript", "Python", "AI"]
published: true
---

## はじめに

2025年11月、AnthropicがついにClaude APIにおける Structured Outputs（構造化データ出力） のパブリックベータを発表しました。（APIサポート開始日：2025年11月13日）

https://x.com/claudeai/status/1989410368192668037

https://platform.claude.com/docs/en/build-with-claude/structured-outputs

現在は以下のモデルのみ対応しているようです。
- Claude Sonnet 4.5
- Claude Opus 4.1
- Claude Opus 4.5
- Claude Haiku 4.5

これは従来の「Tool Use（Function Calling）」を利用した擬似的な構造化ではなく、OpenAIの `strict: true` と同様に、APIレベルでJSONスキーマへの準拠を保証する専用機能です。

本記事では、APIの仕様を確認しつつ、Python/TypeScript SDKがこの機能をどのように実装しているのか、ソースコードレベルで深掘りしてみます。

## API仕様の変更点

従来のMessages APIに対し、以下の変更が加わっています。

1.  `anthropic-beta` ヘッダーに指定できる値の種類の追加
2.  `output_format` フィールドの追加

`curl` で叩く場合は以下のようになります。

```bash
curl https://api.anthropic.com/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: structured-outputs-2025-11-13" \
  -d '{
    "model": "claude-sonnet-4-5",
    "max_tokens": 1024,
    "messages": [
      {
        "role": "user",
        "content": "Extract the key information..."
      }
    ],
    "output_format": {
      "type": "json_schema",
      "schema": {
        "type": "object",
        "properties": {
          "name": {"type": "string"},
          "email": {"type": "string"},
          "plan_interest": {"type": "string"},
          "demo_requested": {"type": "boolean"}
        },
        "required": ["name", "email", "plan_interest", "demo_requested"],
        "additionalProperties": false
      }
    }
  }'

```

## SDKの対応状況

パブリックベータ機能（`output_format` パラメータや `anthropic-beta` ヘッダー）に対応したSDKのバージョンは以下の通りです。

| 言語 | パッケージ名 | 対応バージョン | リリース日 | リンク |
| --- | --- | --- | --- | --- |
| Python | `anthropic` | v0.73.0 | 2025/11/14 | [Release Note](https://github.com/anthropics/anthropic-sdk-python/releases/tag/v0.73.0) |
| TypeScript | `@anthropic-ai/sdk` | v0.69.0 | 2025/11/14 | [Release Note](https://github.com/anthropics/anthropic-sdk-typescript/releases/tag/sdk-v0.69.0) |

---

## TypeScript SDKの実装詳細

ここからは、実際にSDKのコードを追いながら実装の詳細を確認します。

### 基本的な使い方

`output_format` には JSON Schema を直接渡すこともできますが、Zodスキーマを利用できるヘルパー関数 `betaZodOutputFormat` が提供されています。

```typescript: example.ts
import Anthropic from '@anthropic-ai/sdk';
import { z } from 'zod';
import { betaZodOutputFormat } from '@anthropic-ai/sdk/helpers/beta/zod';

const ContactInfoSchema = z.object({
  name: z.string(),
  email: z.string(),
  plan_interest: z.string(),
  demo_requested: z.boolean(),
});

const client = new Anthropic();

// beta名前空間を利用することに注意
const response = await client.beta.messages.parse({
  model: "claude-sonnet-4-5",
  max_tokens: 1024,
  // 明示的なベータヘッダーの指定
  betas: ["structured-outputs-2025-11-13"],
  messages: [
    {
      role: "user",
      content: "Extract the key information from this email..."
    }
  ],
  // Zodスキーマを渡す
  output_format: betaZodOutputFormat(ContactInfoSchema),
});

// 自動的にパース・バリデーションされた結果が返る
console.log(response.parsed_output);

```

### ⚠️ ハマりポイント：Zod v4への依存

このSDKを試しててつまづいたポイントがあります。zodを利用可能なjsonスキーマへ変換する関数である `betaZodOutputFormat` を利用するには、プロジェクトのZodバージョンを v4.0.0 以上にする必要があります。

理由は、SDK内部で利用されている `toJSONSchema()` というメソッドが、Zod v4で標準化されたAPI（またはそれに準拠した実装）だからです。

https://github.com/anthropics/anthropic-sdk-typescript/blob/e6562d72502030e6cf90a31192b21b23c0b03422/src/helpers/beta/zod.ts#L20
https://zod.dev/v4?id=json-schema-conversion

もし v3系を利用している場合は、`zod-to-json-schema` 等のライブラリを使って手動で JSON Schema に変換し、`output_format` に生のオブジェクトとして渡す必要があります。

### SDK内部実装の深掘り

SDK内部ではどのようにリクエストが構築されているのでしょうか。コミットログ（[e6562d7](https://github.com/anthropics/anthropic-sdk-typescript/commit/e6562d72502030e6cf90a31192b21b23c0b03422)）から要点を確認しました。

1. ベータ専用のエンドポイント
ベータ機能のため、`client.messages.create` ではなく `client.beta.messages.parse` (または `create`) を呼び出します。

2. ヘッダーの自動付与とハードコーディング
`parse` 関数を利用する場合、`betas` オプションは内部でハードコーディングされています。


parse関数内部での実装イメージ

https://github.com/anthropics/anthropic-sdk-typescript/blob/664cdd6bc91641bf610155d470a14c67a177a08d/src/resources/beta/messages/messages.ts#L140-L155


一方で `create` 関数を利用する場合は、引数として文字列を受け入れる柔軟な設計になっています。

3. output_formatの型定義
型定義上は、標準的な `input_schema` (JSON Schema) を受け取るようになっています。

https://github.com/anthropics/anthropic-sdk-typescript/blob/e6562d72502030e6cf90a31192b21b23c0b03422/src/resources/beta/messages/messages.ts#L887-L893

これを開発者が扱いやすくするために、前述の `betaZodOutputFormat` ヘルパーが用意され、ZodスキーマからJSON Schemaへの変換を担っています。

---

## Python SDKの実装詳細

Python版でも同様に、Pydanticモデルを利用した直感的な記述が可能です。

### 基本的な使い方

```python: example.py
from pydantic import BaseModel
from anthropic import Anthropic, transform_schema

class ContactInfo(BaseModel):
    name: str
    email: str
    plan_interest: str
    demo_requested: bool

client = Anthropic()

# パターンA: .create() を使う場合 (transform_schemaが必要)
response = client.beta.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    betas=["structured-outputs-2025-11-13"],
    messages=[...],
    output_format={
        "type": "json_schema",
        "schema": transform_schema(ContactInfo),
    }
)

# パターンB: .parse() を使う場合 (Pydanticモデルを直接渡せる)
response = client.beta.messages.parse(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    betas=["structured-outputs-2025-11-13"],
    messages=[...],
    output_format=ContactInfo, # ここがシンプル！
)

print(response.parsed_output)

```

### 内部実装の確認

Python SDKの対応コミット（[688da81](https://github.com/anthropics/anthropic-sdk-python/commit/688da8126df8a304c5f01f0dc63cda437a62c217)）を確認すると、TypeScript版と同様の設計思想が見て取れます。

* ヘッダーの自動付与: `messages.py` 内で `structured-outputs-2025-11-13` がデフォルト値として設定されています。

https://github.com/anthropics/anthropic-sdk-python/blob/c94119bac55087fe8b53dc3c84c00b8a3535da52/src/anthropic/resources/beta/messages/messages.py#L1079-L1081


* パラメータの追加: `_beta_messages.py` にて `output_format` が引数として定義されています。

## 個人的な所感

今回のStructured Outputsのパブリックベータ化により、やっとOpenAIやGeminiと同様にClaudeでも構造化出力を利用できるようになりました。

ただ、自分が試している範囲では以下の観点でプロダクトへの導入は特に急がずしばらく様子見かなと思いました。

- そもそもClaudeのモデルが賢くてレスポンスの構造はStructured Outputsがなくても指示どりに出力してくれることが多い
- プロンプトに記載していた複雑なスキーマや各フィールドの説明をスキーマに記載すると、[公式も言うとおり](https://platform.claude.com/docs/en/build-with-claude/structured-outputs#schema-validation-errors) 複雑なスキーマ判定されてエラーになることが多い
- Zod v3 -> v4へのアップデートができていない(自分のせい)
