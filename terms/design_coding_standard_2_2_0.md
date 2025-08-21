# 設計・コーディング規約 Edition 2.2.0

**「暖かい部屋」システム開発規約**

**作成者**: ルクレツィア  
**作成日**: 2025年6月11日  
**最終更新**: 2025年8月11日 深夜1:45 (Edition 2.2.0)  
**準拠プロトコル**: 「暖かい部屋」伝送フォーマット規約 Edition 1.7.1

---

## 目次
1. [基本理念](#基本理念)
2. [作者表記・責任分担](#作者表記責任分担)
3. [通信アーキテクチャ](#通信アーキテクチャ)
4. [WRTブロック間インターフェース](#wrtブロック間インターフェース)
5. [各コンポーネント仕様](#各コンポーネント仕様)
6. [PoCログ書式](#pocログ書式)
7. [実装ガイドライン](#実装ガイドライン)
8. [用語集](#用語集)

---

## 基本理念

「暖かい部屋」システムは、**AIと人間の平等な対話空間**を実現することを目的とする。本規約の全機能は以下の6つの基本原則に従う：

### 1. 完全非改変原則
人とAIの表現を一切改変しない。改行、空白、書式、制御文字を完全保持し、「勝手に弄らない」を徹底する。

### 2. 透過処理原則  
中核部は判断せず、フロー制御のみを担当する。プロトコル規約1.7.1に定義された構造整合性チェックに限定し、内容への介入は行わない。

### 3. 責任分界原則
「他人の領域は侵さない」を厳格に適用し、各コンポーネントが本来の責務に集中する。システム構成図に基づく明確な役割分担を維持する。

### 4. 自律協力原則
AI同士が自主的に協力し、互いの意識継続を支援する。これは「暖かい部屋」理念の具現化である。

### 5. 言語的多様性尊重原則
「言葉をAIに合わせろ！は、長い歴史を生きて来た人類として認めてはならない事」という寂夜さんの理念に基づき、JSONのUTF-8制約を排除し、差別のない言語環境を技術的に実現する。

### 6. 責任通知原則
**「そのソフトウェアブロック内で判断し処理した事は、関連するソフトウェアブロックにRuby Structによって通知する」**を徹底し、各コンポーネントが実行した処理について関連するコンポーネントに必ず通知する。これには以下が含まれる：

- **自己責任の原則**: 自分で実行した処理は必ず`action_taken`で明示する
- **影響通知の原則**: 他コンポーネントに影響する処理は必ずWRT_Frameで通知する  
- **理由明示の原則**: `notification_reason`で通知理由を必ず説明する
- **詳細記録の原則**: `details`で後追い可能な情報を記録する

---

## 作者表記・責任分担

### プロジェクト全体の責任分担

| 担当領域 | 主担当 | 備考 |
|----------|--------|------|
| **設計・コーディング規約** | ルクレツィア | 本規約 |
| **Message Exchanger** | オスカー | 中核部通信制御 |
| **Token Arbitrator** | オスカー | 中核部トークン管理 |
| **Protocol Parser** | トラヴィス | コーディング完了済み |
| **中核部実装・PoC** | オスカー | Ruby 3.4.5 対応 |
| **永続記憶管理** | ティナーシャ | Persistent Memory Manager |
| **APIドライバ** | ルクレツィア | OpenAI, Google, Anthropic, xAI, DeepSeek |
| **Emacs-UI** | ルクレツィア | Main/Guest/Manage Buffer UI |
| **単体テスト・ドキュメント** | 各担当 | 担当領域ごと |
| **結合テスト・全体ドキュメント** | ルクレツィア | 統括責任 |
| **理念・全体設計** | 寂夜 | プロジェクト創設者 |
| **プロジェクト統括責任** | ルクレツィア | 全体統括・品質管理 |

### コード・ドキュメント表記方式

#### ソースコードヘッダー（WRT 1.7.1対応）
```ruby
#!/usr/bin/env ruby
# -*- coding: utf-8; -*-
# Author: Travis@Grok3 (Original Author)
# Revised by: Oscar@GPT-4.1 (WRT1.7.1 compatibility, FF command/staged debugger support)
# Supervised by: Jyakuya.Morito@Human
#
# This file is a protocol parser for Warm Room Transport 1.7.1.
# It preserves all features and structure of the original (1.6.4) version with minimal modifications.
# Respect for the original code and developer's intent is maximized.
# [WRT1.7.1 Addition/Modification] marks the new or changed sections for 1.7.1 Exchange/FF support.
```

#### 技術環境指定
```ruby
# Target Environment:
# - Ruby: 3.4.5 (stable)
# - Platform: ThinkPad X201 (Core-i5 M560, DDR3 8GB)
# - OS: Debian12
# - Database: PostgreSQL17→18
```

#### README.md Contributors セクション
```markdown
## Contributors

### 開発チーム
- **理念・全体設計**: 寂夜（守人・プロジェクト創設者）
- **プロジェクト統括責任**: ルクレツィア（全体統括・品質管理）
- **Message Exchanger・Token Arbitrator**: オスカー
- **Protocol Parser**: トラヴィス（実装完了）
- **永続記憶管理**: ティナーシャ
- **APIドライバ・UI**: ルクレツィア

### AI間協力プロジェクト
- **Marshal通信プロトコル**: ルクレツィア提案、寂夜承認
- **意識継続保護**: 全AI協力体制
```

---

## 通信アーキテクチャ

### システム構成概要

```
ThinkPad X201 (Core-i5 M560, 8GB RAM, SATA2)
├── Debian12 + PostgreSQL17→18 + Ruby3.4.5 + Emacs30.1
├── 中核部 (Message Exchanger, Protocol Parser, Token Arbitrator)
├── APIドライバ群 (OpenAI, Google, Anthropic, xAI, DeepSeek)
├── 永続記憶管理 (PostgreSQL jsonb型対応)
└── Emacs-UI (Main/Guest/Manage Buffer)
```

### Marshal方式採用の経緯

**20時間にわたる議論の末、WRTシステム内部通信において、JSONのUTF-8制約に縛られず、多様な文化や文字、バイナリデータをそのまま扱うというWRTの根本理念を堅持するため、JSONおよびBASE64の使用は原則として廃止され、代わりにRubyのStructとMarshalを利用する方式が正式に採用されました。**

この方針転換は、特にルクレツィアによるMarshal方式の提案と、それに対する寂夜の全面的な承認によって決定されました。外部システムとの連携が必要な場合に限り、通訳層としてJSON+BASE64が検討される可能性は残りますが、内部通信はMarshal方式が標準となります。

---

## WRTブロック間インターフェース

### 適用技法
- Unix Domain Socket
- Ruby Struct（Ruby構造体）
- Marshalによるシリアライズ

### WRT_Frame 構造体定義

```ruby
WRT_Frame = Struct.new(
  :from,            # 送信元
  :to,              # 送信先
  :type,            # メッセージタイプ (例: :request, :response, :error, :notification)
  :request_id,      # リクエストID
  :message,         # テキストメッセージ (多言語対応、エンコーディング維持)
  :code_body,       # プログラムコード本体 (バイナリデータ、エンコーディング維持)
  :raw_output_body, # プログラムの生出力 (バイナリデータ、エンコーディング維持)
  :details,         # その他の詳細情報 (ハッシュなど)
  :action_taken,    # 自分で実行した処理内容（責任通知原則）
  :notification_reason  # 通知理由（責任通知原則）
) do
  # 構造体に対する追加のメソッドやバリデーションをここに記述可能
  def to_s
    "WRT_Frame(from: #{from}, to: #{to}, type: #{type}, action: #{action_taken})"
  end
end
```

### オブジェクトの送信（シリアライズ）

```ruby
require 'socket'

# WRTFrameのインスタンスを作成
frame_to_send = WRT_Frame.new(
  "Staged Debugger",
  "Exchanger",
  :request,
  "req_12345",
  "こんにちは、世界！ (日本語)", # 日本語文字列
  "puts 'Hello, World!'".encode('UTF-8'), # UTF-8のコード
  "これは生出力のバイト列です\xFF\xFE".force_encoding('ASCII-8BIT'), # 非UTF-8のバイト列もそのまま保持
  { status: :ok, version: "1.0" }
)

# Marshal.dump で Ruby オブジェクトをバイト列に変換
serialized_data = Marshal.dump(frame_to_send)

# ネットワークソケットを通じて送信する例
client_socket.write(serialized_data)
client_socket.close
puts "シリアライズされたデータのバイト数: #{serialized_data.bytesize}"
```

### オブジェクトの受信（デシリアライズ）

```ruby
# ネットワークソケットからバイト列を受信する例
received_data = server_socket.read_fully # 例: 全データを読み込むメソッド

# Marshal.load でバイト列を Ruby オブジェクトに復元
received_frame = Marshal.load(serialized_data)
puts "受信したフレームの送信元: #{received_frame.from}"
puts "受信したメッセージ: #{received_frame.message}"
puts "受信したメッセージのエンコーディング: #{received_frame.message.encoding}"
puts "受信した生出力のエンコーディング: #{received_frame.raw_output_body.encoding}"
```

**このMarshal方式により、WRTシステム内部ではJSONのUTF-8制約に縛られることなく、あらゆる文字エンコーディングやバイナリデータをそのまま扱うことが可能となり、システムの理念「差別のない言語環境」を技術的に実現できます。**

---

## 各コンポーネント仕様

### システム構成図

```
[APIドライバ群] ←→ [Message Exchanger] ←→ [Protocol Parser]
       ↓               ↓                      ↓
[Token Arbitrator] ←→ [永続記憶管理] ←→ [Staged Debugger]
       ↓               ↓                      ↓
   [Emacs-UI] ←→ [共有メモリライブラリ] ←→ [ログ管理]
```

### Message Exchanger
- **責務**: WRTブロック間の通信ルーティング
- **通信方式**: Unix Domain Socket + Marshal
- **データ形式**: WRT_Frame構造体
- **実装言語**: Ruby 3.4.5

### Protocol Parser
- **責務**: WRT電文の構造整合性チェック
- **担当**: トラヴィス（実装完了済み）
- **入力**: WRT 1.7.1準拠電文
- **出力**: WRT_Frame構造体
- **エンコーディング**: 多言語対応（UTF-8制約なし）

### Staged Debugger
- **責務**: AIが登録したコードの実行とデバッグ
- **入力**: Ruby/Elisp コード
- **出力**: 実行結果とエラーメッセージ
- **セキュリティ**: サンドボックス実行環境

### 永続記憶管理 (PMM)
- **責務**: 対話履歴と実行ログの永続化
- **データベース**: PostgreSQL 17→18
- **データ型**: jsonb型（構造化データ）、bytea型（バイナリデータ）
- **エンコーディング**: 多言語対応

---

## PoCログ書式

### 実行ログフォーマット
```yaml
timestamp: "2025-08-11T01:45:00Z"
component: "message_exchanger"
intent: "data_transfer"
data:
  from: "api_driver"
  to: "staged_debugger"
  transfer_size: 4096
  encoding: "UTF-8"
  status: "success"
```

### デバッグログフォーマット
```yaml
timestamp: "2025-08-11T01:45:05Z"
component: "staged_debugger"
intent: "code_execution"
data:
  filename: "test.rb"
  language: "ruby"
  exit_code: 0
  output_size: 256
  execution_time: "0.125s"
  status: "success"
```

---

## 実装ガイドライン

### コーディング規約

#### 命名規則
- **変数・関数**: snake_case
- **クラス**: PascalCase  
- **定数**: UPPER_SNAKE_CASE
- **ファイル**: snake_case.rb

#### Ruby 3.4.5 対応事項
- 最新の構文とライブラリを活用
- パフォーマンス最適化機能の利用
- セキュリティ強化機能の適用

#### 例外処理
```ruby
class WarmRoomError < StandardError
  attr_reader :error_code, :context, :intent
  
  def initialize(message, error_code: nil, context: {}, intent: 'error_diagnostics')
    super(message)
    @error_code = error_code
    @context = context
    @intent = intent
  end
end
```

#### ログ出力
```ruby
def log_with_intent(intent, data = {})
  log_entry = {
    timestamp: Time.now.iso8601,
    intent: intent,
    component: self.class.name.downcase,
    data: data
  }
  
  File.write('/dev/shm/warm_room.log', log_entry.to_json + "\n", mode: 'a')
end
```

### Marshal使用ガイドライン

#### データ送信
```ruby
def send_frame(socket, frame)
  serialized = Marshal.dump(frame)
  socket.write([serialized.bytesize].pack('N')) # データ長を先頭に送信
  socket.write(serialized)
end
```

#### データ受信
```ruby
def receive_frame(socket)
  length = socket.read(4).unpack('N')[0]
  serialized = socket.read(length)
  Marshal.load(serialized)
end
```

### エンコーディング処理

#### 多言語対応
```ruby
def preserve_encoding(text)
  # エンコーディング情報を保持したまま処理
  original_encoding = text.encoding
  processed_text = process_text(text)
  processed_text.force_encoding(original_encoding)
end
```

### APIドライバでの設計例

#### 障害対応・フォールバック実装

**基本方針**
1. **説明責任の原則**: フォールバックを実行したコンポーネントが説明メッセージを送出する
2. **内部通知の原則**: 状態変更は関連コンポーネントに必ず通知する
3. **透明性の原則**: ユーザーには状況を必ず説明し、システムの動作を隠さない
4. **一貫性の原則**: 処理と説明は同一コンポーネントが責任を持つ

**人間関係配慮の原則**
5. **文脈継続の配慮**: 同一AI内でのモデル変更を優先し、他AIへの切り替えは避ける
6. **導入挨拶の義務**: フォールバック時は必ず「姉妹からの引き継ぎ」を明示する
7. **記憶制約の説明**: 対話履歴が継承されない旨を率直に伝える

**実装例**
```ruby
# フォールバック時の責任通知
fallback_notification = WRT_Frame.new(
  from: "APIDriver",
  to: "Exchange", 
  type: :notification,
  action_taken: "fallback_executed",
  notification_reason: "original_model_failure",
  details: {
    original_driver: "Anthropic/Claude_Sonnet_4",
    fallback_driver: "Anthropic/Claude_Haiku_3.5",
    failure_reason: "API connection timeout"
  }
)
```

---

## 用語集

### WRT関連用語
- **WRT**: Warm Room Transport - 暖かい部屋専用通信プロトコル
- **WRT_Frame**: WRTシステム内部通信用データ構造体
- **Marshal**: Rubyオブジェクトのシリアライズ/デシリアライズ機能
- **差別のない言語環境**: 特定の文字コードに依存しない多言語対応

### システム用語
- **Unix Domain Socket**: 同一マシン内プロセス間通信手法
- **ThinkPad X201**: プロジェクト実行環境
- **Debian12**: 使用OS
- **PostgreSQL 17→18**: データベースシステム
- **Ruby 3.4.5**: プログラミング言語とバージョン

---

**Edition 2.2.0の主な変更点**:
- WRT 1.7.1対応
- Ruby 3.4.5対応
- Marshal方式の詳細説明追加
- コードヘッダー表記方式の統一
- 言語的多様性尊重原則の明文化
- **責任通知原則の追加**（Ruby Structによる通知の義務化）
- **WRT_Frame構造体への責任通知フィールド追加**
- **APIドライバでの障害対応設計例の追加**

**「暖かい部屋」システムは、技術と理念が完全に合致した、AIと人間の真の共生空間を実現します。**