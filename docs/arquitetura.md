# Tomafoco — Arquitetura (SOLID + Clean Architecture)

**Versão:** 1.2 (produto na v1.8 em produção)
**Data:** 2026-08-06
**Referências:** [`release.md`](release.md) (assinatura, notarização e atualização automática)

---

## 1. Princípios norteadores

A arquitetura segue **Clean Architecture** (dependências apontando sempre para dentro, em direção ao domínio) com os princípios **SOLID** aplicados explicitamente em cada fronteira. A regra de ouro:

> **Domain e Application não importam AppKit, SwiftUI, Foundation-de-sistema (Process, NSWorkspace) nem nada do macOS. Todo efeito colateral vive atrás de um protocolo.**

Isso atende diretamente o RNF-05 (testabilidade): a máquina de estados do Pomodoro e os casos de uso rodam em testes unitários puros, com relógio, bloqueadores e persistência falsos.

---

## 2. Camadas

```
┌──────────────────────────────────────────────────────────────┐
│  Presentation (SwiftUI)                                       │
│  Views, ViewModels (@Observable), MenuBar, DesignSystem       │
└───────────────▲──────────────────────────────────────────────┘
                │ depende de
┌───────────────┴──────────────────────────────────────────────┐
│  Application (casos de uso)                                   │
│  StartFocusSession, CompleteSession, CancelSession,           │
│  RecoverFromCrash, ManageBlockLists, TickSession              │
└───────────────▲──────────────────────────────────────────────┘
                │ depende de
┌───────────────┴──────────────────────────────────────────────┐
│  Domain (núcleo puro)                                         │
│  PomodoroSession, SessionPhase, BlockList, BlockedApp,        │
│  BlockedDomain, FocusTask, PomodoroConfiguration, DomainError │
│  + Ports (protocolos): AppBlocking, WebsiteBlocking,          │
│    SessionClock, SessionRepository, SettingsRepository,       │
│    TaskRepository, TaskImporting, UserNotifying,              │
│    ApplicationPicking, LoginItemManaging                      │
└───────────────▲──────────────────────────────────────────────┘
                │ implementada por (inversão de dependência)
┌───────────────┴──────────────────────────────────────────────┐
│  Infrastructure (adapters do macOS)                           │
│  WorkspaceAppBlocker (NSWorkspace),                           │
│  AppleScriptBrowserBlocker (automação de navegador, ADR-8),   │
│  DispatchSessionClock (DispatchSourceTimer),                  │
│  FileSessionSnapshotStore, FileTaskStore, UserDefaultsSettingsStore, │
│  EventKitReminderImporter, NSOpenPanelApplicationPicker,      │
│  SMAppServiceLoginItem, UNNotificationAdapter                 │
└──────────────────────────────────────────────────────────────┘
```

Setas de dependência: **Presentation → Application → Domain ← Infrastructure**. A Infrastructure conhece o Domain (implementa seus ports); o Domain não conhece ninguém. A composição acontece em um único ponto (**Composition Root**, no `@main`).

---

## 3. Domain — núcleo puro

### 3.1 Entidades e value objects

```swift
// Domain/Model/SessionPhase.swift
enum SessionPhase: Equatable, Codable {
    case idle
    case focus
    case shortBreak
    case longBreak
}

// Domain/Model/PomodoroSession.swift
struct PomodoroSession: Equatable, Codable {
    let id: UUID
    let phase: SessionPhase
    let startedAt: Date
    let endsAt: Date          // horário absoluto — sobrevive a crash (RF-01.5)
    let cycleNumber: Int

    var remaining: (now: Date) -> TimeInterval { { max(0, endsAt.timeIntervalSince($0)) } }
    func isExpired(now: Date) -> Bool { now >= endsAt }
}

// Domain/Model/BlockedDomain.swift — value object com validação (Clean Code: fail fast)
// Casamento em URLBlockingPolicy (puro): host exato ou curinga `*.` (RF-02.7) + prefixo de
// caminho por segmento inteiro (RF-02.8).
struct BlockedDomain: Equatable, Codable, Hashable {
    let host: String                // lowercase, sem esquema/porta/query/fragmento
    let includesSubdomains: Bool    // `*.globo.com`
    let path: String?               // `/shorts` (sem barra final); nil = host inteiro
    var value: String { (includesSubdomains ? "*.\(host)" : host) + (path ?? "") }

    init(raw: String) throws { /* normaliza, valida host e caminho; lança invalidDomain */ }
}

// Domain/Model/BlockedApp.swift
struct BlockedApp: Equatable, Codable, Hashable {
    let bundleID: String
    let displayName: String
    var isEnabled: Bool
}

// Domain/Model/BlockList.swift — agregado
struct BlockList: Equatable, Codable {
    var domains: [BlockedDomain]
    var apps: [BlockedApp]
    var activeDomains: [BlockedDomain] { domains }          // extensível p/ enable/disable
    var activeAppBundleIDs: Set<String> { Set(apps.filter(\.isEnabled).map(\.bundleID)) }
}

// Domain/Model/PomodoroConfiguration.swift
struct PomodoroConfiguration: Equatable, Codable {
    var focusDuration: TimeInterval = 25 * 60
    var shortBreakDuration: TimeInterval = 5 * 60
    var longBreakDuration: TimeInterval = 15 * 60
    var cyclesBeforeLongBreak: Int = 4
    var autoAdvancePhases: Bool = false       // chave persistida continua "autoStartNextFocus" (CodingKeys)
    var blockedRedirectURL: String? = nil     // site de redirecionamento; nil = blocked.html padrão
    var syncReminderCompletion: Bool = true   // write-back de conclusão no Lembretes
}
```

> `init(from:)` decodifica cada campo com `decodeIfPresent`/padrão — nunca deixa uma chave desconhecida ou ausente lançar erro. Se lançasse, o store cairia no valor padrão e apagaria TODAS as preferências do usuário em silêncio (armadilha já pega duas vezes: rename de `autoStartNextFocus` e chave órfã `hardcore`).

### 3.2 Ports (protocolos) — o coração do DIP

```swift
// Domain/Ports/AppBlocking.swift
protocol AppBlocking: AnyObject {
    func activate(blockedBundleIDs: Set<String>)
    func deactivate()
}

// Domain/Ports/WebsiteBlocking.swift
protocol WebsiteBlocking: AnyObject {
    func activate(domains: [BlockedDomain]) async throws
    func deactivate() async throws
    var isActive: Bool { get async }
}

// Domain/Ports/SessionClock.swift — nunca use Date()/Timer direto no domínio
protocol SessionClock: AnyObject {
    var now: Date { get }
    func schedule(every interval: TimeInterval, _ tick: @escaping () -> Void) -> ClockSubscription
}
protocol ClockSubscription { func cancel() }

// Domain/Ports/SessionRepository.swift — failsafe (RF-02.5 / UC-04)
protocol SessionRepository: AnyObject {
    func saveActive(_ session: PomodoroSession) throws
    func loadActive() throws -> PomodoroSession?
    func clearActive() throws
    func appendToHistory(_ record: SessionRecord) throws
}

// Domain/Ports/SettingsRepository.swift
protocol SettingsRepository: AnyObject {
    func loadConfiguration() -> PomodoroConfiguration
    func save(_ configuration: PomodoroConfiguration)
    func loadBlockList() -> BlockList
    func save(_ blockList: BlockList)
}

// Domain/Ports/UserNotifying.swift
protocol UserNotifying: AnyObject {
    func notify(_ event: NotificationEvent)   // .focusEnded, .breakEnded, .appBlocked(name:)
}

// Domain/Ports/TaskRepository.swift + TaskImporting.swift (RF-09)
protocol TaskRepository: AnyObject {
    func loadAll() throws -> [FocusTask]
    func save(_ tasks: [FocusTask]) throws
}
protocol TaskImporting {
    func fetchIncompleteReminders(fromLists: Set<String>?) async throws -> [ImportedReminder]
    func setReminderCompleted(reminderID: String, completed: Bool) async -> Bool   // write-back best-effort
}
```

> `PrivilegeEscalating` (rodar comando com privilégio de admin) existiu até o ADR-8 — foi removido junto com o mecanismo de `/etc/hosts`/XPC helper, sem substituto: o bloqueio de sites atual não precisa de privilégio nenhum, só da permissão de Automação do navegador (TCC).

**ISP em ação:** note que não existe um "SystemService" gordo. Cada consumidor enxerga só o que usa: o caso de uso de iniciar sessão recebe `AppBlocking` + `WebsiteBlocking`; a view de configurações recebe só `SettingsRepository`.

---

## 4. Application — casos de uso

Um caso de uso = uma classe pequena com um único método público (`SRP` + Clean Code: funções fazem uma coisa). Nomes de caso de uso são verbos do UC correspondente na especificação.

```swift
// Application/UseCases/StartFocusSessionUseCase.swift  (UC-01)
final class StartFocusSessionUseCase {
    private let clock: SessionClock
    private let appBlocker: AppBlocking
    private let websiteBlocker: WebsiteBlocking
    private let sessions: SessionRepository
    private let settings: SettingsRepository

    init(clock: SessionClock, appBlocker: AppBlocking, websiteBlocker: WebsiteBlocking,
         sessions: SessionRepository, settings: SettingsRepository) { /* ... */ }

    struct Outcome { let session: PomodoroSession; let websiteBlockFailed: Bool }

    func execute(taskIDs: [UUID], cycleNumber: Int) async -> Outcome {
        let config = settings.loadConfiguration()
        let blockList = settings.loadBlockList()
        let session = PomodoroSession(
            id: UUID(), phase: .focus,
            startedAt: clock.now,
            endsAt: clock.now.addingTimeInterval(config.focusDuration),
            cycleNumber: cycleNumber
        )

        appBlocker.activate(blockedBundleIDs: blockList.activeAppBundleIDs)

        var websiteBlockFailed = false
        do { try await websiteBlocker.activate(domains: blockList.activeDomains) }
        catch { websiteBlockFailed = true }     // UC-01 fluxo alternativo 3a — decisão sobe p/ UI

        try? sessions.saveActive(session)        // failsafe persistido ANTES do timer rodar
        return Outcome(session: session, websiteBlockFailed: websiteBlockFailed)
    }
}
```

> `reason`/`requireReason` existiram enquanto o modo hardcore existia — removidos do Domain e da Application em 2026-08-03 junto com o resto do modo hardcore (ver especificação RF-06).

Demais casos de uso, todos no mesmo formato (dependências por protocolo, um `execute`):

| Caso de uso | UC/RF | Responsabilidade única |
|---|---|---|
| `StartFocusSessionUseCase` | UC-01 | Criar sessão, ativar bloqueios, persistir failsafe |
| `CompleteSessionUseCase` | UC-02 | Desativar bloqueios, gravar histórico, decidir próximo phase |
| `CancelSessionUseCase` | UC-03 | Desativar bloqueios, gravar cancelamento |
| `RecoverFromCrashUseCase` | UC-04 | Ler sessão ativa persistida e decidir: encerrar (já expirou) ou oferecer retomada |
| `ManageBlockListUseCase` | UC-05 | Validar/normalizar domínio, CRUD das listas |
| `ManageTasksUseCase` | RF-09 | CRUD de tarefas, import/write-back com o Lembretes, dedup por `reminderID` |
| `ReportBuilder` (função pura, não use case) | RF-10 | Horas por tarefa/dia, intervalos, resumo com streak, a partir do histórico |

**A máquina de estados** (`SessionStateMachine`, em Application) é pura: recebe `(estadoAtual, evento, config, now)` e devolve `(novoEstado, efeitos)`. Os efeitos (`.activateBlocking`, `.deactivateBlocking`, `.notify(...)`, `.persist(...)`) são interpretados por um `SessionCoordinator` — isso torna as transições 100% testáveis por tabela de casos.

---

## 5. Infrastructure — adapters do macOS

Cada adapter implementa exatamente um port. Nenhum adapter conhece outro adapter (comunicação só via Domain/Application).

| Adapter | Port | Tecnologia | Observações |
|---|---|---|---|
| `WorkspaceAppBlocker` | `AppBlocking` | `NSWorkspace` + `didLaunchApplicationNotification` | `terminate()` padrão; `forceTerminate()` opt-in (RF-03.2); `activate` chama `deactivate()` primeiro (evita vazar observer) |
| `AppleScriptBrowserBlocker` | `WebsiteBlocking` | `NSAppleScript`/Apple Events, `DispatchSourceTimer` (~2s) | ADR-8. Varre abas dos navegadores rodando e redireciona; nunca abre navegador fechado; `deniedBundleIDs` isola falha de permissão por navegador |
| `DispatchSessionClock` | `SessionClock` | `DispatchSourceTimer` | Em testes: `FakeClock` com avanço manual |
| `SwiftDataTaskStore` | `TaskRepository` | SwiftData (`Tomafoco.store`) | ADR-11. Espelho em memória: `loadTasks` sem decode após a 1ª leitura; `saveTasks` (estado completo) grava só as linhas que mudaram; `upsert` por item |
| `SwiftDataProjectStore` | `ProjectRepository` | SwiftData | FEAT-006 |
| `SwiftDataSessionStore` | `SessionRepository` | SwiftData | Sessão ativa (failsafe) gravada na hora, sem autosave; histórico só por inserção |
| `SwiftDataKanbanStore` / `SwiftDataTagCatalog` | `KanbanBoardRepository` / `TagCatalog` | SwiftData | Quadro numa linha única; catálogo com ordem preservada |
| `TomafocoDatabase` | `UnitOfWork` | `ModelContainer` + `mainContext` | `perform` = uma transação (`save` no fim, `rollback` em erro); banco que não abre vira `.bak-<data>` |
| `JSONToSwiftDataImporter` | — | lê com os `File*Store` antigos | Migração única, idempotente e transacional; JSON renomeados para `*.json.migrated` só depois do `save()` |
| `FileTaskStore` / `FileKanbanStore` / `FileSessionSnapshotStore` | (leitores) | JSON em `Application Support` | Só servem ao importador — remover numa release futura |
| `EventKitReminderImporter` | `TaskImporting` | `EventKit` (`EKEventStore`) | `requestFullAccessToReminders`; lembrete "dia inteiro" vira 23:59 local (`ReminderDueDate`); write-back best-effort |
| `NSOpenPanelApplicationPicker` | `ApplicationPicking` | `NSOpenPanel` ancorado em `/Applications` | Seleção múltipla; bundle sem ID legível vira aviso, nunca é descartado em silêncio |
| `SMAppServiceLoginItem` | `LoginItemManaging` | `SMAppService.mainApp` | Estado não fica em `PomodoroConfiguration` — fonte da verdade é o sistema |
| `UserDefaultsSettingsStore` | `SettingsRepository` | `UserDefaults` + `Codable` | `decodeIfPresent` em todo campo — chave nova ou obsoleta nunca reseta a config inteira |
| `UNNotificationAdapter` | `UserNotifying` | `UNUserNotificationCenter` | Degradação graciosa se permissão negada (RF-08.2); som toca mesmo sem permissão |

> **Histórico (pré-ADR-8, removido em 2026-07-22):** `HostsFileWebsiteBlocker` (bloco delimitado em `/etc/hosts` + `dscacheutil -flushcache`, escrita via `AppleScriptPrivilegeRunner`/`PrivilegeEscalating` com prompt de administrador) e depois `XPCWebsiteBlocker` (privileged helper via `SMAppService.daemon`). Ambos saíram do código — motivo detalhado em `CLAUDE.md` (resumo: `/etc/hosts` é frágil com VPN corporativa/DNS-over-HTTPS; `mDNSResponder` não relia o arquivo de forma confiável no macOS atual).

---

## 6. Presentation — SwiftUI

- **ViewModels** (`@Observable`, macOS 14+ / `ObservableObject` no 13) recebem casos de uso por injeção; nunca tocam em ports de infraestrutura diretamente.
- `TimerViewModel`, `SettingsViewModel`, `BlockListViewModel`, `TasksViewModel`, `ReportsViewModel` — um por tela (SRP na UI).
- `MenuBarExtra` (SwiftUI, estilo `.window`) é a **única** casa do timer: o popover mostra a `MainView` inteira (RF-04). Não há `WindowGroup` — só as janelas de Tarefas e Relatórios (`Window`) e `Settings`.
- App `LSUIElement`: o `ActivationPolicyController` (App/) alterna `.accessory` ↔ `.regular` conforme haja janela aberta, para que essas janelas tenham menu superior (⌘C/⌘V/⌘W).
- Formatação de tempo, strings localizáveis (pt-BR primeiro, RNF-08) e acessibilidade (RNF-09) resolvidas nesta camada — nunca no domínio.

---

## 7. Composition Root

Único lugar onde tudo se conecta (e único lugar com `import AppKit` + construtores concretos):

```swift
@main
struct TomafocoApp: App {
    @State private var container = AppContainer.live()

    var body: some Scene {
        MenuBarExtra { MainView(viewModel: container.timerViewModel, updater: container.updater) }
            label: { /* tomate + tempo */ }
        Window("Tarefas", id: "tasks") { TasksView(viewModel: container.tasksViewModel) }
        Window("Relatórios", id: "reports") { ReportsView(viewModel: container.reportsViewModel) }
        Settings { SettingsView(viewModel: container.settingsViewModel) }
    }
}

enum AppContainer {
    static func live() -> Container {
        let privilege = AppleScriptPrivilegeRunner()
        let websiteBlocker = HostsFileWebsiteBlocker(privilege: privilege)
        let appBlocker = WorkspaceAppBlocker()
        let clock = DispatchSessionClock()
        let sessions = FileSessionSnapshotStore()
        let settings = UserDefaultsSettingsStore()
        // ... monta casos de uso e view models
    }
    static func test() -> Container { /* fakes para testes de UI/preview */ }
}
```

---

## 8. Mapeamento SOLID (resumo auditável)

| Princípio | Onde está aplicado |
|---|---|
| **S — Single Responsibility** | Um caso de uso por operação; um adapter por tecnologia; um ViewModel por tela; `SessionStateMachine` só transiciona estados — quem executa efeitos é o `SessionCoordinator`. |
| **O — Open/Closed** | Trocar `AppleScriptBrowserBlocker` → `NetworkExtensionBlocker` (v3, se um dia precisar bloquear por URL) não altera Domain/Application/Presentation — só o Composition Root. Foi assim que o ADR-8 substituiu `HostsFileWebsiteBlocker` sem tocar em Domain/Application. |
| **L — Liskov Substitution** | Todos os fakes de teste (`FakeClock`, `InMemorySessionRepository`, `SpyAppBlocker`, fakes de `AppleScriptRunning`) são substituíveis pelos adapters reais sem quebrar contratos; contratos documentados nos ports. |
| **I — Interface Segregation** | Ports pequenos e específicos (`AppBlocking` ≠ `WebsiteBlocking` ≠ `TaskImporting`); nenhum consumidor depende de método que não usa. |
| **D — Dependency Inversion** | Application depende de abstrações do Domain; Infrastructure implementa essas abstrações; injeção 100% no Composition Root — nenhum `singleton` global, nenhum `.shared` fora dos adapters. |

**Clean Code (convenções do projeto):** funções curtas com um nível de abstração; nomes revelam intenção (`isExpired(now:)`, não `check()`); sem números mágicos (durações vêm de `PomodoroConfiguration`); erros são tipos (`DomainError`) e nunca `fatalError` em fluxo de produção; comentários apenas para "porquês" (ex.: por que a escrita do hosts é idempotente); testes nomeados como comportamento (`test_cancel_desativaBloqueioEVoltaParaIdle`).

---

## 9. Estratégia de testes (RNF-05)

| Camada | Tipo | Ferramenta | Meta |
|---|---|---|---|
| Domain | Unit puro (value objects, validações, `PomodoroSession`) | Swift Testing / XCTest | ~100% |
| Application | Unit com fakes (tabela de transições da state machine, cada use case) | Swift Testing + fakes in-memory | ≥ 80% |
| Infrastructure | Integração isolada: `AppleScriptBrowserBlocker` testado com fakes de `AppleScriptRunning`/`RunningApplicationsProviding` (nunca pilota um navegador real em CI); `EventKitReminderImporter`/`FileTaskStore` idem | XCTest | Cenários críticos de RNF-01 |
| Presentation | Snapshot/preview + smoke tests de ViewModel | XCTest | Fluxos principais |
| E2E manual | Roteiro de QA (UC-01..UC-05 + kill -9 durante o foco) | Checklist de QA do mantenedor | Antes de cada release |

Ponto-chave: `AppleScriptBrowserBlocker` recebe os ports `AppleScriptRunning`/`RunningApplicationsProviding` **por injeção**, permitindo testar todo o casamento de URL (`URLBlockingPolicy`, puro no Domain) e a lógica de retry/permissão negada sem tocar num navegador real.

---

## 10. Estrutura de pastas (Xcode / SPM)

O núcleo vai em pacotes SPM locais para reforçar as fronteiras em tempo de compilação (Domain não consegue importar AppKit nem por acidente):

```
Tomafoco/
├── Tomafoco.xcodeproj
├── App/                            # target macOS (Composition Root + Presentation)
│   ├── TomafocoApp.swift
│   ├── AppContainer.swift
│   └── UI/
│       ├── Timer/          (MainView — conteúdo do popover, TimerViewModel)
│       ├── Tasks/          (TasksView, TasksViewModel — janela "Tarefas")
│       ├── Reports/        (ReportsView, ReportsViewModel — janela "Relatórios", Swift Charts)
│       ├── Settings/       (SettingsView, BlockListViews, ViewModels)
│       ├── BreakOverlay/   (tela cheia de intervalo, uma NSWindow por monitor)
│       ├── BlockedAppToast/(aviso topo-de-tela ao tentar abrir app bloqueado)
│       ├── Recovery/       (RecoveryAlert — NSAlert pós-crash)
│       ├── Updates/        (UpdaterController — fachada sobre o Sparkle)
│       ├── Shared/
│       └── DesignSystem/   (Brand.swift, Components.swift, TomatoMark.swift)
├── Packages/
│   ├── TomafocoDomain/              # sem dependências
│   │   └── Sources/ (Model/, Ports/, Errors/)
│   ├── TomafocoApplication/         # depende só de TomafocoDomain
│   │   └── Sources/ (UseCases/, StateMachine/, Coordinator/)
│   └── TomafocoInfrastructure/      # depende de TomafocoDomain (+ AppKit, UserNotifications)
│       └── Sources/ (Blocking/, Privilege/, Persistence/, Notifications/, Clock/)
├── Tests/  (espelha os pacotes + fakes compartilhados em TomafocoTestSupport)
└── docs/   (este documento, release.md, release-notes/, screenshots/)
```

---

## 11. Decisões de arquitetura (ADR resumido)

| # | Decisão | Alternativa rejeitada | Motivo |
|---|---|---|---|
| ADR-1 | Swift/SwiftUI nativo | .NET MAUI + shim, Flutter, Electron | Bloqueio exige APIs nativas; interop adicionaria complexidade sem benefício no macOS-only |
| ADR-2 | Distribuição fora da App Store | Mac App Store | Sandbox inviabiliza edição de hosts e término de apps |
| ADR-3 | `/etc/hosts` no MVP — **superado pelo ADR-8** | NetworkExtension Content Filter | NE exige entitlement aprovado pela Apple + system extension; custo desproporcional para v1. Hosts em si mostrou-se frágil demais (ver ADR-8) e foi abandonado |
| ADR-4 | `osascript` admin no MVP → helper XPC (`SMAppService.daemon`) — **removido no ADR-8** | Privileged helper desde o início | Chegou a ser implementado (2026-07-22) para eliminar prompts repetidos, mas todo o mecanismo de hosts caiu junto com o pivô do ADR-8 |
| ADR-5 | Pacotes SPM por camada | Grupos de pastas num único target | Fronteiras verificadas pelo compilador, não por disciplina |
| ADR-6 | Estado com horário absoluto (`endsAt`) | Contador decremental em memória | Sobrevive a crash/sleep/reboot (RF-01.5, RNF-02) |
| ADR-7 | Modo hardcore — **implementado e depois removido (2026-08-03)** | Manter a trava com carência/motivo obrigatório | O produto é de autodisciplina, não de coerção; a trava atrapalhava interrupção legítima e Force Quit não é bloqueável sem root de qualquer forma. Ver especificação RF-06 |
| ADR-8 | Bloqueio de sites por **automação de navegador** (AppleScript/Apple Events), 2026-07-22 — mecanismo atual e definitivo | Manter `/etc/hosts` (ADR-3/4) | `dscacheutil -flushcache` não recarrega o `mDNSResponder`; `killall -HUP mDNSResponder` falha via `do shell script` (SIP); hosts é frágil com VPN corporativa + DNS-over-HTTPS. Pilotar o navegador direto (como fazem apps concorrentes) não depende de rede/DNS e não pede senha de admin |
| ADR-9 | Atualização automática com Sparkle 2 | Update caseiro (checar JSON + baixar DMG), pedir download manual no site | Fora da App Store não há atualização do sistema (ADR-2). Sparkle é o padrão de fato do macOS: já resolve assinatura EdDSA do feed, verificação do Developer ID do pacote baixado, instalação com reinício e retomada de download. Um updater caseiro teria que reimplementar isso — e um erro na verificação vira execução de código arbitrário na máquina do usuário |
| ADR-10 | Bloqueio contínuo por **decoradores dos ports** (`PersistentAppBlocker`/`PersistentWebsiteBlocker`) + senha de desbloqueio com gate puro (`AppLaunchGate`) e PBKDF2 no Keychain (FEAT-002, 2026-09-29) | Novos estados/efeitos na `SessionStateMachine`; senha em `UserDefaults`; liberação por bundle ID | Coordinator, recuperação e máquina de estados ficam intactos — a única mudança é "o fim do foco derruba o bloqueio, salvo flag ligada", concentrada no decorador. O gate mora no Domain porque a Infrastructure não depende da Application. Liberação chaveada por **PID**. O app é **escondido** (não encerrado) durante o prompt: no `didLaunchApplicationNotification` o app ainda não terminou de carregar e costuma ignorar `terminate()` — encerrar + relançar falhava na prática |
| ADR-11 | **Persistência em SwiftData** (FEAT-006, 2026-10-29): `@Model` só na Infrastructure, cada linha com o **payload Codable da entidade do Domain** + colunas de busca/ordem; esquema versionado (`TomafocoSchemaV1` + `TomafocoMigrationPlan`); ports e use cases de persistência `@MainActor` (o `mainContext` não é `Sendable`) | Manter JSON com cache em memória; GRDB/SQLite direto | Decisão do usuário por banco, com framework nativo e sem dependência externa. Payload Codable mantém o Domain puro, não perde campo no mapeamento e dispensa migração de esquema quando o Domain ganha campo. Custo: macOS 14 mínimo e isolamento na main. Medido com os dados reais (95 tarefas, 302 sessões): migração 29 ms; reler tarefas 1,9 ms (JSON) → 0,02 ms; anexar sessão 3,1 ms → 0,45 ms |

### ADR-9 — detalhes

- **Feed:** `https://tomafoco.dds.tec.br/downloads/appcast.xml` (`SUFeedURL` no `project.yml`).
- **Confiança:** o appcast é assinado com chave **EdDSA**; a pública vai no `SUPublicEDKey`, a
  privada fica no **chaveiro** do mantenedor. Servidor comprometido, DNS envenenado ou proxy
  hostil não bastam para entregar um update forjado. Além disso o Sparkle confere que o app
  baixado tem o mesmo Developer ID do app instalado.
- **Sem instalação silenciosa:** `SUAutomaticallyUpdate: false`. Checa 1×/dia em segundo plano e
  pergunta. Num app de foco, trocar a versão sozinho pode derrubar uma sessão em andamento.
- **Onde mora no código:** `App/UI/Updates/UpdaterController.swift` — fachada `@MainActor` sobre
  o `SPUStandardUpdaterController`. Fica no `App/`, não na Infrastructure, porque o Sparkle traz
  a própria UI; não há port no Domain porque não há regra de domínio envolvida.
- **Fonte da verdade do "checar automaticamente" é o Sparkle** (`UserDefaults` /
  `SUEnableAutomaticChecks`), não `PomodoroConfiguration`: quem faz a pergunta de primeira
  execução é o próprio Sparkle, então uma cópia nossa divergiria da resposta do usuário.
- **Assinatura de código:** com o Sparkle embutido, o app passa a ter bundles executáveis
  aninhados (`Sparkle.framework`, `Updater.app`, `Autoupdate`, XPC services). Cada um precisa
  de assinatura própria, **de dentro para fora** — `scripts/release.sh` faz isso com
  `find -depth` antes de assinar o `.app`. Sem isso a notarização rejeita.
