# 貢獻指南

[English](CONTRIBUTING.md) | [官话 - 简体中文](CONTRIBUTING-cmn_CN.md) | [官話 - 正體中文](CONTRIBUTING-cmn_TW.md) | 廣東話

## 💻 設定開發環境

呢份專案係用咗 [Vite](https://vitejs.dev/) 建立，請確保你已經單咗 [Node.js](https://nodejs.org/) 同 [pnpm](https://pnpm.io/)，建議用 [Visual Studio Code](https://code.visualstudio.com/) 進行開發。

## 🔧 開發同建置專案

### 開發（Chrome 或 Edge）

#### Chrome 或 Edge 嘅第一種方法

<details>
 <summary>詳細內容</summary>

1. 執行 pnpm 指令

```bash
# 安裝依賴
pnpm install

# 建立一個用家帳戶資料夾，用於延伸功能存儲登入狀態
mkdir web-ext-profile

# 運行專案
pnpm dev

# 打完呢條指令之後，會自動開啓一個新嘅 Chrome 視窗並且打開 BiliBili 網站
pnpm start:chromium
```

2. 之後每次修改延伸功能，佢會重新載入，你可以 refresh 個網頁睇吓改變之後嘅效果

</details>

#### Chrome 或 Edge 嘅另外一種方法

<details>
 <summary>詳細內容</summary>

1. 執行 pnpm 指令

```bash
# 安裝依賴
pnpm install

# 運行專案
pnpm dev
```

2. 喺 Chrome 入邊打開 `chrome://extensions` 頁面抑或喺 Edge 度打開 `edge://extensions` 頁面

3. 打開`開發者模式`，撳`載入解壓縮`

<img width="655" alt="Snipaste_2022-03-27_18-17-04" src="https://user-images.githubusercontent.com/33394391/160276882-13da0484-92c1-47dd-add8-7655c5c2bf1c.png">
<br/>
<img width="655" alt="image" src="https://user-images.githubusercontent.com/33394391/232246901-e3544c16-bde2-480d-b770-ca5242793963.png">

4. 喺瀏覽器度載入產生嘅 `extension/` 資料夾

每一次執過 code 之後，你都要撳 [Extensions Reloader](https://chromewebstore.google.com/detail/extensions-reloader/fimgfedafeadlieiabdeeaodndnlbhid) 粒掣，然之後 refresh 個 page，確保係有效果。

</details>

#### 建置（Chrome 或 Edge）

建置延伸功能，要執行下底嘅指令

```bash
pnpm build
```

然之後打包 `extension` 下嘅檔案

### 開發（Firefox）

#### Firefox 嘅第一種方法

<details>
 <summary>詳細內容</summary>

1. 執行 pnpm 指令

```bash
# 安裝依賴
pnpm install

# 建立一個用家帳戶資料夾，用於延伸功能存儲登入狀態
mkdir web-ext-profile

# 運行專案
pnpm dev

# 打完呢條指令之後，會自動開啓一個新嘅 Firefox 視窗並且打開 BiliBili 網站
pnpm start:firefox
```

2. 之後每次修改延伸功能，佢會重新載入，你可以 refresh 個網頁睇吓改變之後嘅效果

</details>

#### Firefox 嘅另外一種方法

<details>
 <summary>詳細內容</summary>

1. 執行 pnpm 指令

```bash
# 安裝依賴
pnpm install

# 運行專案
pnpm dev-firefox
```

2. 喺瀏覽器度輸入 `about:addons`，撳 `Extensions` 然之後 `Debug Add-ons`

<img width="655" alt="image" src="https://github.com/hakadao/BewlyBewly/assets/33394391/7c49e4ca-2a87-4c56-bc00-3259d6eba128">

3. 喺瀏覽器度載入產生嘅 `extension-firefox/` 資料夾

</details>

#### 建置（Firefox）

建置延伸功能，要執行下底嘅指令

```bash
pnpm build-firefox
```

然之後打包 `extension-firefox` 下嘅檔案

## 🤝 貢獻

### 關於分支

- **main**：用呢個分支進行執 bug、新功能嘅開發、改進效能抑或執語系檔（i18n）。

### i18n

- 喺翻譯嗰陣，若然你遇到一種你唔熟嘅語言，你可以用第種識翻譯嘅語言來翻譯，兼且喺 PR 講明你唔識譯邊種語言。
- **請手動維護 i18n 國際化語系檔！！！** 請勿使用 `i18n Ally` 抑或其他擴充套件維護。 我知你可能唔係幾明，抑或可能唔鍾意咁樣，但係用 `i18n Ally` 進行維護之後，你唔之你翻譯咗嘅內容擺喺邊處，或剷咗程式碼註解。
