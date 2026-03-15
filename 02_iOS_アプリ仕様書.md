# MedVoice iOS アプリ仕様書

| 項目 | 内容 |
|------|------|
| 文書番号 | MV-DOC-002 |
| バージョン | v2.0.0 |
| 作成日 | 2026-03-15 |
| 対象読者 | iOS 開発者 |

---

## 1. アプリ基本情報

| 項目 | 内容 |
|------|------|
| アプリ名 | MedVoice |
| Bundle ID | jp.medvoice.app |
| 最小 iOS バージョン | iOS 17.0 |
| 対応デバイス | iPhone・iPad |
| 開発言語 | Swift 5.9+ |
| UI フレームワーク | SwiftUI |
| アーキテクチャ | MVVM + Combine |

---

## 2. 音声認識技術選定：Apple Speech Framework

### 2.1 技術比較

| 評価項目 | SFSpeechRecognizer (Apple) | 選定理由 |
|---------|---------------------------|---------|
| リアルタイム性 | ◎ 録音中にリアルタイム認識 | 録音終了と同時にテキスト確定 |
| 処理完了タイミング | ◎ 録音停止と同時に完了 | 待ち時間ゼロ |
| オフライン対応 | ◯ 部分対応（iOS 16+） | ネットワーク障害に強い |
| 導入コスト | ◎ 無料（Apple 標準 API） | 追加コストなし |
| プライバシー | ◎ デバイス上処理オプション | 医療情報の保護 |
| 日本語精度 | ◯ 良好（医療用語は Chain 0 で補正） | 運用実績あり |
| 実装複雑性 | ◯ 標準 API、十分なドキュメント | 保守性が高い |

**結論**: SFSpeechRecognizer を採用。録音終了と同時にテキストが確定するリアルタイム性が、ユーザー体験において最も重要な要素であるため。

---

## 3. プロジェクトディレクトリ構造

```
MedVoice/
├── App/
│   ├── MedVoiceApp.swift          # アプリエントリーポイント
│   └── AppDelegate.swift          # BGProcessingTask 登録
├── Features/
│   ├── Recording/
│   │   ├── Views/
│   │   │   ├── RecordingView.swift        # メイン録音画面
│   │   │   ├── ClinicalModeView.swift     # 診察モード UI
│   │   │   └── AcademicModeView.swift     # 学会モード UI
│   │   ├── ViewModels/
│   │   │   └── RecordingViewModel.swift   # 録音状態管理
│   │   └── Models/
│   │       └── RecordingSession.swift     # 録音セッションデータ
│   ├── Result/
│   │   ├── Views/
│   │   │   └── ResultView.swift           # 結果表示画面
│   │   └── ViewModels/
│   │       └── ResultViewModel.swift
│   └── Settings/
│       └── Views/
│           └── SettingsView.swift
├── Core/
│   ├── Audio/
│   │   ├── AudioRecorder.swift            # AVFoundation 録音管理
│   │   └── AppleSpeechTranscriber.swift   # SFSpeechRecognizer ラッパー
│   ├── Network/
│   │   ├── APIClient.swift                # バックエンド API クライアント
│   │   └── APIModels.swift               # リクエスト/レスポンスモデル
│   └── Background/
│       └── BackgroundProcessor.swift      # BGProcessingTask 管理
├── Resources/
│   ├── Assets.xcassets
│   ├── Localizable.strings
│   └── Info.plist
└── Tests/
    ├── Unit/
    └── Integration/
```

---

## 4. AVFoundation 録音実装

```swift
// AudioRecorder.swift
import AVFoundation
import Combine

final class AudioRecorder: NSObject, ObservableObject {

    // MARK: - Published Properties
    @Published var isRecording: Bool = false
    @Published var currentLevel: Float = 0.0
    @Published var elapsedTime: TimeInterval = 0.0

    // MARK: - Private Properties
    private let engine = AVAudioEngine()
    private var levelTimer: Timer?
    private var elapsedTimer: Timer?
    private var startTime: Date?

    /// AVAudioEngine の入力ノードから SFSpeechRecognizer に渡すためのバッファを提供
    var inputNode: AVAudioInputNode {
        return engine.inputNode
    }

    // MARK: - Setup

    func configureAudioSession() throws {
        let session = AVAudioSession.sharedInstance()
        try session.setCategory(
            .record,
            mode: .measurement,
            options: [.duckOthers, .allowBluetooth]
        )
        try session.setPreferredSampleRate(16_000)
        try session.setPreferredIOBufferDuration(0.1)
        try session.setActive(true, options: .notifyOthersOnDeactivation)
    }

    // MARK: - Recording Control

    func startRecording() throws {
        guard !isRecording else { return }

        try configureAudioSession()
        try engine.start()

        isRecording = true
        startTime = Date()

        // 音量レベル監視（UI メーター用）
        levelTimer = Timer.scheduledTimer(withTimeInterval: 0.05, repeats: true) { [weak self] _ in
            self?.updateAudioLevel()
        }

        // 経過時間カウンター
        elapsedTimer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
            guard let self, let start = self.startTime else { return }
            self.elapsedTime = Date().timeIntervalSince(start)
        }
    }

    func stopRecording() {
        guard isRecording else { return }

        engine.stop()
        engine.inputNode.removeTap(onBus: 0)

        levelTimer?.invalidate()
        elapsedTimer?.invalidate()
        levelTimer = nil
        elapsedTimer = nil

        isRecording = false
        startTime = nil

        try? AVAudioSession.sharedInstance().setActive(false)
    }

    // MARK: - Audio Level

    private func updateAudioLevel() {
        let inputNode = engine.inputNode
        let level = inputNode.volume
        currentLevel = level
    }
}
```

---

## 5. AppleSpeechTranscriber.swift — 完全実装

```swift
// AppleSpeechTranscriber.swift
import Speech
import AVFoundation
import Combine

/// SFSpeechRecognizer を使用したリアルタイム音声文字起こしクラス
/// - 診察モード: シングルタスクで録音終了まで継続
/// - 学会モード: 55 秒ごとにタスクを再起動してチャンクを蓄積
final class AppleSpeechTranscriber: NSObject, ObservableObject {

    // MARK: - Published Properties

    @Published var liveTranscription: String = ""   // UI リアルタイム表示用
    @Published var isTranscribing: Bool = false
    @Published var errorMessage: String? = nil

    // MARK: - Private Properties

    private let speechRecognizer: SFSpeechRecognizer
    private var recognitionRequest: SFSpeechAudioBufferRecognitionRequest?
    private var recognitionTask: SFSpeechRecognitionTask?
    private let audioRecorder: AudioRecorder

    /// 学会モード用: 確定済みチャンクテキストの蓄積
    private var accumulatedChunks: [String] = []

    /// 55 秒タスク再起動タイマー（学会モードのみ）
    private var chunkTimer: Timer?

    /// 現在のモード
    private var currentMode: RecordingMode = .clinical

    // MARK: - Constants

    /// SFSpeechRecognizer の最大連続認識時間制限（約 60 秒）に対し、
    /// 余裕を持って 55 秒でタスクを再起動する
    private let chunkDuration: TimeInterval = 55.0

    // MARK: - Initialization

    init(audioRecorder: AudioRecorder) {
        self.audioRecorder = audioRecorder
        self.speechRecognizer = SFSpeechRecognizer(locale: Locale(identifier: "ja-JP"))!
        super.init()
        speechRecognizer.delegate = self
    }

    // MARK: - Authorization

    /// マイクと音声認識の権限を要求
    func requestAuthorization() async -> Bool {
        let speechStatus = await withCheckedContinuation { continuation in
            SFSpeechRecognizer.requestAuthorization { status in
                continuation.resume(returning: status)
            }
        }

        let micStatus = await AVAudioApplication.requestRecordPermission()

        return speechStatus == .authorized && micStatus
    }

    // MARK: - Public Interface

    /// リアルタイム文字起こしを開始する
    /// - Parameter mode: .clinical（診察）または .academic（学会）
    func startLiveTranscription(mode: RecordingMode) throws {
        guard speechRecognizer.isAvailable else {
            throw TranscriberError.recognizerUnavailable
        }

        currentMode = mode
        accumulatedChunks = []
        liveTranscription = ""
        isTranscribing = true

        // 最初の認識タスクを開始
        try startNewRecognitionTask()

        // 学会モードのみ 55 秒チャンクタイマーを起動
        if mode == .academic {
            startChunkTimer()
        }
    }

    /// 録音を停止し、最終テキストを即座に返す
    /// - Returns: 完全な文字起こしテキスト（録音終了と同時に確定）
    @discardableResult
    func stopAndFinalize() -> String {
        // チャンクタイマー停止
        chunkTimer?.invalidate()
        chunkTimer = nil

        // 現在の認識タスクを終了し、最終テキストを確定
        recognitionRequest?.endAudio()

        // 現在のセグメントのテキストを取得
        let currentSegmentText = liveTranscription

        // 学会モード: 蓄積チャンクを結合
        let finalText: String
        if currentMode == .academic {
            var allChunks = accumulatedChunks
            if !currentSegmentText.isEmpty {
                allChunks.append(currentSegmentText)
            }
            finalText = allChunks.joined(separator: "\n\n")
        } else {
            finalText = currentSegmentText
        }

        // 認識タスクのクリーンアップ
        recognitionTask?.cancel()
        recognitionTask = nil
        recognitionRequest = nil

        isTranscribing = false

        return finalText
    }

    // MARK: - Private: Recognition Task Management

    /// 新しい SFSpeechAudioBufferRecognitionRequest タスクを開始する
    private func startNewRecognitionTask() throws {
        // 前のタスクをキャンセル（チャンク再起動時）
        recognitionTask?.cancel()
        recognitionTask = nil

        // 新しいリクエストを作成
        let request = SFSpeechAudioBufferRecognitionRequest()
        request.shouldReportPartialResults = true
        request.requiresOnDeviceRecognition = false  // 精度優先（オンラインモード）

        // 医療用語ヒントを提供（認識精度向上）
        request.contextualStrings = MedicalTermHints.japaneseTerms

        recognitionRequest = request

        // AVAudioEngine の入力ノードにタップを設置
        let inputNode = audioRecorder.inputNode
        let recordingFormat = inputNode.outputFormat(forBus: 0)

        inputNode.installTap(onBus: 0, bufferSize: 1024, format: recordingFormat) { [weak self] buffer, _ in
            self?.recognitionRequest?.append(buffer)
        }

        // 認識タスクを開始
        recognitionTask = speechRecognizer.recognitionTask(with: request) { [weak self] result, error in
            guard let self else { return }

            if let result {
                DispatchQueue.main.async {
                    self.liveTranscription = result.bestTranscription.formattedString
                }
            }

            if let error {
                // エラーコード 203 (認識タスク中断) は学会モードのチャンク切り替え時の正常動作
                let nsError = error as NSError
                if nsError.code != 203 {
                    DispatchQueue.main.async {
                        self.errorMessage = "音声認識エラー: \(error.localizedDescription)"
                    }
                }
            }
        }
    }

    // MARK: - Private: Chunk Timer (Academic Mode)

    /// 55 秒ごとにチャンクを確定し、新しいタスクを開始する
    private func startChunkTimer() {
        chunkTimer = Timer.scheduledTimer(withTimeInterval: chunkDuration, repeats: true) { [weak self] _ in
            self?.rotateChunk()
        }
    }

    private func rotateChunk() {
        // 現在のセグメントテキストを確定チャンクとして保存
        let completedChunk = liveTranscription
        if !completedChunk.isEmpty {
            accumulatedChunks.append(completedChunk)
        }

        // 現在の認識タスクを終了
        recognitionRequest?.endAudio()
        audioRecorder.inputNode.removeTap(onBus: 0)

        // 新しい認識タスクを開始
        liveTranscription = ""
        do {
            try startNewRecognitionTask()
        } catch {
            errorMessage = "チャンク再起動エラー: \(error.localizedDescription)"
        }
    }
}

// MARK: - SFSpeechRecognizerDelegate

extension AppleSpeechTranscriber: SFSpeechRecognizerDelegate {
    func speechRecognizer(_ speechRecognizer: SFSpeechRecognizer, availabilityDidChange available: Bool) {
        DispatchQueue.main.async {
            if !available {
                self.errorMessage = "音声認識が利用できません。ネットワーク接続を確認してください。"
            }
        }
    }
}

// MARK: - Supporting Types

enum RecordingMode {
    case clinical   // 診察モード
    case academic   // 学会モード
}

enum TranscriberError: LocalizedError {
    case recognizerUnavailable
    case authorizationDenied
    case audioEngineError(Error)

    var errorDescription: String? {
        switch self {
        case .recognizerUnavailable:
            return "音声認識エンジンが利用できません"
        case .authorizationDenied:
            return "音声認識の権限が拒否されています。設定アプリから許可してください"
        case .audioEngineError(let error):
            return "音声エンジンエラー: \(error.localizedDescription)"
        }
    }
}

/// 医療用語ヒント（認識精度向上用）
enum MedicalTermHints {
    static let japaneseTerms: [String] = [
        "McMurray test", "SOAP", "主訴", "現病歴", "既往歴",
        "アレルギー", "バイタルサイン", "収縮期血圧", "拡張期血圧",
        "心拍数", "体温", "酸素飽和度", "聴診", "触診", "打診",
        "処方", "投薬", "経過観察", "再診", "紹介状",
        "Lachman test", "anterior drawer test", "Apley test",
        "ROM", "ADL", "NSAIDs", "PTP", "BP", "HR", "SpO2"
    ]
}
```

---

## 6. RecordingViewModel — 録音から API 送信までのフロー

```swift
// RecordingViewModel.swift
import SwiftUI
import Combine

@MainActor
final class RecordingViewModel: ObservableObject {

    // MARK: - Published Properties

    @Published var recordingState: RecordingState = .idle
    @Published var liveTranscription: String = ""
    @Published var elapsedTime: TimeInterval = 0
    @Published var processingProgress: ProcessingProgress = .none
    @Published var errorMessage: String?

    // MARK: - Private Properties

    private let audioRecorder: AudioRecorder
    private let transcriber: AppleSpeechTranscriber
    private let apiClient: APIClient
    private var cancellables = Set<AnyCancellable>()

    private var currentMode: RecordingMode = .clinical
    private var doctorName: String = ""

    // MARK: - Initialization

    init() {
        self.audioRecorder = AudioRecorder()
        self.transcriber = AppleSpeechTranscriber(audioRecorder: audioRecorder)
        self.apiClient = APIClient()

        setupBindings()
    }

    private func setupBindings() {
        // リアルタイム文字起こしを UI に反映
        transcriber.$liveTranscription
            .receive(on: DispatchQueue.main)
            .assign(to: &$liveTranscription)

        // 経過時間を UI に反映
        audioRecorder.$elapsedTime
            .receive(on: DispatchQueue.main)
            .assign(to: &$elapsedTime)

        // エラーを UI に反映
        transcriber.$errorMessage
            .receive(on: DispatchQueue.main)
            .compactMap { $0 }
            .assign(to: &$errorMessage)
    }

    // MARK: - Public Interface

    /// 録音開始
    func startRecording(mode: RecordingMode, doctorName: String) async {
        let authorized = await transcriber.requestAuthorization()
        guard authorized else {
            errorMessage = "マイクと音声認識の権限が必要です"
            return
        }

        currentMode = mode
        self.doctorName = doctorName

        do {
            try audioRecorder.startRecording()
            try transcriber.startLiveTranscription(mode: mode)
            recordingState = .recording
        } catch {
            errorMessage = "録音開始エラー: \(error.localizedDescription)"
        }
    }

    /// 録音停止 → テキスト確定 → API 送信
    func stopRecordingAndProcess() async {
        guard recordingState == .recording else { return }

        recordingState = .processing
        processingProgress = .transcribing

        // 1. 録音停止と同時にテキスト確定（即座に返る）
        let finalText = transcriber.stopAndFinalize()
        audioRecorder.stopRecording()

        processingProgress = .sendingToAPI

        // 2. バックエンド API に送信
        do {
            let result = try await apiClient.processReport(
                transcription: finalText,
                mode: currentMode,
                doctorName: doctorName
            )

            processingProgress = .completed(result)
            recordingState = .completed(result)

        } catch {
            errorMessage = "処理エラー: \(error.localizedDescription)"
            recordingState = .error(error)
        }
    }
}

// MARK: - Supporting Enums

enum RecordingState {
    case idle
    case recording
    case processing
    case completed(ReportResult)
    case error(Error)
}

enum ProcessingProgress {
    case none
    case transcribing
    case sendingToAPI
    case completed(ReportResult)
}
```

---

## 7. 学会モード（60 分）長時間録音の処理

### 7.1 55 秒チャンクループの仕組み

SFSpeechRecognizer は内部的に約 60 秒の認識タスク制限を持つ。学会モードでは、この制限に達する前の 55 秒でタスクを自動的に再起動し、テキストを蓄積する。

```
時間軸:
00:00  録音開始 → 認識タスク #1 開始
00:55  チャンク #1 確定（〜55秒分のテキスト保存）→ 認識タスク #2 開始
01:50  チャンク #2 確定 → 認識タスク #3 開始
...（繰り返し）
60:00  録音停止 → 最終チャンク確定 → 全チャンクを結合
```

### 7.2 並列処理アーキテクチャ

60 分録音中、Apple Speech は録音と並行してリアルタイムに認識が進む。録音終了後の追加処理待ち時間はゼロ。バックエンドの Claude 処理（約 20 分）が主な待ち時間となる。

```
録音中（60分）:
  ├── AVAudioEngine: 音声収録
  ├── SFSpeechRecognizer: リアルタイム認識（55秒チャンク）
  └── UI: 文字起こし結果をリアルタイム表示

録音終了後（〜20分）:
  ├── テキスト確定（即座）
  ├── API 送信
  ├── Claude Chain A × 6（チャンクサマリー） — BGProcessingTask で実行
  ├── Claude Chain B（統合）
  ├── Claude Chain C（実践的示唆）
  └── Claude Chain D（学会チェックリスト）
```

---

## 8. BGProcessingTask — バックグラウンド AI 処理

学会モードの Claude 処理（約 20 分）をアプリがバックグラウンドに移行しても継続できるよう、BGProcessingTask を使用する。

```swift
// BackgroundProcessor.swift
import BackgroundTasks
import UIKit

final class BackgroundProcessor {

    static let taskIdentifier = "jp.medvoice.app.academic-processing"

    // MARK: - Registration (AppDelegate)

    static func register() {
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: taskIdentifier,
            using: nil
        ) { task in
            guard let processingTask = task as? BGProcessingTask else { return }
            BackgroundProcessor.handleProcessingTask(processingTask)
        }
    }

    // MARK: - Schedule

    static func scheduleProcessingTask(transcriptionText: String, doctorName: String) {
        let request = BGProcessingTaskRequest(identifier: taskIdentifier)
        request.requiresNetworkConnectivity = true
        request.requiresExternalPower = false

        // ユーザーデータを UserDefaults に一時保存
        UserDefaults.standard.set(transcriptionText, forKey: "pending_transcription")
        UserDefaults.standard.set(doctorName, forKey: "pending_doctor_name")

        do {
            try BGTaskScheduler.shared.submit(request)
        } catch {
            print("BGTask スケジュール失敗: \(error)")
        }
    }

    // MARK: - Handler

    private static func handleProcessingTask(_ task: BGProcessingTask) {
        let transcription = UserDefaults.standard.string(forKey: "pending_transcription") ?? ""
        let doctorName = UserDefaults.standard.string(forKey: "pending_doctor_name") ?? ""

        let processingTask = Task {
            let apiClient = APIClient()
            do {
                _ = try await apiClient.processReport(
                    transcription: transcription,
                    mode: .academic,
                    doctorName: doctorName
                )
                task.setTaskCompleted(success: true)
            } catch {
                task.setTaskCompleted(success: false)
            }

            // 一時データをクリア
            UserDefaults.standard.removeObject(forKey: "pending_transcription")
            UserDefaults.standard.removeObject(forKey: "pending_doctor_name")
        }

        task.expirationHandler = {
            processingTask.cancel()
        }
    }
}
```

### Info.plist への追加設定

```xml
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>jp.medvoice.app.academic-processing</string>
</array>
<key>NSMicrophoneUsageDescription</key>
<string>診察・学会の音声を録音して医療文書を自動生成するために使用します</string>
<key>NSSpeechRecognitionUsageDescription</key>
<string>録音音声をリアルタイムでテキストに変換するために使用します</string>
```

---

## 9. UI 仕様（SwiftUI）

### 9.1 メイン画面 — モード選択

```swift
// RecordingView.swift
struct RecordingView: View {
    @StateObject private var viewModel = RecordingViewModel()
    @State private var selectedMode: RecordingMode = .clinical
    @State private var doctorName: String = ""

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                // モード切り替え
                Picker("モード", selection: $selectedMode) {
                    Text("診察モード").tag(RecordingMode.clinical)
                    Text("学会モード").tag(RecordingMode.academic)
                }
                .pickerStyle(.segmented)
                .padding(.horizontal)

                // 医師名入力
                TextField("医師名", text: $doctorName)
                    .textFieldStyle(.roundedBorder)
                    .padding(.horizontal)

                // リアルタイム文字起こし表示
                ScrollView {
                    Text(viewModel.liveTranscription.isEmpty
                         ? "録音を開始してください"
                         : viewModel.liveTranscription)
                        .frame(maxWidth: .infinity, alignment: .leading)
                        .padding()
                }
                .background(Color(.systemGray6))
                .cornerRadius(12)
                .padding(.horizontal)
                .frame(minHeight: 200)

                // 経過時間表示
                if viewModel.recordingState == .recording {
                    Text(formatTime(viewModel.elapsedTime))
                        .font(.system(size: 48, weight: .thin, design: .monospaced))
                        .foregroundColor(.red)
                }

                // 録音ボタン
                RecordButton(viewModel: viewModel, mode: selectedMode, doctorName: doctorName)
            }
            .navigationTitle("MedVoice")
        }
    }
}
```

### 9.2 診察モード UI 特徴

| 要素 | 仕様 |
|------|------|
| 録音ボタン | 大きな赤丸ボタン（タップしやすい） |
| リアルタイム表示 | 発話と同時にテキストが流れる |
| 経過時間 | 秒単位の大きな数字表示 |
| 処理状態 | シンプルなプログレスインジケーター |
| 結果表示 | SOAP セクションをアコーディオン表示 |

### 9.3 学会モード UI 特徴

| 要素 | 仕様 |
|------|------|
| 録音ボタン | 赤丸ボタン + 録音中パルスアニメーション |
| 経過時間 | MM:SS 形式の大きな表示 |
| チャンク表示 | 確定チャンク数（例: 「チャンク 3/12 確定」） |
| 処理進捗 | 「AI 処理中... (推定 20 分)」バナー |
| バックグラウンド | ホームボタンを押しても処理継続の通知 |

---

## 10. テストチェックリスト

### 機能テスト

- [ ] マイク権限リクエストが正しく表示される
- [ ] 音声認識権限リクエストが正しく表示される
- [ ] 診察モードで録音開始・停止が正常動作する
- [ ] リアルタイム文字起こしが UI に即座に反映される
- [ ] 録音停止と同時にテキストが確定する（待ち時間なし）
- [ ] 確定テキストがバックエンド API に正常送信される
- [ ] 学会モードで 55 秒チャンクが正常に切り替わる
- [ ] 学会モードで 60 分録音が途切れなく継続する
- [ ] BGProcessingTask がバックグラウンドで正常動作する
- [ ] エラー発生時にユーザーフレンドリーなメッセージが表示される

### 音声認識精度テスト

- [ ] 日本語医療用語の認識（「主訴」「既往歴」「バイタルサイン」）
- [ ] ノイズ環境下での認識精度
- [ ] 複数話者が混在する場合の動作

### パフォーマンステスト

- [ ] 診察モード: 録音停止から API 送信まで 1 秒以内
- [ ] 学会モード: 60 分録音でメモリリーク発生なし
- [ ] バッテリー消費が許容範囲内（60 分録音で 20% 以下）

### エッジケーステスト

- [ ] 録音中にアプリをバックグラウンドに移行
- [ ] 録音中に電話着信
- [ ] ネットワーク切断状態での動作
- [ ] iOS 権限が拒否された状態での動作
