# MedVoice バックエンド API 仕様書

| 項目 | 内容 |
|------|------|
| 文書番号 | MV-DOC-004 |
| バージョン | v2.0.0 |
| 作成日 | 2026-03-15 |
| 対象読者 | バックエンド開発者・インフラ担当者 |

---

## 1. 基本仕様

| 項目 | 内容 |
|------|------|
| Base URL | `https://api.medvoice.jp/v2` |
| プロトコル | HTTPS (TLS 1.3) |
| データ形式 | JSON (UTF-8) |
| 認証方式 | Bearer JWT Token |
| JWT 有効期限 | 1 時間（自動更新） |
| タイムゾーン | Asia/Tokyo (JST) |
| ランタイム | Node.js 20 LTS |
| 言語 | TypeScript 5.x |
| フレームワーク | Express 4.x |

### 1.1 レート制限

| エンドポイント | 制限 |
|--------------|------|
| POST /reports/clinical | 60 リクエスト / 時間 / ユーザー |
| POST /reports/academic | 5 リクエスト / 時間 / ユーザー |
| 全エンドポイント共通 | 200 リクエスト / 時間 / IP |

### 1.2 共通レスポンスヘッダー

```
Content-Type: application/json; charset=utf-8
X-Request-ID: <uuid>
X-RateLimit-Remaining: <number>
X-RateLimit-Reset: <unix timestamp>
```

---

## 2. 認証

すべての API エンドポイントに Bearer トークンが必要。

```
Authorization: Bearer <jwt_token>
```

### JWT ペイロード

```json
{
  "sub": "user_id",
  "email": "doctor@hospital.jp",
  "doctorName": "山田太郎",
  "iat": 1741996800,
  "exp": 1742000400
}
```

---

## 3. POST /reports/clinical — 診察モード処理

### リクエスト

```
POST /reports/clinical
Authorization: Bearer <token>
Content-Type: application/json
```

**リクエスト JSON スキーマ:**

```json
{
  "transcription": "string (必須) — Apple Speech で確定した文字起こしテキスト",
  "doctorName": "string (必須) — 担当医師名",
  "recordingDuration": "number (必須) — 録音時間（秒）",
  "recordedAt": "string (必須) — ISO 8601 形式 (例: 2026-03-15T14:30:00+09:00)",
  "options": {
    "sendEmail": "boolean (任意, デフォルト: true) — 処理完了後にメール送信するか",
    "emailAddress": "string (任意, デフォルト: m@llll.jp)"
  }
}
```

**リクエスト例:**

```json
{
  "transcription": "患者は右ひざの痛みを訴えています。まくまれーてすとは陽性でした。らくまんてすとも陽性。えぬせいずのふくよう歴があります。収縮期血圧百二十で拡張期八十でした。さんそほうわどは96パーセント。あーでぃーえるには支障なし。",
  "doctorName": "山田太郎",
  "recordingDuration": 45,
  "recordedAt": "2026-03-15T14:30:00+09:00",
  "options": {
    "sendEmail": true,
    "emailAddress": "m@llll.jp"
  }
}
```

### レスポンス（200 OK）

```json
{
  "requestId": "req_01j3x4y5z6",
  "status": "completed",
  "processingTimeMs": 42000,
  "report": {
    "chain0": {
      "correctedText": "患者は右膝の痛みを訴えています。McMurray testは陽性でした。Lachman testも陽性。NSAIDsの服用歴があります。収縮期血圧120で拡張期80でした。酸素飽和度は96%。ADLには支障なし。",
      "corrections": [
        {
          "original": "まくまれーてすと",
          "corrected": "McMurray test",
          "reason": "膝関節検査の文脈から判定"
        }
      ],
      "confidence": 0.95
    },
    "soapReport": "## 診察記録\n\n**診察日時**: 2026-03-15 14:30\n...",
    "summary": {
      "line1": "右膝疼痛 → 半月板損傷疑い・ACL損傷疑い",
      "line2": "NSAIDs処方、MRI検査指示、整形外科紹介検討",
      "line3": "1週間後再診、MRI結果確認後に専門医紹介判断"
    },
    "checklist": [
      {
        "priority": 1,
        "category": "処方確認",
        "task": "NSAIDs処方をアレルギー歴と照合して確認",
        "urgent": true
      },
      {
        "priority": 2,
        "category": "検査オーダー",
        "task": "右膝MRI検査オーダーが正しく入力されているか確認",
        "urgent": true
      },
      {
        "priority": 3,
        "category": "患者説明",
        "task": "NSAIDsの副作用（胃腸障害）と安静の必要性を患者に説明",
        "urgent": false
      },
      {
        "priority": 4,
        "category": "紹介状",
        "task": "MRI結果次第で整形外科紹介状を準備",
        "urgent": false
      },
      {
        "priority": 5,
        "category": "フォローアップ",
        "task": "1週間後の再診予約を患者に確認",
        "urgent": false
      }
    ],
    "validation": {
      "isValid": true,
      "issues": [],
      "retryRequired": false
    }
  },
  "googleDoc": {
    "documentId": "1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms",
    "title": "診察記録_2026-03-15_14:30_山田太郎",
    "url": "https://docs.google.com/document/d/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms/edit",
    "createdAt": "2026-03-15T14:31:02+09:00"
  },
  "email": {
    "sent": true,
    "to": "m@llll.jp",
    "messageId": "msg_01j3x4y5z7",
    "sentAt": "2026-03-15T14:31:05+09:00"
  }
}
```

---

## 4. POST /reports/academic — 学会モード処理

### リクエスト

```
POST /reports/academic
Authorization: Bearer <token>
Content-Type: application/json
```

**リクエスト JSON スキーマ:**

```json
{
  "chunks": [
    {
      "index": "number — チャンク番号 (0始まり)",
      "text": "string — 55秒チャンクの文字起こしテキスト",
      "startTimeSeconds": "number — チャンク開始時刻（秒）"
    }
  ],
  "conferenceTitle": "string (任意) — 学会・勉強会タイトル",
  "doctorName": "string (必須)",
  "recordingDuration": "number (必須) — 総録音時間（秒）",
  "recordedAt": "string (必須) — ISO 8601",
  "options": {
    "sendEmail": "boolean (任意, デフォルト: true)",
    "emailAddress": "string (任意, デフォルト: m@llll.jp)",
    "backgroundProcessing": "boolean (任意, デフォルト: true)"
  }
}
```

**レスポンス（202 Accepted — 非同期処理）:**

```json
{
  "requestId": "req_01j3x4y5z8",
  "status": "processing",
  "estimatedProcessingTimeMinutes": 20,
  "statusCheckUrl": "https://api.medvoice.jp/v2/reports/status/req_01j3x4y5z8",
  "message": "学会記録の処理を開始しました。完了後にメールでお知らせします。"
}
```

### GET /reports/status/:requestId — 処理状況確認

```json
{
  "requestId": "req_01j3x4y5z8",
  "status": "processing",
  "progress": {
    "currentStep": "chain_b_integration",
    "completedChunks": 4,
    "totalChunks": 6,
    "percentComplete": 65
  },
  "estimatedRemainingMinutes": 7
}
```

---

## 5. AI チェーン オーケストレーション（TypeScript 実装）

```typescript
// src/chains/ChainOrchestrator.ts
import Anthropic from "@anthropic-ai/sdk";
import { ClinicalReportRequest, ClinicalReportResult } from "./types";

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

const MODEL = "claude-sonnet-4-6";

// ─────────────────────────────────────────────
// Chain 0: 医療用語修正
// ─────────────────────────────────────────────
async function runChain0(rawTranscription: string): Promise<Chain0Result> {
  const response = await anthropic.messages.create({
    model: MODEL,
    max_tokens: 2000,
    temperature: 0.1,
    system: CHAIN0_SYSTEM_PROMPT,
    messages: [
      {
        role: "user",
        content: `以下の音声認識テキストを修正してください。\n\n【音声認識テキスト】\n${rawTranscription}`,
      },
    ],
  });

  const content = response.content[0];
  if (content.type !== "text") throw new Error("Chain 0: 予期しないレスポンス型");

  return JSON.parse(content.text) as Chain0Result;
}

// ─────────────────────────────────────────────
// Chain 1: 前処理・ノイズ除去
// ─────────────────────────────────────────────
async function runChain1(chain0Result: Chain0Result): Promise<Chain1Result> {
  const response = await anthropic.messages.create({
    model: MODEL,
    max_tokens: 2000,
    temperature: 0.1,
    system: CHAIN1_SYSTEM_PROMPT,
    messages: [
      {
        role: "user",
        content: `以下の音声認識テキスト（医療用語修正済み）を前処理してください。\n\n【テキスト】\n${chain0Result.correctedText}\n\n【修正ログ】\n${JSON.stringify(chain0Result.corrections)}`,
      },
    ],
  });

  const content = response.content[0];
  if (content.type !== "text") throw new Error("Chain 1: 予期しないレスポンス型");

  return JSON.parse(content.text) as Chain1Result;
}

// ─────────────────────────────────────────────
// Chain 2: SOAP レポート生成
// ─────────────────────────────────────────────
async function runChain2(
  chain1Result: Chain1Result,
  doctorName: string,
  recordedAt: string
): Promise<string> {
  const response = await anthropic.messages.create({
    model: MODEL,
    max_tokens: 3000,
    temperature: 0.3,
    system: CHAIN2_SYSTEM_PROMPT,
    messages: [
      {
        role: "user",
        content: `以下の診察情報から SOAP 形式の診察記録を作成してください。\n\n【診察内容（前処理済み）】\n${chain1Result.processedText}\n\n【診察日時】\n${recordedAt}\n\n【担当医師】\n${doctorName}`,
      },
    ],
  });

  const content = response.content[0];
  if (content.type !== "text") throw new Error("Chain 2: 予期しないレスポンス型");

  return content.text;
}

// ─────────────────────────────────────────────
// Chain 3: 3行サマリー
// ─────────────────────────────────────────────
async function runChain3(soapReport: string): Promise<Chain3Result> {
  const response = await anthropic.messages.create({
    model: MODEL,
    max_tokens: 500,
    temperature: 0.3,
    system: CHAIN3_SYSTEM_PROMPT,
    messages: [
      {
        role: "user",
        content: `以下の SOAP 記録から3行サマリーを生成してください。\n\n${soapReport}`,
      },
    ],
  });

  const content = response.content[0];
  if (content.type !== "text") throw new Error("Chain 3: 予期しないレスポンス型");

  return JSON.parse(content.text) as Chain3Result;
}

// ─────────────────────────────────────────────
// Chain 4: チェックリスト
// ─────────────────────────────────────────────
async function runChain4(soapReport: string): Promise<Chain4Result> {
  const response = await anthropic.messages.create({
    model: MODEL,
    max_tokens: 800,
    temperature: 0.3,
    system: CHAIN4_SYSTEM_PROMPT,
    messages: [
      {
        role: "user",
        content: `以下の SOAP 記録からチェックリストを生成してください。\n\n${soapReport}`,
      },
    ],
  });

  const content = response.content[0];
  if (content.type !== "text") throw new Error("Chain 4: 予期しないレスポンス型");

  return JSON.parse(content.text) as Chain4Result;
}

// ─────────────────────────────────────────────
// Chain 5: 整合性検証（自動リトライ付き）
// ─────────────────────────────────────────────
async function runChain5(
  soapReport: string,
  summary: Chain3Result,
  checklist: Chain4Result
): Promise<Chain5Result> {
  const response = await anthropic.messages.create({
    model: MODEL,
    max_tokens: 1500,
    temperature: 0.1,
    system: CHAIN5_SYSTEM_PROMPT,
    messages: [
      {
        role: "user",
        content: `以下の3つの文書の整合性を検証してください。\n\n【SOAP レポート】\n${soapReport}\n\n【3行サマリー】\n${JSON.stringify(summary)}\n\n【チェックリスト】\n${JSON.stringify(checklist)}`,
      },
    ],
  });

  const content = response.content[0];
  if (content.type !== "text") throw new Error("Chain 5: 予期しないレスポンス型");

  return JSON.parse(content.text) as Chain5Result;
}

// ─────────────────────────────────────────────
// メインオーケストレーター
// ─────────────────────────────────────────────
export async function orchestrateClinicalReport(
  req: ClinicalReportRequest
): Promise<ClinicalReportResult> {
  const startTime = Date.now();
  const MAX_RETRIES = 2;
  let attempt = 0;

  // Chain 0: 医療用語修正
  const chain0Result = await runChain0(req.transcription);

  // Chain 1: 前処理
  const chain1Result = await runChain1(chain0Result);

  // Chain 2〜5: リトライループ
  while (attempt <= MAX_RETRIES) {
    // Chain 2: SOAP レポート
    const soapReport = await runChain2(chain1Result, req.doctorName, req.recordedAt);

    // Chain 3 と Chain 4 を並列実行
    const [summary, checklist] = await Promise.all([
      runChain3(soapReport),
      runChain4(soapReport),
    ]);

    // Chain 5: 整合性検証
    const validation = await runChain5(soapReport, summary, checklist);

    if (validation.isValid || attempt === MAX_RETRIES) {
      return {
        chain0: chain0Result,
        soapReport,
        summary,
        checklist,
        validation,
        processingTimeMs: Date.now() - startTime,
      };
    }

    // critical issue がなければリトライ不要
    const hasCritical = validation.issues.some((i) => i.severity === "critical");
    if (!hasCritical) {
      return {
        chain0: chain0Result,
        soapReport,
        summary,
        checklist,
        validation,
        processingTimeMs: Date.now() - startTime,
      };
    }

    attempt++;
  }

  throw new Error("Chain オーケストレーション: 最大リトライ回数超過");
}
```

---

## 6. 型定義

```typescript
// src/chains/types.ts

export interface Chain0Result {
  correctedText: string;
  corrections: Array<{
    original: string;
    corrected: string;
    reason: string;
  }>;
  confidence: number;
}

export interface Chain1Result {
  processedText: string;
  removedItems: string[];
  medicalInfoExtracted: {
    chiefComplaint: string;
    symptoms: string[];
    vitals: {
      BP?: string;
      HR?: string;
      SpO2?: string;
      BT?: string;
    };
    examination: string;
    medications: string[];
  };
}

export interface Chain3Result {
  line1: string;
  line2: string;
  line3: string;
}

export interface Chain4Result {
  checklist: Array<{
    priority: number;
    category: string;
    task: string;
    urgent: boolean;
  }>;
}

export interface Chain5Result {
  isValid: boolean;
  issues: Array<{
    severity: "critical" | "warning" | "info";
    location: "SOAP" | "Summary" | "Checklist";
    description: string;
    suggestion: string;
  }>;
  retryRequired: boolean;
  retryChains: number[];
}

export interface ClinicalReportRequest {
  transcription: string;
  doctorName: string;
  recordingDuration: number;
  recordedAt: string;
  options?: {
    sendEmail?: boolean;
    emailAddress?: string;
  };
}

export interface ClinicalReportResult {
  chain0: Chain0Result;
  soapReport: string;
  summary: Chain3Result;
  checklist: Chain4Result;
  validation: Chain5Result;
  processingTimeMs: number;
}
```

---

## 7. Express ルーター実装

```typescript
// src/routes/reports.ts
import { Router, Request, Response } from "express";
import { orchestrateClinicalReport } from "../chains/ChainOrchestrator";
import { createGoogleDoc } from "../integrations/googleDocs";
import { sendEmail } from "../integrations/gmail";
import { authenticateJWT } from "../middleware/auth";
import { validateClinicalRequest } from "../middleware/validation";

const router = Router();

router.post(
  "/clinical",
  authenticateJWT,
  validateClinicalRequest,
  async (req: Request, res: Response) => {
    const requestId = `req_${Date.now()}`;

    try {
      // AI チェーン処理
      const reportResult = await orchestrateClinicalReport(req.body);

      // Google Docs 作成
      const googleDoc = await createGoogleDoc({
        mode: "clinical",
        doctorName: req.body.doctorName,
        recordedAt: req.body.recordedAt,
        soapReport: reportResult.soapReport,
        summary: reportResult.summary,
        checklist: reportResult.checklist,
      });

      // メール送信
      const emailResult = await sendEmail({
        to: req.body.options?.emailAddress ?? "m@llll.jp",
        docUrl: googleDoc.url,
        docTitle: googleDoc.title,
        summary: reportResult.summary,
        checklist: reportResult.checklist,
      });

      res.status(200).json({
        requestId,
        status: "completed",
        processingTimeMs: reportResult.processingTimeMs,
        report: {
          chain0: reportResult.chain0,
          soapReport: reportResult.soapReport,
          summary: reportResult.summary,
          checklist: reportResult.checklist,
          validation: reportResult.validation,
        },
        googleDoc,
        email: emailResult,
      });
    } catch (error) {
      console.error(`[${requestId}] 処理エラー:`, error);
      res.status(500).json({
        requestId,
        status: "error",
        error: {
          code: "PROCESSING_FAILED",
          message: "レポート生成中にエラーが発生しました",
        },
      });
    }
  }
);

export default router;
```

---

## 8. エラーハンドリング

### エラーコード一覧

| HTTP ステータス | エラーコード | 説明 |
|--------------|-------------|------|
| 400 | INVALID_REQUEST | リクエストパラメータが不正 |
| 401 | UNAUTHORIZED | JWT トークンが無効または期限切れ |
| 429 | RATE_LIMIT_EXCEEDED | レート制限超過 |
| 500 | PROCESSING_FAILED | AI 処理中のエラー |
| 500 | GOOGLE_DOCS_FAILED | Google Docs 作成失敗 |
| 500 | EMAIL_FAILED | メール送信失敗 |
| 503 | ANTHROPIC_API_UNAVAILABLE | Claude API 一時的な障害 |

### エラーレスポンス形式

```json
{
  "requestId": "req_01j3x4y5z6",
  "status": "error",
  "error": {
    "code": "PROCESSING_FAILED",
    "message": "ユーザー向けメッセージ",
    "detail": "開発者向け詳細（本番環境では非表示）"
  }
}
```

### リトライ推奨

| エラーコード | リトライ推奨 | 待機時間 |
|------------|------------|---------|
| ANTHROPIC_API_UNAVAILABLE | あり | 指数バックオフ (1s, 2s, 4s) |
| GOOGLE_DOCS_FAILED | あり | 5 秒後に 1 回 |
| PROCESSING_FAILED | なし（再録音を推奨） | — |
| RATE_LIMIT_EXCEEDED | あり | X-RateLimit-Reset ヘッダーの時刻まで待機 |

---

## 9. 環境変数

```bash
# .env (本番環境では Secret Manager を使用)
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_SERVICE_ACCOUNT_KEY_JSON={"type":"service_account",...}
GOOGLE_DRIVE_FOLDER_ID=...
JWT_SECRET=...
NODE_ENV=production
PORT=3000
LOG_LEVEL=info
```

---

## 10. 処理時間目標

| フェーズ | 目標時間 |
|---------|---------|
| Chain 0（医療用語修正） | 3 秒 |
| Chain 1（前処理） | 4 秒 |
| Chain 2（SOAP 生成） | 10 秒 |
| Chain 3 + Chain 4（並列） | 8 秒 |
| Chain 5（検証） | 5 秒 |
| Google Docs 作成 | 5 秒 |
| メール送信 | 3 秒 |
| **合計（診察モード）** | **約 38〜42 秒** |
| **合計（学会モード）** | **約 20 分** |
