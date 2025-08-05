# **Strict-Ts Project Implementation Plan**

This document provides a neutral, actionable guide to the technical implementation of the `Strict-Ts` project.

## **Preparation Stage: Project Initialization and Infrastructure**

1.  **Environment Setup**
    1.1. Initialize a Node.js project (`npm init`).
    1.2. Install TypeScript as a development dependency (`npm i -D typescript`) and configure `tsconfig.json`.
    1.3. Select and configure a testing framework, such as `Vitest` or `Jest` (`npm i -D vitest`).
    1.4. Install and configure `ESLint` and `Prettier` to ensure code quality.

2.  **Core Dependency Installation**
    2.1. Install an AST parser: `npm i @typescript-eslint/typescript-estree`.
    2.2. Install an AST traversal tool: `npm i estraverse` or `@babel/traverse`.
    2.3. Install a code generator: `npm i escodegen` or `@babel/generator`.
    2.4. Install a Source Map handling tool: `npm i source-map`.

---

## **Milestone 1: Core Batch Processing Engine (MVP)**

**Goal:** Create a core transpiler capable of taking a single TypeScript file, performing basic optimizations, and outputting JavaScript.

3.  **Step 1: Establish the Core Transpilation Pipeline**
    3.1. Design a main `Compiler` or `Transpiler` class to act as the coordinator for all operations.
    3.2. Implement a `transpile(sourceCode: string, fileName: string): { code: string, map: object }` method.
    3.3. Integrate the following sub-processes within this method:
        3.3.1. Parse the `sourceCode` into an AST using `@typescript-eslint/typescript-estree`.
        3.3.2. **Placeholder:** Create an empty `transform(ast)` function that currently returns the original AST.
        3.3.3. Use a code generator to convert the AST back into a JavaScript code string.
        3.3.4. **Placeholder:** Implement basic Source Map generation logic to ensure input line numbers correspond to the output.

4.  **Step 2: Implement Type Information Retrieval**
    4.1. Modify the main pipeline to accept a `typescript.Program` object. This will require using the TypeScript API to create a `Program` based on a virtual file system or real files.
    4.2. Obtain the `TypeChecker` from the `Program` (`program.getTypeChecker()`).
    4.3. Develop a `TypeService` or utility class that takes the `TypeChecker` and an AST node and can return the node's TypeScript `Type` object.

5.  **Step 3: Implement Semantic Analysis and Annotation**
    5.1. Implement an AST visitor/traverser that will walk the entire AST.
    5.2. During traversal, add a `meta` or `symbol` property to each node to store analysis results from `Strict-Ts`.
    5.3. For `as` expressions (`TSAsExpression`):
        5.3.1. Get the type of the expression itself and the type of the assertion target.
        5.3.2. Implement an `isUpcasting(sourceType, targetType)` function to determine if it's a safe up-cast.
        5.3.3. Annotate the node's `meta` property as `SAFE_CAST` or `UNSAFE_DOWNCAST`.
    5.4. For `any` and `unknown` types:
        5.4.1. Identify variable declarations or expressions typed as `any` or `unknown`.
        5.4.2. Annotate the node's `meta` property with its type origin (e.g., `ANY_ORIGIN`, `UNKNOWN_ORIGIN`).

6.  **Step 4: Implement Statement-Level Dead Code Elimination (Branch Pruning)**
    6.1. Extend the AST traverser to handle `IfStatement` nodes.
    6.2. Analyze the `test` expression of the `IfStatement`.
    6.3. **Initial Implementation:** If `test` is a boolean literal (`true` or `false`), mark the corresponding branch (`consequent` or `alternate`) as `DEAD_CODE`.
    6.4. Modify the code generation phase to skip the generation of any node marked as `DEAD_CODE`.

7.  **Step 5: Integration and Testing**
    7.1. Create a series of `.ts` test files containing simple type assertions and `if(true/false)` branches.
    7.2. Write unit tests or snapshot tests to assert that the output of the `transpile` method is as expected (e.g., the `else` branch is removed).

---

## **Milestone 2: Global Analysis and Tree-shaking**

**Goal:** Extend the analysis scope from a single file to the entire project and implement module-level dead code elimination.

8.  **Step 6: Design and Implement the Dependency Graph**
    8.1. Design a `SymbolNode` data structure to represent "symbols" (functions, classes, variables). It should contain the name, source file, AST node reference, type information, etc.
    8.2. Design a `DependencyGraph` class for the entire project. It should store all `SymbolNode`s and the dependency relationships (edges) between them.
    8.3. Dependency edges should include the type of dependency (e.g., `CALL`, `REFERENCE`, `IMPORT`).

9.  **Step 7: Implement Global Symbol Analysis**
    9.1. Modify the main pipeline to accept a list of files (entry points).
    9.2. For each file in the project, perform AST parsing and symbol extraction.
        9.2.1. Traverse each file's AST to identify all `export`s and top-level declarations, creating `SymbolNode`s for them and adding them to the graph.
        9.2.2. Identify `import` statements and create file-to-file dependency edges in the graph.
        9.2.3. Inside function bodies or classes, identify references to other symbols and create finer-grained dependency edges in the graph.

10. **Step 8: Implement Reachability Analysis (Tree-shaking)**
    10.1. Implement a `markReachable(graph, entryPoints)` algorithm.
    10.2. Starting from the entry point files, recursively traverse the dependency graph.
    10.3. Add an `isReachable = true` flag to all visited `SymbolNode`s.
    10.4. During the dead code annotation phase, mark all top-level declarations (and their corresponding AST nodes) where `isReachable` is not `true` as `DEAD_CODE`.

11. **Step 9: Refine Source Map Generation**
    11.1. Ensure that original source location information from the Source Map is correctly passed and updated across multiple AST transformation stages (e.g., branch pruning, tree-shaking).
    11.2. Research and use the `SourceMapConsumer` and `SourceMapGenerator` from the `source-map` library to handle this chained transformation process.

---

## **Milestone 3: Incremental Computation and Dev Server Integration**

**Goal:** Implement the ability to perform incremental updates when files change.

12. **Step 10: Implement a Caching Mechanism**
    12.1. Design a caching strategy for analysis results. The cache key could be the file path, and the value could be the analysis results for that file (AST, symbol list, dependencies, etc.).
    12.2. Before each analysis, check the file's modification time or content hash. If it hasn't changed, read the results from the cache.

13. **Step 11: Implement the Incremental Update Algorithm**
    13.1. Watch for file system change events (add, modify, delete).
    13.2. When a file `A` changes:
        13.2.1. Invalidate the cache for `A`, re-analyze it, and update its nodes and outgoing edges in the dependency graph.
        13.2.2. Identify all files that depend on `A` (`B`, `C`, ...).
        13.2.3. Recursively perform an incremental analysis on `B`, `C`, etc., to check if their dependencies are still valid and update their analysis results, continuing until no more changes propagate through the dependency chain.
    13.3. Re-run the global reachability analysis.

14. **Step 12: Develop an Integration Plugin**
    14.1. Choose a target build tool (e.g., Vite).
    14.2. Read its plugin development documentation.
    14.3. Create a Vite plugin that calls the `Strict-Ts` core transpilation logic in its `transform` hook.
    14.4. Integrate `Strict-Ts`'s file watching and incremental update capabilities with Vite's dev server and HMR mechanism.

---

## **Milestone 4: Advanced Optimizations and Ecosystem Polish**

**Goal:** Implement all advanced features and configuration options from the design document.

15. **Step 13: Implement the Configuration System**
    15.1. Use `cosmiconfig` or a similar tool to read configuration files from the project (`tsconfig.json` or `strict-ts.config.json`).
    15.2. Pass the read configuration to the various modules of the compiler.
    15.3. During analysis and annotation stages, execute different logic or report diagnostics at different levels based on configuration options (e.g., `downcastingTransform`, `unreachableBranch`).

16. **Step 14: Implement Advanced Contracts and Optimizations**
    16.1. **`UNSAFE_` Annotations:** In the semantic analysis phase, identify and process various `UNSAFE_` annotations and prefixes to allow them to override default analysis behavior.
    16.2. **Assertion Function Validation:** Implement analysis of `is T` type assertion function bodies to validate the completeness of their checks.
    16.3. **`enum` Optimization:** Implement global tracking of `enum` usage to determine if reverse lookups are needed and inline members when appropriate.
    16.4. **Unused Property Removal:** Extend the dependency graph analysis to track class member access. If a property or method is never accessed, remove it during code generation.

17. **Step 15: Documentation and Publishing**
    17.1. Write a detailed README and user documentation explaining all features, contracts, and configuration options.
    17.2. Set up a CI/CD pipeline to automatically run tests and publish the package to NPM.