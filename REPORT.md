# CI Pipeline Report

## CI Pipeline 說明

本作業新增 `.github/workflows/ci_112033230.yaml`，讓 GitHub Actions 在每次 `push` 時自動執行 CI pipeline。Pipeline 使用 Node.js 24，符合專案 `package.json` 中 `>=22 <25` 的版本限制，並透過 `npm ci` 依照 `package-lock.json` 安裝固定版本的依賴。

Pipeline 包含三個主要品質檢查：

1. TypeScript typecheck：執行 `npm run typecheck`，使用 `tsc --noEmit` 檢查型別，不輸出編譯結果。
2. Prettier check：執行 `npm run format:check`，確認所有檔案符合 `.prettierrc` 的格式設定。
3. Test：執行 Vitest 測試，並同時產生 JUnit XML 測試報告。

GitHub Actions 預設會在任一步驟回傳非 0 exit code 時讓 job 失敗，因此 TypeScript、Prettier 或測試任一檢查失敗時，整個 pipeline 都會顯示 failed。

測試結果顯示方式採用兩種策略：

1. Vitest 產生 `reports/vitest-junit.xml`。
2. 使用 `dorny/test-reporter@v3` 讀取 JUnit XML，將測試結果顯示在 GitHub Actions 結果頁面。
3. 使用 `actions/upload-artifact@v7` 上傳 `reports/`，讓原始測試報告可以在 workflow run 中下載。

## Workflow 主要內容

```yaml
name: CI 112033230

on:
  push:

permissions:
  contents: read
  actions: read
  checks: write

jobs:
  ci:
    name: Typecheck, Format, and Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Setup Node.js
        uses: actions/setup-node@v6
        with:
          node-version: 24
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: TypeScript typecheck
        run: npm run typecheck

      - name: Prettier check
        run: npm run format:check

      - name: Run tests
        run: |
          mkdir -p reports
          npm test -- --reporter=default --reporter=junit --outputFile.junit=reports/vitest-junit.xml

      - name: Publish test results
        uses: dorny/test-reporter@v3
        if: ${{ !cancelled() && hashFiles('reports/vitest-junit.xml') != '' }}
        with:
          name: Vitest Results
          path: reports/vitest-junit.xml
          reporter: java-junit

      - name: Upload test report artifact
        uses: actions/upload-artifact@v7
        if: ${{ !cancelled() && hashFiles('reports/vitest-junit.xml') != '' }}
        with:
          name: vitest-junit-report
          path: reports/
          if-no-files-found: error
```

## CI 執行結果截圖

成功執行截圖：

請在 push 到 GitHub 後，進入 repository 的 Actions 頁面，打開 `CI 112033230` workflow 的成功執行結果，將截圖貼在此處。

截圖需包含：

1. Workflow run 狀態為成功。
2. `TypeScript typecheck`、`Prettier check`、`Run tests` 步驟皆成功。
3. GitHub Actions 結果頁面中顯示 Vitest 測試結果或 artifact。

## 失敗案例說明

失敗案例建議使用「測試失敗」，因為最容易從 GitHub Actions 結果頁看到測試報告。

製造錯誤方式：

暫時將 `test/app.test.ts` 中 `/health` 測試的預期值從：

```ts
expect(response.json()).toEqual({ status: 'ok' });
```

改成：

```ts
expect(response.json()).toEqual({ status: 'fail' });
```

接著 commit 並 push，GitHub Actions 會執行 `Run tests` 步驟。因為實際 API 回傳 `{ status: 'ok' }`，但測試預期 `{ status: 'fail' }`，Vitest 會回傳非 0 exit code，使 pipeline 顯示 failed。

失敗截圖：

請在 GitHub Actions 頁面打開 failed workflow run，將截圖貼在此處。截圖需包含 failed 狀態，以及測試失敗訊息或 `Vitest Results` 的失敗測試結果。

修正方式：

將測試預期值改回正確內容：

```ts
expect(response.json()).toEqual({ status: 'ok' });
```

再次 commit 並 push 後，pipeline 應恢復成功。
