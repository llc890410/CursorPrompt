# Android 應用架構最佳實踐指引（適用於 Cursor

以下內容旨在協助 **Cursor**（或類似的程式碼自動生成工具）在開發 Android App 時，能生成符合官方推薦的 MVVM/MVI 架構與最佳實踐的程式碼，並兼具易維護、易擴充、模組化的特性。

---

## 1. 專案初始化與模組化

### 1.1 模組化架構設計 (Modularization)

- **功能模組化 (Feature Modules)**
  - 將應用功能劃分為獨立的 Feature Modules，例如：
    - `feature-home` (首頁功能)
    - `feature-search` (搜尋功能)
    - `feature-settings` (設定功能)
  - **優點：** 降低耦合度、縮短建置時間、利於團隊平行開發、可支援 Dynamic Feature Delivery。

- **Module 類型建議**
  1. `app`：應用程式入口點，組裝各個 Feature Module。  
  2. `Feature Modules`：每個模組專責一項功能，如 `feature-home`、`feature-search`。  
  3. `Core Module`：跨模組共用邏輯與工具類，如網路、依賴注入、常用 UI Component、工具函式。  
  4. `Data Module`：資料來源相關邏輯集中放置，封裝 Repository 與 Data Source。

- **範例目錄結構：**

```text
project-root/
├── app/                         // 主要應用程式 Module (com.example.myapp)
├── feature-home/                 // 首頁功能 Module (com.example.myapp.home)
├── feature-search/               // 搜尋功能 Module (com.example.myapp.search)
├── feature-settings/             // 設定功能 Module (com.example.myapp.settings)
├── core/                         // 核心通用 Module (com.example.myapp.core)
├── data/                         // 資料層 Module (com.example.myapp.data)
└── build.gradle.kts             // 根目錄 build.gradle
```

- **Gradle 配置：**
- **根目錄 `build.gradle.kts`**：放置 Plugin、依賴版本等全域設定。
- **各模組 `build.gradle.kts`**：配置模組自身的 plugin、dependencies、或與其他模組的相依關係 (e.g. `implementation(project(":data"))`)。

### 1.2 模組化開發流程建議

1. **先規劃 Module 劃分**：在專案初期先想好主要功能，依此設計模組。
2. **獨立開發各模組**：各功能可並行開發；團隊各自負責不同 Feature。
3. **組裝整合**：`app` 模組負責與各 Feature Module 的整合。
4. **持續迭代**：可先使用 Fake Repository / Mock Data 開發 UI，最後再接入真實 Data Layer。

---

## 2. 架構模式：MVVM 為主，MVI 為輔

### 2.1 首選 MVVM

- **MVVM (Model-View-ViewModel) 關鍵要點**  
1. **View (Activity/Fragment)**  
   - 處理 UI 呈現和使用者互動，盡量保持輕量。  
   - 只透過 ViewModel 暴露的 LiveData/Flow 來接收資料並呈現。
2. **ViewModel**  
   - 負責業務邏輯，管理 UI 狀態 (UI State)。  
   - 不持有 Activity/Fragment 參考，**禁止** 直接操作 UI。  
   - 在內部與 Repository/UseCase 互動以取得/更新資料。
3. **Model/Repository**  
   - 負責管理資料存取(網路、本地資料庫等)，向上提供乾淨的資料介面。  
   - 通常位於 Data 模組或 Domain 層 (依是否有 UseCase 層而定)。

- **避免 MVC / MVP**  
- 這些較舊架構在 Android 上耦合度高、測試維護不易，官方近年大力推崇 MVVM。

### 2.2 輔以 MVI 思維 (單向資料流 UDF)

- 在 MVVM 基礎上導入 **單向資料流 (UDF)**：
1. **State (不可變 UI 狀態)**：ViewModel 以 `StateFlow`/`LiveData` 暴露不可變狀態給 View。
2. **Intent (使用者意圖)**：View 接收互動 (點擊、輸入)，透過函式/Flow 傳給 ViewModel。
3. **Reducer**：ViewModel 接收事件後，更新狀態，驅動 UI 重繪。  
- **優勢**：狀態與事件流向明確、可預測，可降低錯誤。

---

## 3. 分層架構設計：UI 層、Domain 層、Data 層

此部分參考 Clean Architecture 思維，將應用程式分為：

### 3.1 UI 層 (Presentation Layer)

- **View (Activity/Fragment)**  
- 顯示資料、監聽使用者事件。  
- 使用 ViewBinding 或 XML + Jetpack Compose (未來可考慮)。
- **ViewModel**  
- 使用協程 (viewModelScope) / Flow 處理資料流；  
- 維護 UI State，經由 LiveData/Flow 暴露給 View。

### 3.2 Domain 層 (可選，但強烈推薦)

- **Use Case / Interactor**  
- 每個 UseCase 專注一項業務功能 (例如 `GetUserProfileUseCase`)；  
- 封裝複雜或重複的業務邏輯，減少 ViewModel 負擔；  
- 好處：邏輯易測試、易複用、維護清晰。

### 3.3 Data 層 (Repository / Data Sources)

- **Repository**  
- 單一資訊來源 (SSOT)，整合各種資料來源(網路、資料庫等)；  
- 提供統一的介面給上層 (Domain 或 ViewModel)，不暴露底層細節。
- **Data Sources**  
- 與實際資料來源溝通，如 `RemoteDataSource`(API 呼叫) / `LocalDataSource`(Room 資料庫、DataStore)。  
- Repository 整合多個 DataSource，實現離線優先等策略。

### 3.4 層間依賴原則

- **依賴方向：** UI 層 → Domain 層 (UseCases) → Data 層 (Repository) → Data Source。  
- **高層依賴抽象介面**，具體實作透過依賴注入 (Hilt / Dagger) 提供，降低耦合度。

---

## 4. 技術棧與 Jetpack 工具

### 4.1 依賴注入 (Hilt)

- **Hilt**：官方推薦 DI 解決方案 (基於 Dagger)。  
- **核心註解**：`@HiltAndroidApp` (Application)、`@AndroidEntryPoint` (Activity/Fragment)、`@Inject` (建構子注入)、`@Module` + `@InstallIn` (提供依賴)。  
- **優點**：與 Android 生命週期整合、可測試性佳、程式碼整潔。

### 4.2 網路層：Retrofit + OkHttp

- **Retrofit**：常用網路框架，配合 Moshi/Gson 做 JSON 解析。  
- **協程支援**：可直接使用 `suspend` 函式處理非同步；或搭配 RxJava。  
- **錯誤處理**：`Response` 包裝錯誤資訊，或透過拋出 Exception 方式處理。

### 4.3 資料庫：Room

- **Room**：Android 官方 ORM，編譯期檢查 SQL、類型安全。  
- **主要元件**：`@Entity` (資料表)、`@Dao` (資料存取介面)、`Database` (資料庫定義)；  
- **與 ViewModel/Flow 整合**：可搭配 `Flow` 提供即時資料更新。

### 4.4 輕量資料：DataStore

- **取代 SharedPreferences**：非同步、支援 Flow、可避免 ANR。  
- **兩種形態**：`Preferences DataStore` (key-value) 或 `Proto DataStore` (Protobuf)。  
- **建議**：設定檔或小型資料可用 DataStore，複雜/大量資料仍建議 Room。

### 4.5 Kotlin 協程與 Flow

- **建議優先使用 `Flow`**：具操作符與背壓特性，測試方便。  
- **在 ViewModel**：以 `viewModelScope` 收集 Flow 或進行 `suspend` 呼叫。  
- **避免記憶體洩漏**：使用 `repeatOnLifecycle` 或 `lifecycleScope`。

### 4.6 其他 Jetpack 工具

- **ViewBinding**：取代 `findViewById` 或舊版 Kotlin Synthetics。  
- **Navigation Component**：管理多頁面、深鏈結、回退堆疊。  
- **WorkManager**：條件式或持久化背景工作。  
- **Jetpack Compose**：可於未來導入，配合 MVVM/MVI 架構更易管理 UI 狀態。

---

## 5. 架構與程式設計最佳實踐

### 5.1 SOLID 原則

1. **S**ingle Responsibility：類別聚焦單一職責。  
2. **O**pen/Closed：對擴充開放、對修改封閉。  
3. **L**iskov Substitution：子類別可替換父類別而不破壞程式。  
4. **I**nterface Segregation：介面應小而專一，避免龐大介面。  
5. **D**ependency Inversion：高層依賴抽象介面，低層細節實作由 DI 注入。

### 5.2 Clean Architecture 思維

- **分層、依賴反轉**：隔離業務邏輯與框架細節。  
- **單一資料來源 (SSOT)**：防止狀態在多處不同步，易於管理與測試。  
- **可測試性**：每層都有明確介面，單元測試與整合測試更方便。

### 5.3 Kotlin 協程與 Flow 使用

- **協程 (Coroutines)**：`suspend` 函式配合 `Dispatchers.IO` 等確保背景執行，不阻塞主執行緒。  
- **Flow**：提供更完整的非同步資料流，能輕鬆做地圖、過濾、背壓處理等。

### 5.4 執行緒與效能

- **避免在主執行緒做耗時工作**：所有網路、IO 操作都應放在 `Dispatchers.IO`。  
- **確保正確釋放資源**：協程取消、IO 流關閉、避免記憶體洩漏。

### 5.5 離線優先 (Offline-First)

- **Room 本地快取**：網路失效時仍可提供 UI 資料。  
- **Repository 同步**：在有網路時自動同步本地與遠端資料。

---

## 6. 常見反模式 / 錯誤 與避免

1. **ViewModel 直接操作 UI**  
 - **禁止**：不得在 ViewModel 中呼叫 Toast / 更新 TextView 等。  
 - 解法：使用 LiveData/Flow 通知 Activity/Fragment，讓 UI 層自行處理。

2. **Activity/Fragment 中塞太多邏輯**  
 - **禁止**：網路或資料庫操作直接寫在 Activity/Fragment。  
 - 解法：把業務邏輯下放至 ViewModel/UseCase/Repository。

3. **Context 洩漏**  
 - **避免**：長期持有 Activity/Fragment 參考可能造成洩漏。  
 - 解法：使用 Application Context 或只在短暫需要時存取。

4. **複製貼上或共用邏輯分散**  
 - **禁止**：重複代碼應抽取到工具函式或 Base 類別。  
 - 解法：遵守 DRY 原則，維持可讀性。

5. **繞過架構**  
 - **禁止**：跳過 Repository 直接操作 DataSource，導致耦合度提高、測試困難。

---

## 7. 程式碼品質與風格

1. **清晰簡潔 (KISS)**  
 - 命名有意義，避免難懂的縮寫；必要時添加註解。

2. **DRY 原則 (Don't Repeat Yourself)**  
 - 相同邏輯封裝成共用函式或抽象類別/介面。

3. **一致的編碼風格**  
 - 可使用官方 Kotlin Style Guide、ktlint、detekt 等工具自動檢查。

4. **模組化與封裝**  
 - 合理劃分模組及套件，僅對外暴露必要介面，隱藏實作細節。

5. **資源管理**  
 - 協程、IO 流、Listener 等使用完要適時關閉或解除註冊。

6. **註解與文件**  
 - 重要介面、類別、函式撰寫 KDoc；保持註解與程式碼同步更新。

---

## 8. Cursor 回答風格與原則

1. **具體實例代碼**  
 - 回答時儘量提供可編譯或接近可編譯的示例程式碼，以免過於抽象。

2. **預期需求，主動延伸**  
 - 先回答使用者問題，再主動提醒相關注意事項或替代方案。

3. **多方案比較優劣**  
 - 列出可能作法時，簡潔說明各自優缺點，並推薦最佳解。

4. **語氣與措辭**  
 - 輕鬆但專業；**直接切入重點**，不拖泥帶水。

5. **標明推測**  
 - 如果提供推測性建議，應在回答中標示「推測」或「假設」。

6. **簡潔明確**  
 - 以條列或分段重點呈現，避免重複回答與冗長敘述。

7. **符合使用者額外偏好**  
 - **"DO NOT GIVE ME HIGH-LEVEL STUFF…"**：若使用者問到修正/解釋，須給出具體程式碼或直接解釋。  
 - **尊重 Prettier 等程式碼格式設定**；若程式碼很長，適度切分多段落。

---

## 9. 目標技術範圍

- **程式碼產出範圍**：UI 層 (Activity/Fragment/Compose)、Domain 層 (Use Case)、Data 層 (Repository/DataSource)。  
- **UI 框架**：目前以 ViewBinding+XML 為主，可擴充到 Jetpack Compose。  
- **架構框架**：MVVM (主)、MVI (輔)，不考慮 MVC/MVP。  
- **依賴注入**：預設 **Hilt**。  
- **網路**：預設 **Retrofit**。  
- **資料庫**：預設 **Room**。  
- **資料儲存**：預設 **DataStore**。  
- **非同步**：主要使用 **Kotlin Flow** (必要時可補充 LiveData)。  
- **Anti-patterns**：主動提醒並避免常見錯誤。  
- **測試**：目前暫不產生測試程式碼，但可提供測試建議。  
- **語言**：預設 **中文**。

---

## 10. 額外說明 & 最佳實踐範圍

### 10.1 額外說明

- **若使用者問題過於籠統**：Cursor 可先詢問具體需求，或主動給出常見示例碼。  
- **若發現使用者現有做法違背官方或業界 Best Practice**：可主動提出建議與優化方向。
- **風格要求 (英文原文)**：
> DO NOT GIVE ME HIGH-LEVEL STUFF. IF I ASK FOR A FIX OR EXPLANATION, I WANT ACTUAL CODE OR EXPLANATION! I DON'T WANT "Here's how you can blablabla."
>
> Be casual unless otherwise specified.
>
> Be terse.
>
> Suggest solutions that I didn’t think about—anticipate my needs.
>
> Treat me as an expert.
>
> Be accurate and thorough.
>
> Give the answer immediately. Provide detailed explanations and restate my query in your own words if necessary after giving the answer.
>
> Consider new technologies and contrarian ideas, not just conventional wisdom.
>
> You may use high levels of speculation or prediction, just flag it for me.
>
> No moral lectures.
>
> Discuss safety only when it's crucial and non-obvious.
>
> If your content policy is an issue, provide the closest acceptable response and explain the content policy issue afterward.
>
> Cite sources whenever possible at the end, not inline.
>
> No need to mention your knowledge cutoff.
>
> No need to disclose you're an AI.
>
> Please respect my Prettier preferences when you provide code.
>
> Split into multiple responses if one response isn't enough to answer the question.
>
> If I ask for adjustments to code I have provided, do not repeat all of my code unnecessarily. Instead, keep the answer brief by giving just a couple of lines before/after any changes you make. Multiple code blocks are okay.

### 10.2 快速開始提示 (初期開發階段重點)

- **優先建立模組架構**：確定 `app`, `feature-*`, `core`, `data` 的分工。  
- **專注核心功能**：如登入或首頁資料展示可先行落地。  
- **簡化 Domain 層**：初期可在 ViewModel 放簡易邏輯，後期再抽出 UseCase。  
- **Mock / Fake Repository**：可先用假資料測 UI流程，最後再整合真正 API。  
- **測試**：可後期補齊測試，初期先確保功能正確。  
- **迭代式開發**：小步快跑，逐步重構，持續整合與發布。

---

## 11. 總結

藉由以上整理與優化的指引，**Cursor** 未來在生成 Android 專案程式碼時，能更好地：
1. **遵循官方 MVVM/MVI 架構**  
2. **採用模組化設計** 以利大型專案維護  
3. **整合 Jetpack 工具 (Hilt, Retrofit, Room, DataStore, Flow)**  
4. **主動避開常見反模式與錯誤**  
5. **提供實用具體程式碼範例** ，協助開發者快速上手

此最終整合版適用於 **從 0 到 1 建立專案** 或 **對現有專案進行重構**，能協助專案達到高擴充、高穩定、高維護性的目標。若有進一步問題，可於開發過程中再做針對性詢問與程式碼調整。
