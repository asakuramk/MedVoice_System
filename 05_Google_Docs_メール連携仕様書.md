# MedVoice Google Docs・メール連携仕様書

| 項目 | 内容 |
|------|------|
| 文書番号 | MV-DOC-005 |
| バージョン | v2.0.0 |
| 作成日 | 2026-03-15 |
| 対象読者 | バックエンド開発者・システム管理者 |

---

## 1. Google Workspace 連携概要

MedVoice は Google Service Account を使用して、医師の Google Drive に診察記録文書を自動作成し、Gmail で通知メールを送信する。

### 1.1 使用する Google API

| API | 用途 | スコープ |
|-----|------|---------|
| Google Docs API v1 | 文書作成・フォーマット | `https://www.googleapis.com/auth/documents` |
| Google Drive API v3 | ファイル管理・共有 | `https://www.googleapis.com/auth/drive` |
| Gmail API v1 | メール送信 | `https://www.googleapis.com/auth/gmail.send` |

---

## 2. Google Service Account セットアップ

### 2.1 セットアップ手順

```
1. Google Cloud Console でプロジェクト作成
   https://console.cloud.google.com/

2. 必要な API を有効化
   - Google Docs API
   - Google Drive API
   - Gmail API

3. Service Account を作成
   IAM と管理 → サービスアカウント → 作成
   名前: medvoice-backend
   ID: medvoice-backend@[project-id].iam.gserviceaccount.com

4. JSON キーを生成
   サービスアカウント → キー → 新しいキーを追加 → JSON

5. Domain-wide Delegation を設定（Gmail API 用）
   Google Workspace 管理コンソール → セキュリティ → API 制御
   → ドメイン全体の委任 → 追加
   クライアント ID: [Service Account のクライアント ID]
   スコープ: (上記 3 スコープを入力)

6. 保存先フォルダを医師の Google Drive で共有
   Service Account のメールアドレスに「編集者」権限を付与
```

### 2.2 Service Account 認証（TypeScript）

```typescript
// src/integrations/googleAuth.ts
import { google } from "googleapis";

const SCOPES = [
  "https://www.googleapis.com/auth/documents",
  "https://www.googleapis.com/auth/drive",
  "https://www.googleapis.com/auth/gmail.send",
];

export function getGoogleAuth() {
  const keyJson = JSON.parse(process.env.GOOGLE_SERVICE_ACCOUNT_KEY_JSON!);

  const auth = new google.auth.GoogleAuth({
    credentials: keyJson,
    scopes: SCOPES,
  });

  return auth;
}

export function getGmailAuthForUser(userEmail: string) {
  const keyJson = JSON.parse(process.env.GOOGLE_SERVICE_ACCOUNT_KEY_JSON!);

  // Gmail 送信は実際のユーザーへの委任が必要
  const auth = new google.auth.JWT({
    email: keyJson.client_email,
    key: keyJson.private_key,
    scopes: ["https://www.googleapis.com/auth/gmail.send"],
    subject: userEmail, // 委任するユーザーのメールアドレス
  });

  return auth;
}
```

---

## 3. Google Docs API 実装

### 3.1 文書タイトル形式

| モード | タイトル形式 | 例 |
|-------|------------|-----|
| 診察モード | `診察記録_YYYY-MM-DD_HH:mm_[医師名]` | `診察記録_2026-03-15_14:30_山田太郎` |
| 学会モード | `学会録音_YYYY-MM-DD_HH:mm_[タイトル]` | `学会録音_2026-03-15_10:00_心不全の最新治療` |

### 3.2 文書作成・フォーマット実装

```typescript
// src/integrations/googleDocs.ts
import { google, docs_v1 } from "googleapis";
import { getGoogleAuth } from "./googleAuth";
import { ClinicalDocumentData, AcademicDocumentData } from "./types";

interface GoogleDocResult {
  documentId: string;
  title: string;
  url: string;
  createdAt: string;
}

// ─────────────────────────────────────────────
// 診察モード: Google Doc 作成
// ─────────────────────────────────────────────
export async function createGoogleDoc(data: ClinicalDocumentData): Promise<GoogleDocResult> {
  const auth = await getGoogleAuth();
  const docsClient = google.docs({ version: "v1", auth });
  const driveClient = google.drive({ version: "v3", auth });

  // タイトル生成
  const jstDate = new Date(data.recordedAt);
  const dateStr = jstDate.toLocaleDateString("ja-JP", {
    year: "numeric", month: "2-digit", day: "2-digit",
    timeZone: "Asia/Tokyo",
  }).replace(/\//g, "-");
  const timeStr = jstDate.toLocaleTimeString("ja-JP", {
    hour: "2-digit", minute: "2-digit",
    timeZone: "Asia/Tokyo",
    hour12: false,
  });

  const title = data.mode === "clinical"
    ? `診察記録_${dateStr}_${timeStr}_${data.doctorName}`
    : `学会録音_${dateStr}_${timeStr}_${data.conferenceTitle ?? "無題"}`;

  // 空のドキュメントを作成
  const createResponse = await docsClient.documents.create({
    requestBody: { title },
  });

  const documentId = createResponse.data.documentId!;

  // batchUpdate でコンテンツを挿入・フォーマット
  const requests = buildDocumentRequests(data, title);

  await docsClient.documents.batchUpdate({
    documentId,
    requestBody: { requests },
  });

  // 指定フォルダに移動
  const folderId = process.env.GOOGLE_DRIVE_FOLDER_ID;
  if (folderId) {
    await driveClient.files.update({
      fileId: documentId,
      addParents: folderId,
      removeParents: "root",
      fields: "id, parents",
    });
  }

  return {
    documentId,
    title,
    url: `https://docs.google.com/document/d/${documentId}/edit`,
    createdAt: new Date().toISOString(),
  };
}

// ─────────────────────────────────────────────
// batchUpdate リクエスト構築
// ─────────────────────────────────────────────
function buildDocumentRequests(
  data: ClinicalDocumentData,
  title: string
): docs_v1.Schema$Request[] {
  const requests: docs_v1.Schema$Request[] = [];

  // テキスト挿入（末尾から逆順に挿入して位置がずれないようにする）
  // 実際の実装では、各セクションのインデックスを管理しながら挿入する

  const fullContent = buildDocumentContent(data);

  // 全テキストを一括挿入
  requests.push({
    insertText: {
      location: { index: 1 },
      text: fullContent.text,
    },
  });

  // 見出しフォーマット（HEADING_1）
  for (const heading of fullContent.headings) {
    requests.push({
      updateParagraphStyle: {
        range: {
          startIndex: heading.startIndex,
          endIndex: heading.endIndex,
        },
        paragraphStyle: {
          namedStyleType: "HEADING_1",
        },
        fields: "namedStyleType",
      },
    });
  }

  // 小見出しフォーマット（HEADING_2）
  for (const subheading of fullContent.subheadings) {
    requests.push({
      updateParagraphStyle: {
        range: {
          startIndex: subheading.startIndex,
          endIndex: subheading.endIndex,
        },
        paragraphStyle: {
          namedStyleType: "HEADING_2",
        },
        fields: "namedStyleType",
      },
    });
  }

  // 太字フォーマット（バイタルサイン・診断名など）
  for (const bold of fullContent.boldRanges) {
    requests.push({
      updateTextStyle: {
        range: {
          startIndex: bold.startIndex,
          endIndex: bold.endIndex,
        },
        textStyle: { bold: true },
        fields: "bold",
      },
    });
  }

  return requests;
}

// ─────────────────────────────────────────────
// 文書コンテンツ構築
// ─────────────────────────────────────────────
function buildDocumentContent(data: ClinicalDocumentData): DocumentContent {
  const lines: string[] = [];
  const headings: TextRange[] = [];
  const subheadings: TextRange[] = [];
  const boldRanges: TextRange[] = [];

  let currentIndex = 1; // Google Docs のインデックスは 1 始まり

  function addLine(text: string): void {
    lines.push(text + "\n");
    currentIndex += text.length + 1;
  }

  function addHeading(text: string): void {
    const start = currentIndex;
    addLine(text);
    headings.push({ startIndex: start, endIndex: currentIndex - 1 });
  }

  function addSubheading(text: string): void {
    const start = currentIndex;
    addLine(text);
    subheadings.push({ startIndex: start, endIndex: currentIndex - 1 });
  }

  // ── ヘッダー ──
  addHeading(`MedVoice 診察記録`);
  addLine(`診察日時: ${data.recordedAt}`);
  addLine(`担当医師: ${data.doctorName}`);
  addLine(`作成日時: ${new Date().toLocaleString("ja-JP", { timeZone: "Asia/Tokyo" })}`);
  addLine("─".repeat(40));
  addLine("");

  // ── 3行サマリー ──
  addSubheading("【サマリー】");
  addLine(`▶ ${data.summary.line1}`);
  addLine(`▶ ${data.summary.line2}`);
  addLine(`▶ ${data.summary.line3}`);
  addLine("");

  // ── SOAP レポート ──
  addSubheading("【SOAP 形式診察記録】");
  addLine(data.soapReport);
  addLine("");

  // ── チェックリスト ──
  addSubheading("【確認チェックリスト】");
  for (const item of data.checklist.checklist) {
    const urgentMark = item.urgent ? "🔴" : "⚪";
    addLine(`${urgentMark} ${item.priority}. [${item.category}] ${item.task}`);
  }
  addLine("");

  // ── 医療用語修正ログ ──
  addSubheading("【音声認識修正ログ（Chain 0）】");
  for (const correction of data.chain0.corrections) {
    addLine(`・「${correction.original}」→「${correction.corrected}」（${correction.reason}）`);
  }

  return {
    text: lines.join(""),
    headings,
    subheadings,
    boldRanges,
  };
}

interface TextRange {
  startIndex: number;
  endIndex: number;
}

interface DocumentContent {
  text: string;
  headings: TextRange[];
  subheadings: TextRange[];
  boldRanges: TextRange[];
}
```

---

## 4. Gmail API 実装

```typescript
// src/integrations/gmail.ts
import { google } from "googleapis";
import { getGmailAuthForUser } from "./googleAuth";
import { Chain3Result, Chain4Result } from "../chains/types";

interface EmailOptions {
  to: string;
  docUrl: string;
  docTitle: string;
  summary: Chain3Result;
  checklist: Chain4Result;
  mode?: "clinical" | "academic";
}

interface EmailResult {
  sent: boolean;
  to: string;
  messageId: string;
  sentAt: string;
}

export async function sendEmail(options: EmailOptions): Promise<EmailResult> {
  // 送信元アカウントのメールアドレス（Service Account に委任）
  const senderEmail = process.env.GMAIL_SENDER_EMAIL ?? "noreply@medvoice.jp";
  const auth = getGmailAuthForUser(senderEmail);
  const gmailClient = google.gmail({ version: "v1", auth });

  const subject = options.mode === "academic"
    ? `【MedVoice】学会記録が作成されました — ${options.docTitle}`
    : `【MedVoice】診察記録が作成されました — ${options.docTitle}`;

  const htmlBody = buildEmailHtml(options);

  // RFC 2822 形式のメッセージを Base64 エンコード
  const message = [
    `From: MedVoice <${senderEmail}>`,
    `To: ${options.to}`,
    `Subject: =?UTF-8?B?${Buffer.from(subject).toString("base64")}?=`,
    "MIME-Version: 1.0",
    'Content-Type: text/html; charset="UTF-8"',
    "Content-Transfer-Encoding: base64",
    "",
    Buffer.from(htmlBody).toString("base64"),
  ].join("\r\n");

  const encodedMessage = Buffer.from(message)
    .toString("base64")
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");

  const sendResponse = await gmailClient.users.messages.send({
    userId: "me",
    requestBody: { raw: encodedMessage },
  });

  return {
    sent: true,
    to: options.to,
    messageId: sendResponse.data.id!,
    sentAt: new Date().toISOString(),
  };
}

// ─────────────────────────────────────────────
// HTML メールテンプレート
// ─────────────────────────────────────────────
function buildEmailHtml(options: EmailOptions): string {
  const checklistHtml = options.checklist.checklist
    .map(
      (item) => `
      <tr>
        <td style="padding: 8px; border-bottom: 1px solid #eee;">
          <span style="color: ${item.urgent ? "#e53e3e" : "#888"}; font-weight: bold;">
            ${item.urgent ? "🔴" : "⚪"} ${item.priority}.
          </span>
        </td>
        <td style="padding: 8px; border-bottom: 1px solid #eee; color: #555;">
          [${item.category}]
        </td>
        <td style="padding: 8px; border-bottom: 1px solid #eee;">
          ${item.task}
        </td>
      </tr>`
    )
    .join("");

  return `
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body style="font-family: 'Hiragino Sans', 'Yu Gothic', Arial, sans-serif; background-color: #f5f7fa; margin: 0; padding: 20px;">

  <div style="max-width: 600px; margin: 0 auto; background: white; border-radius: 12px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">

    <!-- ヘッダー -->
    <div style="background: linear-gradient(135deg, #1a5276, #2980b9); padding: 28px 32px; color: white;">
      <h1 style="margin: 0; font-size: 22px; font-weight: 700;">MedVoice</h1>
      <p style="margin: 6px 0 0; font-size: 14px; opacity: 0.85;">診察記録が作成されました</p>
    </div>

    <!-- 本文 -->
    <div style="padding: 28px 32px;">

      <h2 style="color: #1a5276; font-size: 16px; margin-bottom: 8px;">${options.docTitle}</h2>
      <p style="color: #666; font-size: 14px; margin-bottom: 24px;">
        診察記録の作成が完了しました。以下のボタンから Google Docs でご確認ください。
      </p>

      <!-- Google Docs リンクボタン -->
      <div style="text-align: center; margin-bottom: 32px;">
        <a href="${options.docUrl}"
           style="display: inline-block; background: #2980b9; color: white; text-decoration: none;
                  padding: 14px 32px; border-radius: 8px; font-size: 16px; font-weight: 600;">
          📄 Google Docs で開く
        </a>
      </div>

      <!-- 3行サマリー -->
      <div style="background: #eaf4fb; border-left: 4px solid #2980b9; padding: 16px 20px; border-radius: 0 8px 8px 0; margin-bottom: 24px;">
        <h3 style="color: #1a5276; font-size: 14px; margin: 0 0 12px;">📋 サマリー</h3>
        <p style="margin: 4px 0; font-size: 14px; color: #333;">▶ ${options.summary.line1}</p>
        <p style="margin: 4px 0; font-size: 14px; color: #333;">▶ ${options.summary.line2}</p>
        <p style="margin: 4px 0; font-size: 14px; color: #333;">▶ ${options.summary.line3}</p>
      </div>

      <!-- チェックリスト -->
      <h3 style="color: #1a5276; font-size: 14px; margin-bottom: 12px;">✅ 確認チェックリスト</h3>
      <table style="width: 100%; border-collapse: collapse; font-size: 13px;">
        ${checklistHtml}
      </table>

    </div>

    <!-- フッター -->
    <div style="background: #f5f7fa; padding: 20px 32px; border-top: 1px solid #e2e8f0;">
      <p style="margin: 0; font-size: 12px; color: #999; text-align: center;">
        このメールは MedVoice システムが自動送信しています。<br>
        本文書はドラフトです。最終確認は担当医師が行ってください。<br>
        <a href="${options.docUrl}" style="color: #2980b9;">文書を開く</a>
      </p>
    </div>

  </div>
</body>
</html>`;
}
```

---

## 5. 文書フォーマット仕様

### 5.1 診察記録ドキュメント構成

```
【文書タイトル】診察記録_2026-03-15_14:30_山田太郎
─────────────────────────────────────────
[HEADING 1] MedVoice 診察記録
診察日時: 2026-03-15 14:30
担当医師: 山田太郎
作成日時: 2026-03-15 14:31

[HEADING 2] 【サマリー】
▶ 右膝疼痛 → 半月板損傷疑い・ACL損傷疑い
▶ NSAIDs処方、MRI検査指示、整形外科紹介検討
▶ 1週間後再診、MRI結果確認後に専門医紹介判断

[HEADING 2] 【SOAP 形式診察記録】
（Chain 2 の出力をそのまま挿入）

[HEADING 2] 【確認チェックリスト】
🔴 1. [処方確認] NSAIDs処方をアレルギー歴と照合して確認
🔴 2. [検査オーダー] 右膝MRI検査オーダーが正しく入力されているか確認
⚪ 3. [患者説明] NSAIDsの副作用と安静の必要性を患者に説明
⚪ 4. [紹介状] MRI結果次第で整形外科紹介状を準備
⚪ 5. [フォローアップ] 1週間後の再診予約を患者に確認

[HEADING 2] 【音声認識修正ログ（Chain 0）】
・「まくまれーてすと」→「McMurray test」（膝関節検査の文脈から判定）
・「らくまんてすと」→「Lachman test」（膝関節検査の文脈から判定）
・「えぬせいず」→「NSAIDs」（非ステロイド性抗炎症薬の略称）
```

### 5.2 学会録音ドキュメント構成

```
【文書タイトル】学会録音_2026-03-15_10:00_心不全の最新治療
─────────────────────────────────────────
[HEADING 1] MedVoice 学会記録
[HEADING 2] 【講演情報】
[HEADING 2] 【講演内容サマリー】（Chain B の出力）
[HEADING 2] 【実践的示唆】（Chain C の出力）
[HEADING 2] 【アクションリスト】（Chain D の出力）
[HEADING 2] 【処理情報】
```

---

## 6. エラーハンドリング

| シナリオ | 対処 |
|---------|------|
| Google Docs API タイムアウト | 5 秒待機後に 1 回リトライ |
| Drive 空き容量不足 | エラーログ記録、管理者アラート送信 |
| Gmail 送信失敗 | Google Doc URL をアプリ内に表示してフォールバック |
| Service Account 権限エラー | 管理者アラート、設定確認ガイドを表示 |

---

## 7. テスト確認事項

- [ ] Service Account が対象 Google Drive フォルダに書き込める
- [ ] 文書タイトルが正しい形式（タイムスタンプ・医師名）で生成される
- [ ] SOAP 内容が文書に正しくフォーマットされる
- [ ] チェックリストの優先度と緊急フラグが文書に反映される
- [ ] 修正ログが文書末尾に付記される
- [ ] Gmail 経由でメールが m@llll.jp に届く
- [ ] メール内の Google Docs リンクが正しく開ける
- [ ] メールの HTML が主要メールクライアントで正しく表示される（Gmail, Apple Mail）
- [ ] Domain-wide Delegation が機能している（Gmail 送信委任）
