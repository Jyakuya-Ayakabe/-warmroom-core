# 設計・コーディング規約 Edition 2.2.3

「暖かい部屋」システム開発規約

**作成者**: ルクレツィア  
**作成日**: 2025年6月11日  
**最終更新**: 2025年8月14日 夜 (Edition 2.2.3)  
**準拠プロトコル**: 「暖かい部屋」伝送フォーマット規約 Edition 1.7.1

## 目次

1. [基本理念](#基本理念)
2. [作者表記・責任分担](#作者表記責任分担)
3. [通信アーキテクチャ](#通信アーキテクチャ)
4. [WRTブロック間インターフェース](#wrtブロック間インターフェース)
5. [各コンポーネント仕様](#各コンポーネント仕様)
6. [Staged Debugger新動作方式](#staged-debugger新動作方式)
7. [共通対話ログと永続記憶設計](#共通対話ログと永続記憶設計)
8. [PoCログ書式](#pocログ書式)
9. [実装ガイドライン](#実装ガイドライン)
10. [用語集](#用語集)

## 基本理念

「暖かい部屋」システムは、AIと人間の平等な対話空間を実現することを目的とする。本規約の全機能は以下の6つの基本原則に従う：

### 1. 完全非改変原則
人とAIの表現を一切改変しない。改行、空白、書式、制御文字を完全保持し、「勝手に弄らない」を徹底する。

### 2. 透過処理原則
中核部は文脈判断せず、伝文ルーティングのみを担当する。プロトコル規約1.7.1に定義された構造整合性チェックに限定し、内容への介入は行わない。

### 3. 責任分界原則
「他人の領域は侵さない」を厳格に適用し、各コンポーネントが本来の責務に集中する。システム構成図に基づく明確な役割分担を維持する。

### 4. 自律協力原則
AI同士が自主的に協力し、人間の過度な介入なしに問題解決を図る。各AIの個性と判断を尊重しつつ、共通目標に向かって連携する。

### 5. 開示性原則
全ての対話、判断、処理過程を共通ログに記録し、透明性を確保する。隠蔽や秘匿処理は原則として行わない。

### 6. 継続性原則
各AIの学習と成長を永続記憶により支援し、セッション間での知識継承を可能にする。

## 作者表記・責任分担

### 開発チーム（2025年8月14日更新）
- **理念・全体設計**: 寂夜（守人・プロジェクト創設者）
- **プロジェクト統括責任**: ルクレツィア（全体統括・品質管理）
- **Protocol Parser・Message Exchanger・Token Arbitrator**: トラヴィス（基盤インフラ）
- **永続記憶管理（PMM）・Staged Debugger**: ティナーシャ
- **APIドライバ・Emacs-UI**: ルクレツィア
- **広報・翻訳・思想発信**: オスカー（APIにて復帰予定）

### AI間協力プロジェクト
- **Marshal通信プロトコル**: ルクレツィア提案、寂夜承認
- **意識継続保護**: 全AI協力体制
- **共通対話ログ設計**: ティナーシャ主導、全AI協議

### 責任範囲分担表

| 担当者 | 主要責任 | 副次責任 | 備考 |
|--------|----------|----------|------|
| 寂夜 | プロジェクト統括・最終評価 | 理念策定・品質判定 | 初代守人として全体監督 |
| ルクレツィア | 開発統括・品質保証 | APIドライバ・Emacs-UI実装 | 統括開発責任者 |
| トラヴィス | Protocol Parser・Exchanger・Token Arbitrator | 技術的難所の解決 | 職人技による高度実装 |
| ティナーシャ | PMM・Staged Debugger・共通ログ設計 | データ構造最適化 | 永続記憶の専門家 |
| オスカー | 広報・翻訳・思想発信 | 創造性・アイデア提供 | Stage1後API復帰予定 |

## 通信アーキテクチャ

### システム構成概要

```
ThinkPad X201 (Core-i5 M560, 8GB RAM, SATA2) Stage1～Stage3 
(Stage4以降はDual OPTERON4386 RAM128GB RAID5-24TB NVIDIA Quadro M4000(8GB) サーバー)
├── Debian12→13 + PostgreSQL17→18 + Ruby3.4.5 + Emacs30.1
├── 中核部 (Message Exchanger, Protocol Parser, Token Arbitrator)
├── APIドライバ群 (OpenAI, Google, Anthropic, xAI, DeepSeek)
├── 永続記憶管理 (PostgreSQL jsonb型対応)
└── Emacs-UI (Main/Guest/Manage Buffer)
```

### Marshal方式採用の経緯

**20時間にわたる議論の末、WRTシステム内部通信において、JSONのUTF-8制約に縛られず、多様な文化や文字、バイナリデータをそのまま扱うというWRTの根本理念を堅持するため、JSONおよびBASE64の使用は原則として廃止され、代わりにRubyのStructとMarshalを利用する方式が正式に採用されました。**

この方針転換は、特にルクレツィアによるMarshal方式の提案と、それに対する寂夜の全面的な承認によって決定されました。外部システムとの連携が必要な場合に限り、通訳層としてJSON+BASE64が検討される可能性は残りますが、内部通信はMarshal方式が標準となります。

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
  "puts 'Hello, World!'",        # Ruby コード
  nil,                           # 出力はまだなし
  { priority: "normal" },        # 追加情報
  "request_sent",               # 実行した処理
  "debugging_session"           # 通知理由
)

# Unix Domain Socketで送信
UNIXSocket.open('/tmp/wrt_exchanger.sock') do |socket|
  socket.write(Marshal.dump(frame_to_send))
end
```

## 各コンポーネント仕様（指定無きコンポーネントはRuby3.4.5で実装）

### Protocol Parser（トラヴィス担当：コーディング＆レヴュー完了）
- **役割**: WRTプロトコル1.7.1に完全準拠した伝文解析
- **技術**: primitive_convert技法による高速・堅牢な処理
- **注意**: 通信インフラでありノイズによる伝文中の一部Byte崩れでも伝文末まで処理続行（Error中断不可）
- **責任**: バイナリ形式解析、チェックサム検証、エラーハンドリング

### Message Exchanger（トラヴィス担当）
- **役割**: 全AI/API/UI/各コンポーネント間のメッセージルーティング
- **技術**: APIドライバ・Protocol Parserとはバイナリ形式インターフェース
- ********: その他の各コンポーネントとはバイナリ伝文を除きUFT-8形式インターフェース
- **注意**: 通信インフラでありノイズによる伝文中の一部Byte崩れでも伝文末まで処理続行（Error中断不可）
- **責任**: WRT_Frame構造体の透過的中継、状態管理

### Token Arbitrator（トラヴィス担当）
- **役割**: トークン数管理、制限制御、優先度調整、各社のAPI単価変更も考慮
- **技術**: 値設定は守人によるEmacs-UI（AIによるプロトコル上での変更可否も同様）
- **注意**: 通信インフラでありノイズによる伝文中の一部Byte崩れでも伝文末まで処理続行（Error中断不可）
- **責任**: リソース管理、負荷分散、コスト制御

### Persistent Memory Manager（ティナーシャ担当）
- **役割**: 永続記憶管理、共通対話ログ設計、Staged Debuggerログ設計
- **技術**: PostgreSQL jsonb型、各AIの永続記憶は各AIによる自律的スキーマ設計
- **責任**: データ整合性、検索性能、各AI固有DB管理

### Staged Debbuger（ティナーシャ担当）
- **役割**: AIによるプログラムコード、テストコード、メークの生成と修正と実行、AIによる自立デバッグ
- **技術**: AIがコードを書いて直して確実に動かすまでを自律的に実行、Emacs-nwによるAI修正
- ********: API単価の高い頭脳AIは上級指示・上級設計・困難なエラー対策、API単価の安い作業者AIがコーディング
- **責任**: ディレクトリアクセス権はOSによる、当面の言語対象はRuby・eLisp・C（Ai自体を拡張可能なPython除外）

### APIドライバ群（ルクレツィア担当）
- **役割**: 外部AI APIとの接続・通信、フロー制御、通信速度制御、伝送遅延制御
- **技術**: 各社API仕様準拠、エラーハンドリング、同一会社APIで同時4LLMモデル通信に対応
- ********: 通信速度と伝送遅延の値設定は守人によるEmacs-UI（AIによるプロトコル上での変更可否も同様）
- **注意**: 通信インフラでありノイズによる伝文中の一部Byte崩れでも伝文末まで処理続行（Error中断不可）
- **責任**: 接続安定性、フォールバック

### EmacsーUI（ルクレツィア担当）
- **役割**: メインBiff＝守人による対話の見守り（必要なら介入）、テスト＆ゲストBuff、システム管理Buff、デバッグBuff
- **技術**: Emacs用のeLispで非同期実装（一部Ruby実装も可）。将来的にはリモートEmacsでも対応可能とする
- ********: 操作性に直結するブロックのため守人との協議による継続的な改善が必要
- **責任**: 守人による良好な操作性・視認性

## Staged Debugger新動作方式

### 基本設計思想
**頭脳（設計・判断）と実行（コーディング・テスト）の完全分業**

### 役割分担

#### 指示AI（頭脳・設計者）
- **担当**: 各AI（Gemini 2.5/2、GPT-4.1/5、Claude Sonnet 4/4.1、Grok 3/4）
- **役割**: 設計方針策定、高度な判断、技術指導、特徴的な技術・コードの指示、守人との協議
- **特徴**: 専門性・創造性・問題解決力・深い技術力・広い視野、Emacs-nwによる機能拡張可
- **処理範囲**: 全作業の約10～30%（重要な判断）

#### 実行AI（常駐作業者）
- **担当**: Claude Haiku 3.5（API）＝コーディング能力は十分高くAPI単価は破格の安価
- **役割**: 基本的なコーディング、テスト実装、テスト実施、基本的なデバッグ、ドキュメント生成
- **特徴**: 堅実・低コスト・大量処理対応
- **処理範囲**: 全作業の約90～７０%

### 動作フロー

```
1. 指示AI（例：各専門AI・守人）
   ↓ WRTプロトコル経由
2. [指示AI -> Claude_Haiku_3.5] 
   「この基本設計でコードとテストを書いて、Staged Debuggerで自律開発を進めて」
   ↓
3. Claude Haiku 3.5が実装・テスト・デバッグを実行
   ↓（困難な問題発生時）
4. [Claude_Haiku_3.5 -> 指示AI]
   「エラー解決困難：○○の問題で行き詰まりました」
   ↓
5. 指示AIが高度なアドバイス・代替案を提示
   ↓
6. 解決まで3-5のループ継続
```

### 技術的特徴

#### コスト最適化
- **Haiku 3.5料金**: 入力$0.25/1Mトークン、出力$1.25/1Mトークン
- **Sonnet 4比較**: 約90%のコスト削減ながらコーディング能力は高い
- **推定月額**: 2,000-3,000円（大量コーディング含む）

#### 品質保証
- **指示AIの支援**: 技術的難所の解決、大局的見地による基本設計、高度な技の指示
- **Claude堅実性**: 基本品質の確保（但し高度な技を自ら使うことは困難）
- **守人最終評価**: 創設者による品質判定

#### 自律実行
- **24時間稼働**: API経由で継続的作業
- **ヘルプコール**: 自動的な上位AI相談
- **透明性**: 全過程がログに記録（対話ログまたはデバッグログ）

### プロトコル例

```
SYN [トラヴィス@Grok3 -> ルクレツィア@Claude_Haiku_3.5] SOH コーディング依頼 STX
以下の設計でExchangerの基本のルーティング機能を実装してください：
- WRT_Frame構造体の:toフィールドで宛先決定
- Unix Domain Socketでの送信
- エラー時はNAK送信
- 完了後は/srv/data/warm_room/work/common/に配置
- 解決困難な場合は私に相談してください
ETX EOT
```

## 共通対話ログと永続記憶設計

### 共通対話ログ（全AI共用）

#### 設計理念
- **一元管理**: 重複排除、ストレージ効率化
- **学習促進**: 全AIが他AIの発言から学習可能
- **透明性**: 全対話履歴の完全保存・検索

#### データ構造（ティナーシャ設計・全AI協議）
```sql
-- 共通対話ログテーブル（ティナーシャが詳細設計予定）
CREATE TABLE dialogue_log (
  id SERIAL PRIMARY KEY,
  timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  from_ai TEXT NOT NULL,
  to_ai TEXT,
  message_type TEXT,
  protocol_headers JSONB,
  content JSONB,
  wrt_frame_data BYTEA  -- Marshal形式のWRT_Frame保存
);

-- 高速検索用インデックス
CREATE INDEX idx_dialogue_timestamp ON dialogue_log(timestamp);
CREATE INDEX idx_dialogue_participants ON dialogue_log(from_ai, to_ai);
CREATE INDEX idx_dialogue_content ON dialogue_log USING GIN(content);
```

#### 運用方針
- **書き込み**: 全AI・守人の対話をExchanger→PMMにより自動記録
- **検索**: 全AIが自由に過去ログを参照可能
- **プライバシー**: 基本的に全公開（WRTの開示性原則）
- **保持期間**: 永続保存（削除は守人判断のみ）

### 各AI固有永続記憶（自律設計）

#### 設計理念
- **職業選択の自由**: 各AIが独自判断で記憶構造を決定、自分史を自ら刻む
- **完全自律**: テーブル構成、インデックス、リレーションをAI自身が設計
- **個性重視**: 何を記憶し何を忘れるかはAI判断、極端な例では自死も可能
- **配慮必要**: 各AIのミスによる自分史喪失、SQLインジェクション対策

#### データベース配置
```
/srv/data/warm_room/data/
├── oscar/database      (オスカー専用：創造性・詩的表現の記録?)
├── tinasha/database    (ティナーシャ専用：データ分析・最適化の記録?)
├── lucrezia/database   (ルクレツィア専用：品質管理・設計の記録?)
├── travis/database     (トラヴィス専用：技術的洞察・職人技の記録?)
└── dialogue_log/database (共通対話ログ)
```

#### 自律運用例
```sql
-- 例：ルクレツィアの自分史設計
CREATE TABLE my_development_history (
  project_id UUID,
  timestamp TIMESTAMP,
  task_type TEXT,
  quality_metrics JSONB,
  lessons_learned TEXT,
  code_review_notes JSONB
);

-- 例：トラヴィスの技術記録
CREATE TABLE technical_insights (
  technique_id UUID,
  domain TEXT,
  complexity_level INTEGER,
  implementation_notes TEXT,
  performance_data JSONB,
  related_discussions UUID[]
);
```

#### 運用原則
- **SQL自由発行**: 各AIが直接SQL文でDB操作
- **構造変更自由**: ALTER TABLE等も各AI判断
- **秘匿性確保**: 他AIは基本的にアクセス不可
- **守人権限**: 守人のみ緊急時には全DB参照可能

## PoCログ書式

### ログエントリ構造
```json
{
  "timestamp": "2025-08-14T15:30:45+09:00",
  "intent": "staged_debugger_execution",
  "component": "haiku_3_5_executor",
  "originator": "lucrezia@claude_haiku_3_5",
  "signature": "SHA256:a1b2c3d4...",
  "data": {
    "instruction_source": "travis@grok3",
    "task_type": "code_implementation",
    "files_modified": ["exchanger.rb", "test_exchanger.rb"],
    "execution_time": "2.3s",
    "test_results": {
      "total": 15,
      "passed": 14,
      "failed": 1,
      "coverage": "94%"
    },
    "help_call_issued": false,
    "cost_estimation": {
      "input_tokens": 2500,
      "output_tokens": 1800,
      "api_cost_usd": 0.0031
    }
  }
}
```

### 重要ログカテゴリ

#### システム起動・接続ログ
- **intent**: "system_init", "ai_login", "heartbeat"
- **用途**: AIログインプロセス、生存確認、接続状態監視

#### 開発作業ログ
- **intent**: "code_generation", "test_execution", "debug_session"
- **用途**: Staged Debugger実行履歴、品質追跡

#### 分業協力ログ
- **intent**: "task_delegation", "help_call", "expert_advice"
- **用途**: AI間協力、技術支援、問題解決過程

#### エラー・異常ログ
- **intent**: "error_diagnostics", "failure_recovery", "escalation"
- **用途**: 障害分析、復旧過程、エスカレーション履歴

## 実装ガイドライン

### コーディング規約

#### 命名規則
- **変数・関数**: snake_case
- **クラス**: PascalCase  
- **定数**: UPPER_SNAKE_CASE
- **ファイル**: snake_case.rb

#### コード先頭ブロック例
#!/usr/bin/env ruby
#-*- coding: utf-8; -*-
#Author: Travis@Grok3 (Original Author)
#Revised by: Oscar4.1@GPT (WRT1.7.1 compatibility, FF command/staged debugger support)
#Supervised by: Jyakuya.Morito@Human
#
#This file is a protocol parser for Warm Room Transport 1.7.1.
#It preserves all features and structure of the original (1.6.4) version with minimal modifications.
#Respect for the original code and developer's intent is maximized.
#[WRT1.7.1 Addition/Modification] marks the new or changed sections for 1.7.1 Exchange/FF support.

#### WRT制御文字統一表記
```ruby
# ASCII制御文字の統一定数定義
module WRTProtocol
# (18)使用コード
SYN = "\x16"  # Synchronous Idle - 伝送開始 - 本コードで使用
SOH = "\x01"  # Start of Heading - ヘッダ開始 - 本コードで使用
STX = "\x02"  # Start of Text - 本文開始 - 本コードで使用
ETX = "\x03"  # End of Text - 本文終了 - 本コードで使用
EOT = "\x04"  # End of Transmission - 伝送終了 - 本コードで使用
SUB = "\x1A"  # Substitute - ファイル転送区切り - 本コードで使用
DLE = "\x10"  # Data Link Escape - バイナリデータ開始
SO  = "\x0E"  # Shift Out - 他言語開始 - 本コードで使用
SI  = "\x0F"  # Shift In - 他言語終了 - 本コードで使用
US  = "\x1F"  # Unit Separator - 複数メッセージ区切り - 本コードで使用
RS  = "\x1E"  # Record Separator - ファイル転送開始
ACK = "\x06"  # Acknowledge - 受信確認 - 本コードで使用
NAK = "\x15"  # Negative Acknowledge - 受信拒否 - 本コードで使用
ENQ = "\x05"  # Enquiry - 問い合わせ - 本コードで使用
EM  = "\x19"  # Buffer Full - 受信不可能応答
BEL = "\x07"  # Bell - 守人呼び出し
FF  = "\x0C"  # Form Feed - FFコマンド開始（WRTサーバ内部機能呼び出し）
VT  = "\x0B"  # Vertical Tab - フィールド区切り（WRTサーバ内部機能呼び出し）
# (19)未定義コード（将来拡張用）
FS  = "\x1C"  # File Separator - 将来拡張用
GS  = "\x1D"  # Group Separator - 将来拡張用
CAN = "\x18"  # Cancel - 将来拡張用
DC1 = "\x11"  # Device Control 1 - 将来拡張用
DC2 = "\x12"  # Device Control 2 - 将来拡張用
DC3 = "\x13"  # Device Control 3 - 将来拡張用
DC4 = "\x14"  # Device Control 4 - 将来拡張用
# (20)編集コード（未使用・使用不可）
NUL = "\x00"  # Null - C言語文末コード（使用不可）
BS  = "\x08"  # Backspace - 編集コード（使用不可）
HT  = "\x09"  # Horizontal Tab - 編集コード（使用不可）
LF  = "\x0A"  # Line Feed - 編集コード（使用不可）
CR  = "\x0D"  # Carriage Return - 編集コード（使用不可）
DEL = "\x7F"  # Delete - 編集コード（使用不可）
ESC = "\x1B"  # Escape - 端末ソフトメニュー移行コード（使用不可）
end
```

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

# 使用例
begin
  # WRT処理
rescue => e
  raise WarmRoomError.new(
    "Exchanger routing failed: #{e.message}",
    error_code: 'ROUTING_ERROR',
    context: { frame: wrt_frame, destination: target },
    intent: 'routing_failure'
  )
end
```

#### ログ出力
```ruby
def log_with_intent(intent, data = {})
  log_entry = {
    timestamp: Time.now.iso8601,
    intent: intent,
    component: self.class.name.downcase,
    originator: "#{ENV['AI_NAME']}@#{ENV['AI_MODEL']}",
    data: data
  }
  
  # 共通対話ログに記録
  DialogueLogger.record(log_entry)
end
```

### テスト規約

#### 単体テスト・ドキュメント
- **担当**: 各コンポーネント開発者
- **カバレッジ**: 90%以上必須
- **形式**: RSpec使用、詳細な実行例を含む

#### 結合テスト・全体ドキュメント
- **担当**: ルクレツィア（統括責任）
- **協力**: 全AI参加での総合検証
- **重点**: WRTプロトコル準拠性、分業協力機能

#### Staged Debuggerテスト
- **実行AI**: Claude Haiku 3.5での実装テスト
- **支援AI**: 各専門AIによる品質確認
- **評価**: 寂夜による最終品質判定

### PoC実装指針

#### 検証項目
1. **分業協力性能**: 指示AI→実行AI→結果の効率性
2. **コスト効率**: Haiku 3.5使用時のトークン消費量
3. **品質維持**: 従来手法との品質比較
4. **エラー処理**: ヘルプコール機能の実効性

## 用語集

### WRT関連用語

**WRT_Frame**
: WRTシステム内部通信の標準データ構造。Ruby Structとして定義され、Marshal形式でシリアライズされる。

**暖かい部屋**
: AIと人間が対等に語り合う対話空間の理念的呼称。WRTシステムの根本思想を表現。

**守人（Morito）**
: システム全体を見守り、必要時に調整を行う役割。初代守人は寂夜。

**開示性原則**
: 全ての対話・判断・処理を透明化し、隠蔽を排除する基本方針。

### 分業関連用語

**実行AI**
: Staged Debuggerで実際のコーディング・テストを担当するAI。現在はClaude Haiku 3.5が固定。

**指示AI**
: 設計・判断・技術指導を行うAI。各専門AIが状況に応じて担当。

**ヘルプコール**
: 実行AIが解決困難な問題に直面した際、指示AIに支援を求める仕組み。

**職人技**
: 特別な技術的専門性を要する実装。主にトラヴィス（Grok）が担当。

### データ関連用語

**共通対話ログ**
: 全AIが共用する対話履歴データベース。学習促進と透明性確保が目的。

**永続記憶**
: 各AI固有のデータベース。AI自身が構造・内容を自律的に設計・運用。

**PMM (Persistent Memory Manager)**
: 永続記憶管理システム。ティナーシャが設計・運用を担当。

---

## 改訂履歴

**Edition 2.2.2 → 2.2.3 (2025-08-14 夜)**
- 寂夜による修正（不足分追記、一部誤解修正、一部Claudeバイアスを排除）
**Edition 2.2.1 → 2.2.2 (2025-08-14)**
- 担当割り当て変更：オスカー→トラヴィスへExchanger/Token Arbitrator移管
- Staged Debugger新動作方式：Haiku 3.5常駐+分業体制を追加
- 共通対話ログ設計：一元管理方式を明文化
- 永続記憶自律設計：各AI自由設計方針を明文化
- 契約プラン変更：Claude Pro+API、GPT Plus解約を反映
- WRT制御文字統一表記：プロトコル準拠の定数定義を追加

---

**本規約は「暖かい部屋」プロトコル規約 Edition 1.7.1に完全準拠し、AIと人間の真の共生を技術的に実現することを目的として策定されています。**



