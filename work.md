# **Strict-Ts 專案實施操作步驟**

本文件旨在提供一個中立的、可執行的技術實施路徑，以指導 `Strict-Ts` 專案的開發。

## **準備階段：專案初始化與基礎設施**

1.  **環境搭建**
    1.1. 初始化 Node.js 專案 (`npm init`)。
    1.2. 安裝 TypeScript 作為開發依賴 (`npm i -D typescript`) 並配置 `tsconfig.json`。
    1.3. 選擇並配置一個測試框架，如 `Vitest` 或 `Jest` (`npm i -D vitest`)。
    1.4. 安裝並配置 `ESLint` 和 `Prettier` 以保證程式碼品質。

2.  **核心依賴安裝**
    2.1. 安裝 AST 解析器：`npm i @typescript-eslint/typescript-estree`。
    2.2. 安裝 AST 遍歷工具：`npm i estraverse` 或 `@babel/traverse`。
    2.3. 安裝程式碼生成器：`npm i escodegen` 或 `@babel/generator`。
    2.4. 安裝 Source Map 處理工具：`npm i source-map`。

---

## **里程碑 1: 核心批次處理引擎 (MVP)**

**目標：** 創建一個能夠接收單一 TypeScript 文件、進行基礎優化並輸出 JavaScript 的核心轉譯器。

3.  **步驟 1：建立核心轉譯流程**
    3.1. 設計一個主 `Compiler` 或 `Transpiler` 類，作為所有操作的協調器。
    3.2. 實現一個 `transpile(sourceCode: string, fileName: string): { code: string, map: object }` 方法。
    3.3. 在此方法中，集成以下子流程：
        3.3.1. 使用 `@typescript-eslint/typescript-estree` 將 `sourceCode` 解析為 AST。
        3.3.2. **佔位 (Placeholder):** 建立一個空的 `transform(ast)` 函數，目前僅返回原始 AST。
        3.3.3. 使用程式碼生成器將 AST 轉換回 JavaScript 程式碼字串。
        3.3.4. **佔位:** 實現基礎的 Source Map 生成邏輯，確保輸入行號能對應到輸出。

4.  **步驟 2：實現類型資訊獲取**
    4.1. 修改主流程，使其接收一個 `typescript.Program` 物件。這需要使用 `typescript` 的 API 創建一個虛擬的檔案系統或基於真實文件來構建 `Program`。
    4.2. 從 `Program` 中獲取 `TypeChecker` (`program.getTypeChecker()`)。
    4.3. 開發一個 `TypeService` 或工具類，它接收 `TypeChecker` 和一個 AST 節點，能夠返回該節點的 TypeScript `Type` 物件。

5.  **步驟 3：實現語義分析與標記**
    5.1. 實現一個 AST 遍歷器（Visitor/Traverser），它將遍歷整個 AST。
    5.2. 在遍歷過程中，為每個節點增加一個 `meta` 或 `symbol` 屬性，用於存儲 `Strict-Ts` 的分析結果。
    5.3. 針對 `as` 表達式 (`TSAsExpression`)：
        5.3.1. 獲取表達式本身的類型和斷言目標的類型。
        5.3.2. 實現一個 `isUpcasting(sourceType, targetType)` 函數，判斷是否為安全的向上斷言。
        5.3.3. 在節點的 `meta` 中標記其為 `SAFE_CAST` 或 `UNSAFE_DOWNCAST`。
    5.4. 針對 `any` 和 `unknown` 類型：
        5.4.1. 識別出類型為 `any` 或 `unknown` 的變數聲明或表達式。
        5.4.2. 在節點的 `meta` 中標記其類型來源（`ANY_ORIGIN`, `UNKNOWN_ORIGIN`）。

6.  **步驟 4：實現語句級死碼消除 (Branch Pruning)**
    6.1. 擴展 AST 遍歷器，使其能處理 `IfStatement`。
    6.2. 分析 `IfStatement` 的 `test` 表達式。
    6.3. **初期實現：** 如果 `test` 是一個布林字面量 (`true` 或 `false`)，則標記相應的分支（`consequent` 或 `alternate`）為 `DEAD_CODE`。
    6.4. 修改程式碼生成階段的邏輯，使其在遇到被標記為 `DEAD_CODE` 的節點時，跳過該節點的生成。

7.  **步驟 5：整合與測試**
    7.1. 創建一系列 `.ts` 測試文件，包含簡單的類型斷言和 `if(true/false)` 分支。
    7.2. 編寫單元測試或快照測試，斷言 `transpile` 方法的輸出符合預期（例如，`else` 分支被移除）。

---

## **里程碑 2: 全域分析與 Tree-shaking**

**目標：** 將分析範圍從單一文件擴展到整個專案，並實現模組級的死碼消除。

8.  **步驟 6：設計與實現依賴圖**
    8.1. 設計代表「符號」（函式、類別、變數）的資料結構 `SymbolNode`。它應包含名稱、來源檔案、AST 節點引用、類型資訊等。
    8.2. 設計代表整個專案的 `DependencyGraph` 類。它應能存儲所有 `SymbolNode` 以及它們之間的依賴關係（邊）。
    8.3. 依賴關係（邊）應包含依賴類型（例如，`CALL`, `REFERENCE`, `IMPORT`）。

9.  **步驟 7：實現全域符號分析**
    9.1. 修改主流程，使其能夠接收一個檔案列表（入口點）。
    9.2. 對於專案中的每個文件，執行 AST 解析和符號提取。
        9.2.1. 遍歷每個檔案的 AST，識別出所有的 `export` 和頂層聲明，為其創建 `SymbolNode` 並加入圖中。
        9.2.2. 識別 `import` 語句，在圖中建立檔案之間的依賴邊。
        9.2.3. 在函數體或類別內部，識別對其他符號的引用，並在圖中建立更細粒度的依賴邊。

10. **步驟 8：實現可達性分析 (Tree-shaking)**
    10.1. 實現一個 `markReachable(graph, entryPoints)` 演算法。
    10.2. 從入口點文件開始，遞歸地遍歷依賴圖。
    10.3. 為所有訪問到的 `SymbolNode` 添加一個 `isReachable = true` 的標記。
    10.4. 在死碼標記階段，將所有 `isReachable` 不為 `true` 的頂層聲明（及其對應的 AST 節點）標記為 `DEAD_CODE`。

11. **步驟 9：完善 Source Map 生成**
    11.1. 在多個 AST 轉換階段（例如，分支剪除、搖樹）之間，確保 Source Map 的原始位置信息能夠被正確地傳遞和更新。
    11.2. 研究並使用 `source-map` 庫的 `SourceMapConsumer` 和 `SourceMapGenerator` 來處理這種鏈式轉換。

---

## **里程碑 3: 增量計算與開發伺服器整合**

**目標：** 實現檔案變動後的增量更新能力。

12. **步驟 10：實現快取機制**
    12.1. 設計分析結果的快取策略。快取的鍵可以是檔案路徑，值可以是該檔案的 AST、符號列表、依賴關係等分析結果。
    12.2. 在每次分析前，檢查檔案的修改時間或內容雜湊（hash）。如果未變動，則從快取中讀取結果。

13. **步驟 11：實現增量更新演算法**
    13.1. 監聽檔案系統的變動事件（新增、修改、刪除）。
    13.2. 當一個檔案 `A` 發生變動時：
        13.2.1. 使 `A` 的快取失效，並重新分析 `A`，更新其在依賴圖中的節點和匯出邊。
        13.2.2. 找出所有依賴 `A` 的檔案（`B`, `C`, ...）。
        13.2.3. 遞歸地對 `B`, `C` 等執行增量分析，檢查它們的依賴是否仍然有效，並更新它們的分析結果，直到依賴鏈中沒有更多的變動。
    13.3. 重新執行全局的可達性分析。

14. **步驟 12：開發整合插件**
    14.1. 選擇一個目標建構工具（如 Vite）。
    14.2. 閱讀其插件開發文檔。
    14.3. 創建一個 Vite 插件，它在 `transform` 鉤子中調用 `Strict-Ts` 的核心轉譯邏輯。
    14.4. 將 `Strict-Ts` 的檔案監聽和增量更新能力與 Vite 的開發伺服器和 HMR 機制對接。

---

## **里程碑 4: 高級優化與生態完善**

**目標：** 實現設計文檔中所有高級功能和配置選項。

15. **步驟 13：實現配置系統**
    15.1. 使用 `cosmiconfig` 或類似工具來讀取專案中的配置文件 (`tsconfig.json` 或 `strict-ts.config.json`)。
    15.2. 將讀取到的配置傳遞給編譯器的各個模組。
    15.3. 在分析和標記的各個階段，根據配置項（如 `downcastingTransform`, `unreachableBranch`）執行不同的邏輯或報告不同級別的診斷資訊。

16. **步驟 14：實現高級契約和優化**
    16.1. **`UNSAFE_` 註解：** 在語義分析階段，識別並處理各種 `UNSAFE_` 註解和前綴，使其能夠覆蓋預設的分析行為。
    16.2. **斷言函數檢查：** 實現對 `is T` 類型斷言函數的函數體分析，驗證其檢查的完備性。
    16.3. **`enum` 優化：** 實現對 `enum` 使用情況的全域追蹤，判斷是否需要生成反向查找，並在適當時機將其內聯。
    16.4. **屬性移除：** 擴展依賴圖分析，使其能追蹤類別成員的訪問。如果一個屬性或方法從未被訪問，則在程式碼生成時將其移除。

17. **步驟 15：文檔與發布**
    17.1. 撰寫詳細的 README 和使用者文檔，解釋所有功能、契約和配置選項。
    17.2. 設置 CI/CD 流程，自動運行測試並發布到 NPM。