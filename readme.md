# Strict-Ts

Strict-Ts 是一個專注於生成高安全性與小體積 JavaScript 的 TypeScript 轉譯器。其設計目標不僅是作為一個獨立的編譯器，更旨在與 Webpack、Vite 等現代建構工具生態無縫兼容。

## 核心哲學：相互保證

Strict-Ts 的強大優化能力，建立在開發者與編譯器之間的一組「相互保證」契約之上。此哲學的核心是：開發者透過遵循特定的編碼原則來清晰地表達意圖，作為回報，Strict-Ts 利用 TypeScript 的型別資訊，對輸出的 JavaScript 進行極致的分析與優化。

## 開發者契約 

為了啟動 Strict-Ts 的優化引擎，開發者需要遵循以下編碼契約。

### `any` 與 `unknown` 的嚴格語義區分

Strict-Ts 對 `any` 與 `unknown` 的語義進行了區分，以提升靜態分析的有效性。

#### `any`：意圖在執行時檢查後使用

`any` 型別被詮釋為開發者的一種聲明：「其值會在執行時進行類型檢查。」Strict-Ts 會基於此假設，在類型防禦區塊內，將其推斷為更精確的類型。

為確保此約定的履行，Strict-Ts 在嚴格模式下，會對未經類型防禦就直接使用的 `any` 型別變數報告編譯時錯誤，以減少潛在的執行時類型錯誤。

```ts
let value: any = someFn();

// 進行了型別防禦，履行了契約
if (value instanceof Foo) {
  // 在此區塊內，Strict-Ts 會將 `value` 安全地推斷為 `Foo` 型別。
}
```

#### `unknown`：不關心其具體型別

`unknown` 型別表示值的具體類型與當前邏輯無關。此類型適用於僅作傳遞或忽略回傳值的場景。

```ts
function someFn(callback: (val: number) => unknown) {
  // 在此函數中，我們不關心 callback 的回傳值是什麼，
  // 只關心它是一個可被呼叫的函數。
}
```

### 類型斷言


建議使用 `satisfies` 運算子來驗證類型，它能在不改變變數原始推斷類型的前提下，驗證其是否符合特定結構。相較之下，`as` 會直接覆蓋編譯器的類型推斷。

若必須使用 `as` 進行斷言，Strict-Ts 會根據斷言的方向採取不同策略：

#### 向上斷言（Up-casting，抽象化）

將一個較具體的類型斷言為其父類型或接口時，此操作被視為類型安全的。Strict-Ts 會在此基礎上繼續應用優化。

```ts
interface Foo { /* ... */ }
interface Bar extends Foo { /* ... */ }

let myBar: Bar = new Bar();
let myFoo: Foo = myBar as Foo; // 安全的向上斷言，保留優化
```

#### 向下斷言（Down-casting，具體化）

將一個較抽象的類型斷言為其子類型時，此操作存在類型不匹配的風險。Strict-Ts 會對其作用域及相關依賴採取較為保守的優化策略。

```ts
let myFoo: Foo = someFn();
let myBar: Bar = myFoo as Bar; // 不安全的向下斷言，優化將變保守
```

開發者可使用 `/*#__UNSAFE_TYPE_TRANSFORM__*/` 註解來標示已知安全的強制轉換。此註解會告知編譯器信任開發者的斷言，並繼續應用常規的優化策略。

```ts
// Object.keys(foo) 預設回傳 string[]，無法直接賦值給 keyof typeof foo。
// for (const key of Object.keys(foo) as keyof typeof foo) {
//   // 此處無法基於型別資訊進行積極優化策略。
// }

for (const key of /*#__UNSAFE_TYPE_TRANSFORM__*/ Object.keys(foo) as keyof typeof foo) {
  // 此處將完全信任開發者，並利用型別資訊進行積極優化策略。
}
```

### 型別斷言函數

回傳值為 `value is T` 的斷言函數，可用於輔助 Strict-Ts 的類型推斷。在斷言函數內部，Strict-Ts 將會嚴格檢查是否提供了足夠的證舉，否則 Strict-Ts 將會報告編譯時錯誤，以減少潛在的執行時類型錯誤。

如果函數的輸入型別是一個有限型別範圍，如：聯合類型、父類別、介面等。那麼斷言是相對安全的，可以利用較少的證據斷言類別。

```ts
function isFoo(value: Foo | Bar): value is Foo {
  return "prop_foo" in value; // 從 Foo | Bar 中判斷是否為 Foo
}

let a: Foo | Bar = someFn();
if (isFoo(a)) {
  // 此處 `a` 將被視為 `Foo` 物件。
} else {
  // 此處 `a` 將被視為 `Bar` 物件。
}
```

當輸入型別為 `any` 時，斷言的風險更高。開發者有責任提供足夠的執行時檢查來支撐這個斷言。

```ts
interface Foo {
  prop: string;
}

function isFoo(value: any): value is Foo {
  // 透過完整的屬性與型別檢查，為型別安全提供了「證據」
  return value && typeof value === 'object' &&
    "prop" in value && typeof value.prop === "string";
}
```

只有當你無法或因效能考量不願執行完整的執行時檢查，卻仍希望編譯器信任你的斷言時，才應該使用 `UNSAFE_ASSERT_` 前綴來顯式標記風險。這是在告訴編譯器：「相信我，我知道這在特定情境下是安全的，請在此基礎上進行優化。」

```ts
// 不安全的實作：檢查不完整，但開發者基於上下文確信其型別
function UNSAFE_ASSERT_isFoo(value: any): value is Foo {
  // 僅檢查了屬性存在，但未檢查型別，這可能在執行時出錯。
  // 使用 `UNSAFE_ASSERT_` 是在承認並標記這個風險。
  return "prop" in value;
}
```

### 純操作聲明

為了進行更徹底的 Tree Shaking（搖樹優化），開發者可標記某些函數調用為「純調用」。如果一個純調用的回傳值未被使用，該調用表達式可以在生產環境 (production) 構建中被移除。

#### 調用點標記 `/*#__PURE__*/`

此註解為業界通用標準，用於標記單次函數調用或物件實例化為純操作。開發者使用此註解時，即是向編譯器表明：「移除這一次調用不會產生非預期的副作用」。

```ts
// 一個有副作用的 ID 生成器
function generate_id(): number {
  // 此生成器會自動跳過已使用的編號，如：

  let id: number = -1;
  while (id < 0 || mem.has(id))
    id = Math.round(Math.random() * 0x1000);

  mem.add(id);
  return id;
}

// 僅當 `unused_id` 未被使用時，此調用才可能在生產模式下被移除。
const unused_id = /*#__PURE__*/ generate_id();

// 此調用因沒有 PURE 標記，將被保留。
generate_id();
```

除了函數調用外，也可以使用在模組匯入。

```ts
// 僅當匯入的項目不被使用時，此匯入才可能在生產模式下被移除。
/*#__PURE__*/ import * as ImportName from "module";
/*#__PURE__*/ import { someFn, constant } from "module";
```

#### 定義點標記 `/*#__HINT_PURE__*/`

`/*#__HINT_PURE__*/` 是 Strict-Ts 提供的一個註解，用於標記一個函數的所有調用在預設情況下都應被視為純調用。這簡化了程式碼，但此行為僅在 Strict-Ts 工具鏈中生效。對於需要跨工具鏈（如 Babel、SWC）保持一致行為的專案，建議使用通用的 `/*#__PURE__*/` 註解。

建議將此註解加入於 `function` 關鍵字與函數名稱中。

```ts
// 範例 1: 作為純調用的便利寫法
function /*#__HINT_PURE__*/ hint_generate_id(): number {
  // ...
}

// 下方的調用，在 Strict-Ts 中會被自動處理，
// 其效果完全等同於對一個普通函式加上 /*#__PURE__*/ 註解。
const my_id = hint_generate_id();

// 等價於：
// const my_id = /*#__PURE__*/ generate_id();
```

```ts
// 範例 2: 移除僅用於開發的函數
// 希望其僅在開發模式下作用，生產環境中消失。
function /*#__HINT_PURE__*/ debug_report(message: string): void {
  // 在開發模式下，這會打印日誌。
  console.log(message);
}

// 在生產模式下，下面這行代碼會被徹底移除，
// 因為函數返回 void，其返回值總是被視為「未使用」。
debug_report("Some debug message...");
```

### 不安全聲明

對於無法確定類型安全的程式碼區域，可使用 `/*#__UNSAFE__*/` 或 `/*#__UNSAFE_START__*/` 與 `/*#__UNSAFE_END__*/` 註解。Strict-Ts 將完全信任開發者所提供的型別資訊，並以此進行優化。

## 編譯器回報

當開發者遵循上述編碼約定，Strict-Ts 能夠執行以下優化：

### 增強的死碼消除

透過全域類型流分析，Strict-Ts 可以識別並移除基於類型邏輯不可達的程式碼分支。

```ts
// 原始碼
interface Foo { /* ... */ }
class A implements Foo { /* ... */ }
class B implements Foo { /* ... */ }
class C implements Foo { /* ... */ }

function branch(value: Foo) {
  if (value instanceof A) return doSthA(value);
  if (value instanceof B) return doSthB(value);
  if (value instanceof C) return doSthC(value);
  return doDefault(value);
}

// ...

// 實際呼叫
const a = new A();
const b = new B();
branch(a);
branch(b);
```

Strict-Ts 會追溯所有的輸入，找出所有潛在未執行的代碼，並移除。

```js
// Strict-Ts 死碼消除示例
class A { /* ... */ }
class B { /* ... */ }


function branch(value: Foo) {
  if (value instanceof A) return doSthA(value);
  return doSthB(value);
}

// ...

const a = new A();
const b = new B();
branch(a);
branch(b);
```

實際上，經過死碼消除後，`branch` 函數的複雜度降低。如果 `branch` 函數的調用點很少或函數體變得足夠簡單，Strict-Ts 會進一步採取函數內聯（Function Inlining）優化，將函數調用直接替換為函數體內容，並結合 Tree Shaking 移除不再被引用的 `branch` 函數本身。最終可能產生如下極致優化的程式碼：

```js
// Strict-Ts 輸出
class A { /* ... */ }
class B { /* ... */ }

// ...

doSthA(new A());
doSthB(new B());
```

### 更積極的 Tree Shaking

Strict-Ts 能夠分析一個類別的所有實例，以確定哪些成員（方法或屬性）從未被使用，並在最終輸出中移除它們。

```ts
// 原始碼
class DataProcessor {
    constructor(private data: string) {}

    encode(): string {
        return btoa(this.data);
    }

    // `decode` 方法從未被任何程式碼呼叫。
    decode(): string {
        return atob(this.data);
    }

    validate(): boolean {
        return this.data.length > 0;
    }
}

const processor = new DataProcessor("hello");
console.log(processor.encode());
if (processor.validate()) {
    // ...
}
```

```js
// Strict-Ts 輸出
// `decode` 方法已被安全地移除。
class DataProcessor {
    constructor(data) {
        this.data = data;
    }
    encode() {
        return btoa(this.data);
    }
    validate() {
        return this.data.length > 0;
    }
}

const processor = new DataProcessor("hello");
console.log(processor.encode());
if (processor.validate()) {
    // ...
}
```

### `enum` 優化

標準 TypeScript `enum` 會生成包含正向與反向查找的物件，即使反向查找未被使用。Strict-Ts 會對 `enum` 的使用情況進行全域追蹤分析。

為提升程式碼可讀性並明確開發者意圖，直接使用 const enum 仍是值得鼓勵的做法。

#### 無反向查找

如果分析發現沒有程式碼使用反向查找（例如 `Status[0]`），則會將該 `enum` 的成員直接內聯為常數值，類似於 `const enum` 的行為，並移除未使用的查找物件。

```ts
// 原始碼
enum Status { sleep, active }

switch (statusA) {
    case Status.sleep: break; // 使用正向查找
    case Status.active: break; // 使用正向查找
    default: // 死代碼
}
```

```ts
// Strict-Ts 輸出
switch (statusA) {
    case 0 /* Status.sleep */:
        break;
    case 1 /* Status.active */:
        break;
}
```

#### 有反向查找

若偵測到反向查找的使用，則會生成與 tsc 相容的查找物件。

```ts
// 原始碼
enum Status { sleep, active }

function getStatusName(status: Status) {
    return Status[status];
}
```

```js
// Strict-Ts 輸出，行為與 tsc 相同
"use strict";
var Status;
(function (Status) {
    Status[Status["sleep"] = 0] = "sleep";
    Status[Status["active"] = 1] = "active";
})(Status || (Status = {}));

function getStatusName(status) {
    return Status[status];
}
```

## 優化實踐

配合這些附加條款，可以產生更健壯的程式碼，並獲得更好的優化效果。

### 使用 `Result<T, E>` 處理可預期錯誤

建議使用 `Result<T, E>` 型別來處理可預期的錯誤，以取代傳統的 `throw` 陳述式。此模式將錯誤處理的邏輯納入類型系統，使編譯器能透過類型防禦（Type Guards）靜態分析程式碼的控制流程。`throw` 則應用於處理中斷正常流程的、不可預期的異常情況。

另外，`Promise`、`async function` 也同樣建議將結果、錯誤以 `Result<T, E>` 形式 `resolve`、`return`。而 `reject`、`throw` 則應用於處理中斷正常流程的、不可預期的異常情況。

## 配置說明

Strict-Ts 的行為可以通過專案的 `tsconfig.json` 文件中一個名為 `"strictTs"` 的屬性，或一個獨立的 `strict-ts.config.json` 文件進行配置。以下是對各個配置項的詳細說明。

### `"downcastingTransform"`

此選項定義了 Strict-Ts 如何處理向下斷言（Down-casting）。它控制著編譯器對這些潛在不安全操作的信任等級，並直接影響相關程式碼路徑的優化策略。

- `"error"`：建議選項。最嚴格的模式。任何向下斷言都會導致編譯時錯誤。此模式強制開發者顯式地處理所有類型不確定性，以獲得最高的運行時安全性。
- `"hint"`：預設選項，建議選項。當偵測到向下斷言時，編譯器會採取保守優化策略（即僅移除類型，不進行基於類型的激進優化），並在控制台中報告一條提示或警告，建議開發者審查此處的類型轉換。
- `"ignore"`：與 `"hint"` 類似，編譯器會採取保守優化策略，但不會在控制台中輸出任何提示。此選項適用於希望逐步遷移，但暫時不想處理大量警告的專案。
- `"trust"`：完全信任開發者的類型斷言。編譯器會在此斷言的基礎上，採取激進優化策略。這可能帶來顯著的性能和體積收益，但如果斷言在運行時被證明是錯誤的，則可能導致非預期的行為或錯誤。
- `"trust-and-hint"`：與 `"trust"` 相同，編譯器會採取激進優化策略，但同時會在控制台輸出提示，告知開發者此處進行了信任轉換。這有助於在享受優化的同時，保持對潛在風險點的追蹤。

#### 相關配置：`"transformFromExternalAny"`

此選項專門用於處理來自外部模組的 `any` 類型的向下斷言。由於第三方庫的類型定義可能使用 `any` 類型，因此為其提供一個獨立的配置。如下示例：

```ts
import { returnAny } from "legacy-module";
// 此函數回傳 any，並向下斷言為 Foo。
// 此操作原在 Strict-Ts 中是不可信的。
const value = returnAny() as Foo; 
```

若設置此選項，將覆蓋 `"downcastingTransform"` 的配置。

- `"inherit"`：預設選項，建議選項。完全遵照 `"downcastingTransform"` 的配置。
- `"error"`：建議選項。最嚴格的模式。任何未經 `UNSAFE_` 豁免的向下斷言都會導致編譯時錯誤。此模式強制開發者顯式地處理所有類型不確定性，以獲得最高的運行時安全性。
- `"hint"`：建議選項。當偵測到向下斷言時，編譯器會採取保守優化策略（即僅移除類型，不進行基於類型的激進優化），並在控制台中報告一條提示或警告，建議開發者審查此處的類型轉換。
- `"ignore"`：與 `"hint"` 類似，編譯器會採取保守優化策略，但不會在控制台中輸出任何提示。此選項適用於希望逐步遷移，但暫時不想處理大量警告的專案。
- `"trust"`：完全信任開發者的類型斷言。編譯器會在此斷言的基礎上，採取激進優化策略。這可能帶來顯著的性能和體積收益，但如果斷言在運行時被證明是錯誤的，則可能導致非預期的行為或錯誤。
- `"trust-and-hint"`：與 `"trust"` 相同，編譯器會採取激進優化策略，但同時會在控制台輸出提示，告知開發者此處進行了信任轉換。這有助於在享受優化的同時，保持對潛在風險點的追蹤。

另外，也可以針對外部模組進行個別設置：

```json
"transformFromExternalAny": {
  "react": "trust",                // 完全信任來自 'react' 模組的 any 斷言
  "some-old-library": "ignore",    // 忽略某個舊庫的 any，不報錯也不提示
  "*": "hint"                      // 對所有其他外部模組進行提示
}
```

#### `UNSAFE_` 豁免行為

當你使用 `/*#__UNSAFE_TYPE_TRANSFORM__*/` 註解時，Strict-Ts 將始終採用激進優化策略（等同於 `"trust"` 模式）。`"downcastingTransform"` 的配置僅影響在使用此豁免時，是否還會額外顯示一條提示。如果配置為 `"error"` 或 `"hint"`，Strict-Ts 仍會輸出一條提示，告知你這裡發生了一次不安全的豁免；在其他模式下，則會靜默處理。

### `"unreachableBranch"`

此選項控制 Strict-Ts 在通過類型流分析發現不可達程式碼分支時的行為。

- `"hint"`：預設選項，建議選項。在移除不可達分支的同時，向開發者顯示一條提示資訊。這有助於發現冗餘的防禦性程式碼或潛在的邏輯錯誤。
- `"error"`：將發現不可達分支視為一個編譯時錯誤。此模式適用於追求極致程式碼質量的專案，認為任何不可達程式碼都應被視為一個需要修復的 bug。
- `"ignore"`：靜默地移除不可達分支，不產生任何輸出。此選項適用於不關心此類提示的開發者。

### `"unusedIdentifier"`

此選項控制如何處理在全域分析後被確定為從未被使用的標識符（Identifier），包括局部變數、函數、類別以及未被匯出的頂層聲明等。

此功能類似於 tsc 的 `noUnusedLocals` 或 ESLint 的 `no-unused-vars`，但其分析基於 Strict-Ts 的全域依賴圖，因此可能比傳統的單檔案 Linter 更為精確。

- `"hint"`：預設選項，建議選項。對於未使用的標識符，在控制台輸出一條提示。相關的死碼（如變數聲明本身）將在優化階段被移除。
- `"error"`：將任何未使用的標識符視為編譯時錯誤，強制開發者保持程式碼的整潔。
- `"ignore"`：靜默地移除與未使用標識符相關的死碼，不產生任何輸出。


## 範例

### Result 結構

以下是 Strict-Ts 對 Result 的實作：

```ts
// Result 是一個聯合型別，代表成功 (OK) 或失敗 (ERR)
type Result<T, E> = OK<T> | ERR<E>;

// 使用 [T] 元組型別是為了方便透過解構賦值 `const [value] = result` 來取值。
// 使用 & { ok: boolean } 交叉型別來附加一個可辨識的屬性，用於型別防禦。
type OK<T> = [T] & { ok: true };
type ERR<T> = [T] & { ok: false };

function ok<T>(value: T): OK<T> {
  return Object.assign<[T], { ok: true }>([value], { ok: true });
}
function err<T>(value: T): ERR<T> {
  return Object.assign<[T], { ok: false }>([value], { ok: false });
}

// ...

let result: Result<string, string> = getMessage();

if (result.ok) {
  // 可以直接利用解構賦值取得
  const [message] = result;
  // ...
} else {
  const [error] = result;
  // ...
}
```

## 未來實踐

### `try-catch` 與 `Result<T, E>` 轉換

作為一個潛在的發展方向，Strict-Ts 或可提供一項功能，在編譯時將遵循特定規範（如 Web API）的 `try-catch` 結構，自動轉換為 `Result<T, E>` 型別。此功能旨在降低採納 Result 模式的成本，但其實現涉及複雜的程式碼分析，屬於長期探索目標。
