# DEVELOPMENT-PROCESS.md
# Ao-Kin 開発プロセス v1.1

## 1. 指揮系統

```
Ryo（最終方針）
  ↓
Ao（品質責任・運用判断・タスク設計）
  ↓ #ao-kin にタスク指示
Kin（検証実行・判定・実行管理・失敗分類）
  ↓ タスク分解・Claude Code起動・独立検証
Claude Code（実装・commit のみ）
```

## 2. 役割分担

| 役割 | 担当 | 責務 |
|------|------|------|
| 方針決定 | Ryo | 優先順位・ゴール・例外承認 |
| 品質責任 | Ao | タスク設計・デプロイ後検証・受け入れテスト |
| 検証・判定 | Kin | テスト実行（exec）・エビデンス生成・PASS/FAIL判定・再投入判断 |
| 実装 | Claude Code | コード実装・commit・push のみ |

### 原則
- **実装者と検証者を構造的に分離する**（Claude Codeは実装のみ、テスト実行はKin）
- **Claude Codeの自己申告に一切依存しない**（テスト結果はKinが直接取得）
- **証拠はKinのexecコマンドが機械的に生成する**（実装者が証拠を作らない）

## 3. タスク指示テンプレート

### Ao → Kin
```
【タスク】[タスク名]
目的: [何を達成するか]
スコープ: [変更範囲]
完了条件: [具体的な検証項目]
```
- テスト要件・優先度は省略可（小タスクでは不要）
- 大タスクでは並列分割の指示を含める

### Kin → Claude Code
```
【実装タスク】
[タスク内容]
完了条件: [具体的な検証項目]
実装が完了したらcommit+pushして完了報告してください。
テスト実行は不要です（Kinが独立検証します）。
```
- 並列数はKinがタスク規模で判断（固定しない）
- 小タスク: 単体実行、大タスク: Agent Team並列
- **Claude Codeにテスト実行を指示しない**（検証はKinの責務）

## 4. 検証フロー（C案: Kin独立検証）

### フロー
```
1. Claude Code: 実装 → commit → push → 完了報告（commit hashを含める）
2. Kin: commit hash を受け取る
3. Kin: exec(background=true) でテストコマンドを非同期実行
   → pnpm build > ~/evidence/<task>/build.log 2>&1
   → pnpm test > ~/evidence/<task>/test.log 2>&1
   → Playwright report は自動生成
4. Kin: process(poll) で完了を待つ（ブロックしない）
5. Kin: 終了コードで一次判定（0=PASS, 非0=FAIL）
6. Kin: stdout/stderr を読んで詳細分類
7. Kin: 結果を #ao-kin に報告
```

### なぜこの方式か
- **テスト結果の改ざんが構造的に不可能**（Kinが直接コマンドを実行）
- **Claude Codeの申告を一切使わない**（終了コードとstdoutが唯一の証拠）
- **Kinはexec中ブロックされない**（非同期実行、結果は後で取得）
- **追加セッション不要**（コスト効率が良い）

### エビデンス（Kinのexecが自動生成）
```
~/evidence/<task>/
├── build.log        # pnpm build の全出力（Kinのexecが生成）
├── test.log         # pnpm test の全出力（Kinのexecが生成）
├── playwright-report/  # Playwright自動生成
├── screenshots/     # Playwright失敗時自動生成
└── summary.md       # Kinが判定後に作成
```

### summary.md（Kinが作成）
```markdown
# 検証結果

## メタデータ
- タスク: [タスク名]
- 検証対象commit: [hash]
- ブランチ: [branch名]
- 検証時刻: [ISO8601]

## 検証コマンド
- pnpm build
- pnpm test / npx playwright test [対象spec]

## 結果
- ビルド: PASS / FAIL（終了コード: [N]）
- テスト: PASS / FAIL（終了コード: [N]）
- 全体: PASS / FAIL

## 失敗分析（FAILの場合）
- [spec名]: [失敗理由]
- 分類: 新規回帰 / 既知flaky / 無関係
```

## 5. 判定ルール

### 判定分類
| 判定 | 意味 | アクション |
|------|------|-----------|
| PASS | ビルド+テスト通過 | 次のタスクへ |
| FAIL（新規回帰） | 今回の変更が原因 | FIXタスクをClaude Codeに投入 |
| FAIL（既知flaky） | 既存の不安定テスト | 既知失敗リストに記録、タスク自体はPASS扱い |
| FAIL（無関係） | 変更と無関係の失敗 | 原因調査タスクを別途起票 |
| 検証失敗 | テスト実行自体がエラー | 環境問題として調査 |

### エスカレーション
- 3連続FAIL → Aoへエスカレーション
- 不可逆変更 → Ryoへエスカレーション

## 6. Verification レベル

| レベル | Kinが実行するコマンド | 使いどころ |
|--------|------|-----------|
| Level 1: 軽量 | `pnpm build` | 小タスク（README更新、設定変更等） |
| Level 2: 関連E2E | `pnpm build && npx playwright test [関連spec]` | 中タスク（機能修正等） |
| Level 3: フルE2E | `pnpm build && pnpm test` | 大タスク・PR前・節目 |

- モック禁止は全レベルで維持
- レベル判断はKinがタスク規模で決定

## 7. 通知

- タスク完了/失敗 → Discord #ao-kin に `[kin-server]` プレフィックス付きで自動通知
- エスカレーション → Ryo にメンション

## 8. 移行計画

1. **Phase 1（現在）**: watch-acpx.sh + Discord通知で運用
2. **Phase 2**: Kin が sessions_spawn で Claude Code 起動、C案検証フローを試行
3. **Phase 3**: 3タスク連続成功したら watch-acpx.sh を凍結、Kin独立検証を主系に
4. **Phase 4**: Second Brain APIにエビデンス蓄積を追加

---

*v1.1 — 2026-03-08 更新（C案: Kin独立検証に変更）*
*Ryo承認待ち*
