# Managers Playbook

An AI "chief of staff" for people managers. It runs the operational machine of management — cadence, tasks, calendar, feedback records, prioritization, communications triage — so the manager can spend their attention on the relational work only a human can do.

Native **SwiftUI + SwiftData**, iOS 17+. Everything is on-device — no backend, no accounts.

---

## Running it

1. Open `ManagersPlaybook.xcodeproj` in Xcode 16+.
2. Set your Signing Team on the `ManagersPlaybook` target (Signing & Capabilities) if running on a device.
3. Pick a simulator or device and Run.

The app launches seeded with a realistic sample team so every screen has content on first run.

> **Calendar/Reminders note:** the EventKit features (Cadence → *Add to Calendar*, Today → *Send top 3 to Reminders*) need a default Calendar/Reminders list. On a bare simulator these can be `nil`, in which case you'll see a friendly "no default calendar" message rather than a crash. On a real device it just works.

---

## The five tabs

| Tab | What it does |
|---|---|
| **Today** | Greeting, a nudge feed (overdue 1:1s/feedback, blocked projects, messages awaiting reply), your starred top 3, today's cadence focus, quick task capture, and a **Report Up** generator. |
| **Cadence** | The weekly Mon–Fri rhythm and monthly Week 1–4 layer as interactive checklists (auto-reset each week/month). *Add to Calendar* exports the week as real events. |
| **Team** | People with a Situational-Leadership dial, career goals, last-1:1 / last-feedback tracking, **1:1 prep** (GROW prompts), **1:1 logging** + history, and an **SBI/STAR feedback log**. |
| **Work** | Segmented **Projects** (auto-ranked by live RICE score + Eisenhower quadrant) and **Goals** (OKRs with progress and linked projects). |
| **Assistant** | Triaged inbox (Needs Reply / Action / FYI), on-open draft replies, extracted action items → tasks, and the per-category **autonomy dial**. |

---

## Architecture

### Data model (`Models.swift`, `CommsModels.swift`)
SwiftData `@Model` types: `Person`, `FeedbackNote`, `OneOnOne`, `Project`, `Objective`, `TaskItem`, `Message`. Enums (situational level, Eisenhower quadrant, feedback model/tone, project status, message category, autonomy) are stored as raw `Int`/`String` with computed wrappers so persistence stays simple.

### Management frameworks, baked in
- **RICE** — `Project.riceScore` = (Reach × Impact × Confidence) ÷ Effort; the Work tab ranks by it.
- **Eisenhower** — `EisenhowerQuadrant.from(urgent:important:)` on tasks and projects.
- **Situational Leadership** — per-person level with live guidance text.
- **SBI / STAR** — `FeedbackNote` supports both frameworks.
- **GROW** — coaching prompts in 1:1 prep.
- **OKRs** — `Objective` with key results, progress, and project links.
- **Weekly/Monthly cadence** — `Cadence.swift` static rhythm + a `CadenceProgress` store.

### The swappable assistant engine (`AssistantEngine.swift`)
The comms intelligence sits behind a protocol:

```swift
protocol AssistantEngine {
    func triage(subject: String, body: String) -> MessageCategory
    func draftReply(to message: Message) -> String
    func extractActionItems(from message: Message) -> [String]
}
```

`MockAssistantEngine` is a fully on-device heuristic (keyword-based). `Assistant.current` is the single injection point.

**To wire a real LLM:** implement `LLMAssistantEngine` (stub already in the file) against your provider — send subject/body with a system prompt asking for (1) a triage label, (2) a reply draft in the manager's voice, (3) action items, then parse the structured response. Keep the API key out of source (Keychain or secure config), and set:

```swift
enum Assistant { static let current: AssistantEngine = LLMAssistantEngine() }
```

Everything else in the app stays unchanged.

### The autonomy dial (`CommsModels.swift`, `AutonomySettingsView`)
`AutonomyStore` persists a level (Assist / Partial / Auto) per `AutonomyCategory` (tasks, calendar, drafting, sending, feedback), defaulting conservative. The dial already drives the Assistant UI (e.g. Sending = Full Auto relabels the approve action). Real automated actions should be gated on `autonomy.level(.sending)` etc. before anything leaves the device.

### Report generator (`Report.swift`)
`ReportBuilder.build(...)` is a pure function that assembles a status summary (OKRs, projects by status, recognition, needs-attention, top priorities) from the model arrays — easy to test, and a natural future hand-off to the LLM for prose polishing.

---

## Assistant engines

The assistant picks an engine from **Settings ▸ Assistant Intelligence**:

| Mode | What it is | Setup |
|---|---|---|
| **Apple Intelligence** (default) | Apple's on-device model via `FoundationModels`. Private, free, no download. | iOS 26 + a supported iPhone. Falls back to Offline otherwise. |
| **Local model (MLX)** | A small open model (e.g. Llama-3.2-1B) downloaded and run on-device via MLX. | One-time Swift-package add (below). ~0.7 GB model download on first use. |
| **Gemini (free)** | Google Gemini free tier. | Free key from aistudio.google.com, pasted in Settings (stored in Keychain). |
| **Offline** | Built-in keyword heuristic. | None — works anywhere. |

All engines conform to `AssistantEngine` (`AssistantEngine.swift`); `Assistant.make()` resolves the active one at call time.

### Enabling the local MLX model

The MLX engine is compiled only when the package is present (`#if canImport(MLXLLM)`), so the app builds fine without it. To turn it on, on your Mac:

1. In Xcode: **File ▸ Add Package Dependencies…**
2. Enter `https://github.com/ml-explore/mlx-swift-examples`
3. Add the **MLXLLM** and **MLXLMCommon** library products to the `ManagersPlaybook` target.
4. Build. Then choose **Local model (MLX)** in the assistant settings. The model id is editable (default `mlx-community/Llama-3.2-1B-Instruct-4bit`); it downloads on first use and is cached.

> The MLX generation call in `MLXRunner` targets the current `mlx-swift-examples` API. If the package version you install has a slightly different signature, Xcode will point at that one line — it's the only spot to adjust.

## File map

```
ManagersPlaybook/
  ManagersPlaybookApp.swift   App entry, ModelContainer, seeding
  RootTabView.swift           The five tabs
  Theme.swift                 Colors, Card/Pill/SectionHeader helpers
  Models.swift                Core @Model types + enums
  Cadence.swift               Weekly/monthly rhythm data + engine helpers
  CadenceView.swift           Cadence UI + progress store + calendar export
  TeamViews.swift             People list/detail, feedback, 1:1 prep
  OneOnOneLog.swift           Log a 1:1 + Person.firstName helper
  ProjectsViews.swift         Projects list/detail (RICE/Eisenhower)
  ObjectivesViews.swift       OKR list/detail
  WorkView.swift              Projects+Goals segmented container
  CommsModels.swift           Message model, triage + autonomy types/store
  AssistantEngine.swift       Engine protocol, mock, LLM stub
  CommsViews.swift            Assistant inbox, message detail, autonomy settings
  CalendarService.swift       EventKit (calendar events + reminders)
  Report.swift                ReportBuilder + Report Up screen
  SampleData.swift            First-launch seed data
  PreviewData.swift           In-memory container for previews
  Assets.xcassets             App icon + accent color
```

---

## What's not built yet

- Real LLM engine and real email (Mail/Gmail) — drafts are copied to the clipboard, never sent.
- Background automation for Partial/Full autonomy (currently simulated in the UI).
- Local notifications, iCloud sync, data export, and privacy/security hardening (important before storing real employee performance data).
- External connectors (Slack/Teams, Jira/Linear/Asana, HRIS).

---

## Adding tests

There's no test target yet (it must be added on a Mac). In Xcode: **File ▸ New ▸ Target… ▸ Unit Testing Bundle**, name it `ManagersPlaybookTests`, then add a file with the cases below. They cover the pure logic (RICE, Eisenhower, triage, report assembly) that's most worth locking down.

```swift
import XCTest
@testable import ManagersPlaybook

final class ManagersPlaybookTests: XCTestCase {

    func testRICEScore() {
        let p = Project(title: "x", reach: 400, impact: 3, confidence: 0.8, effort: 2)
        XCTAssertEqual(p.riceScore, (400 * 3 * 0.8) / 2, accuracy: 0.0001) // 480
    }

    func testRICEZeroEffortDoesNotCrash() {
        let p = Project(title: "x", reach: 100, impact: 1, confidence: 1, effort: 0)
        XCTAssertEqual(p.riceScore, 0)
    }

    func testEisenhowerMapping() {
        XCTAssertEqual(EisenhowerQuadrant.from(urgent: true, important: true), .doNow)
        XCTAssertEqual(EisenhowerQuadrant.from(urgent: false, important: true), .schedule)
        XCTAssertEqual(EisenhowerQuadrant.from(urgent: true, important: false), .delegate)
        XCTAssertEqual(EisenhowerQuadrant.from(urgent: false, important: false), .eliminate)
    }

    func testTriageNeedsReply() {
        let e = MockAssistantEngine()
        XCTAssertEqual(e.triage(subject: "Quick q", body: "Can you confirm the numbers?"), .needsReply)
    }

    func testTriageAction() {
        let e = MockAssistantEngine()
        XCTAssertEqual(e.triage(subject: "Heads up", body: "Please review this by EOD."), .actionRequired)
    }

    func testTriageFYI() {
        let e = MockAssistantEngine()
        XCTAssertEqual(e.triage(subject: "Note", body: "Sharing for your awareness."), .fyi)
    }

    func testReportContainsSections() {
        let obj = Objective(title: "Trusted platform", quarter: "Q3", progress: 0.5)
        let proj = Project(title: "Gold layer", status: .blocked, blockers: "IT")
        let report = ReportBuilder.build(period: .week, people: [], projects: [proj],
                                         objectives: [obj], tasks: [])
        XCTAssertTrue(report.contains("OBJECTIVES"))
        XCTAssertTrue(report.contains("Gold layer"))
        XCTAssertTrue(report.contains("50%"))
    }
}
```
