# 貢獻指南

[English](CONTRIBUTING.md) | [官话 - 简体中文](CONTRIBUTING-cmn_CN.md) | 官話 - 正體中文 | [廣東話](CONTRIBUTING-jyut.md)

## 💻 設定開發環境

此專案使用 [Vite](https://vitejs.dev/) 構建，請確保你離線安裝了 [Node.js](https://nodejs.org/) 和 [pnpm](https://pnpm.io/)，並推薦使用 [Visual Studio Code](https://code.visualstudio.com/) 進行開發。

## 🔧 開發與建置專案

### 開發（Chrome 或 Edge）

#### Chrome 或 Edge 的第一種方法

<details>
 <summary>詳細內容</summary>

1. 執行 pnpm 指令

```bash
# 安裝依賴
pnpm install

# 建立一個用戶的帳戶資料夾，用於擴充功能存儲登入狀態
mkdir web-ext-profile

# 運行專案
pnpm dev

# 打完這條指令之後，會自動開啓一個新的 Chrome 視窗並打開 BiliBili 網站
pnpm start:chromium
```

2. 之後每次修改擴充功能，他都會重新載入內容，你可以透過重新整理頁面來查看變更內容

</details>

#### Chrome 或 Edge 的另外一種方法

<details>
 <summary>詳細內容</summary>

1. 執行 pnpm 指令

```bash
# 安裝依賴
pnpm install

# 運行專案
pnpm dev
```

2. 在地址欄輸入 `chrome://extensions/`（Chrome），`edge://extensions/`（Edge）並按 Enter 鍵

3. 啟用 `開發者模式` 並點擊 `載入解壓縮`

<img width="655" alt="Snipaste_2022-03-27_18-17-04" src="https://user-images.githubusercontent.com/33394391/160276882-13da0484-92c1-47dd-add8-7655c5c2bf1c.png">
<br/>
<img width="655" alt="image" src="https://user-images.githubusercontent.com/33394391/232246901-e3544c16-bde2-480d-b770-ca5242793963.png">

4. 在瀏覽器中載入生成的 `extension/` 資料夾

每次修改後，您需要點選 [Extensions Reloader](https://chromewebstore.google.com/detail/extensions-reloader/fimgfedafeadlieiabdeeaodndnlbhid) 按鈕，然後重新整理頁面，以確保更改生效。

</details>

#### 建置（Chrome 或 Edge）

建置此擴充功能，需要執行以下指令

```bash
pnpm build
```

然後打包 `extension` 下的檔案

### 開發（Firefox）

#### Firefox 的第一種方法

<details>
 <summary>詳細內容</summary>

1. 執行 pnpm 命令

```bash
# 安裝依賴
pnpm install

# 建立一個用戶的帳戶資料夾，用於擴充功能存儲登入狀態
mkdir web-ext-profile

# 運行專案
pnpm dev

# 打完這條指令之後，會自動開啓一個新的 Firefox 視窗並打開 BiliBili 網站
pnpm start:firefox
```

2. 之後每次修改擴充功能，它都會重新加載，你可以透過重新整理頁面來查看變更內容

</details>

#### Firefox 的另一種方法

<details>
 <summary>詳細內容</summary>

1. 執行 pnpm 命令

```bash
# 安裝依賴
pnpm install

# 運行專案
pnpm dev-firefox
```

2. 在瀏覽器中輸入 `about:addons`，點擊 `Extensions` 然後 `Debug Add-ons`

<img width="655" alt="image" src="https://github.com/hakadao/BewlyBewly/assets/33394391/7c49e4ca-2a87-4c56-bc00-3259d6eba128">

3. 然後在瀏覽器中使用 `extension-firefox/` 資料夾載入此擴充功能。

</details>

#### 構建（Firefox）

要構建擴展，運行

```bash
pnpm build-firefox
```

然後打包 `extension-firefox` 下的檔案

## 🤝 貢獻

### 關於分支

- **main**：用於錯誤修正開發新功能、性能改進或修改國際化（i18n）文件的分支。

### I18n

- 在進行翻譯時，如果你遇到一種你不熟悉的語言，你可以使用另一種你已經翻譯過的語言來翻譯，並在 PR 中指出你無法翻譯的那個語言。
- **請手動維護 i18n 國際化語系檔！！！** 請勿使用 `i18n Ally` 或其他擴充套件來進行維護。 我知道你可能會感到困惑，或者可能不喜歡這樣做，但使用 `i18n Ally` 進行維護後，將不確定翻譯放在哪裏，或刪除程式碼註解。
