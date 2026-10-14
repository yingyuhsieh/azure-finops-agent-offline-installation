# Windows 離線安裝與啟動指南

本指南從「已取得完整的 `offline-package`」開始，適用於 Windows x64。依順序完成第一次安裝後，日後只需登入 Azure、設定環境變數並啟動程式。

安裝 Python 套件時不需要連網；AI 回答與 Azure 資料查詢仍須連線至 Microsoft Foundry、Azure 登入及相關 API。目標電腦不需要安裝 Node.js 或 .NET SDK。

## 1. 放置安裝包

在目標電腦建立 `D:\FinOps`，將 `offline-package` **裡面的全部內容**複製進去，包含 `app` 中所有檔案。

```text
D:\FinOps\
├── app\
│   ├── Dashboard.exe
│   ├── appsettings.json
│   ├── wwwroot\
│   └── 其他程式檔案……
├── wheels\
├── requirements-offline.txt
├── python-3.13.16-amd64.exe
└── AzureCLI-x64.msi
```

`wheels` 存放離線套件；`requirements-offline.txt` 記錄固定版本與 SHA-256 檢查碼，兩者須一起保留。套件適用於一般版 CPython 3.13、Windows x64。

`.venv` 是稍後在目標電腦建立的專用 Python 環境。不要沿用從其他電腦或其他路徑複製來的 `.venv`；請在最終安裝位置重新建立。

以下以 `D:\FinOps` 為例。若直接在目前的專案電腦操作，請將此路徑改成 `D:\mcp\azure-finops-agent\offline-package`；若沒有 D 槽，也可改用 `C:\FinOps`。

## 2. 安裝 Python

Python 用來分析上傳的檔案。

1. 在安裝包資料夾連按兩下 `python-3.13.16-amd64.exe`。
2. 勾選 **Add python.exe to PATH**，讓 PowerShell 能找到 Python。
3. 按 **Install Now**，等候完成。若使用自訂安裝，確認包含 **pip**。
4. 關閉安裝程式。

開啟 Windows 開始功能表，搜尋 **PowerShell** 並開啟。若原本已開啟，請關閉後重新開啟，讓設定生效。

以下每段指令貼上後按 Enter，等待完成再繼續。**只貼指令，不要貼執行結果、`PS ...>` 或 `>>` 提示。**

```powershell
python --version
python -m pip --version
python -c "import struct; print(struct.calcsize('P') * 8)"
```

預期依序顯示 Python 3.13 的版本、pip 版本，以及 `64`。如果已安裝相容的 Python 3.13 x64，可直接使用；若顯示其他主次版本或 `32`，先修正 Python 選用版本，再繼續。

## 3. 安裝 Azure CLI

Azure CLI 提供登入 Azure 所需的 `az` 指令。

1. 連按兩下 `AzureCLI-x64.msi`。
2. 依畫面接受授權條款並安裝；若需要管理員權限，依組織規定處理。
3. 完成後關閉並重新開啟 PowerShell。
4. 執行以下檢查：

```powershell
az version
```

看到包含 `azure-cli` 版本的資訊就代表安裝成功。此檢查不需要登入，也不會建立 Azure 資源。

## 4. 確認套件檔案存在

切換到安裝包所在資料夾：

```powershell
cd D:\FinOps
Test-Path .\wheels
Test-Path .\requirements-offline.txt
Test-Path .\app\Dashboard.exe
```

三項檢查都應顯示 `True`。若有 `False`，請先確認路徑或補齊檔案，暫停後續步驟。

## 5. 建立專用環境並離線安裝套件

先建立 `.venv`。它像這個專案專用的工具箱，可避免套件版本影響其他 Python 程式。

```powershell
python -m venv .venv
```

建立成功時可能沒有額外文字，回到 PowerShell 輸入提示即可。接著使用專用環境中的 Python 安裝：

```powershell
.\.venv\Scripts\python.exe -m pip install --no-index --find-links .\wheels --require-hashes -r .\requirements-offline.txt
```

| 指令部分 | 意義 |
| --- | --- |
| `.\.venv\Scripts\python.exe` | 使用本專案的 Python 環境 |
| `--no-index` | 不向網路套件站台查詢 |
| `--find-links .\wheels` | 從本機資料夾取得套件 |
| `-r .\requirements-offline.txt` | 按清單安裝固定版本 |
| `--require-hashes` | 檢查下載檔案是否符合清單中的 SHA-256 |

請勿將這行改為共用環境的 `python -m pip install`。此操作方式不需要執行 `Activate.ps1`，也不需要修改 PowerShell 執行原則。

## 6. 驗證 Python 套件

先檢查套件能否載入：

```powershell
.\.venv\Scripts\python.exe -c "import requests, pandas, numpy, openpyxl, tabulate, dateutil, pdfminer, pyarrow; print('Python packages OK')"
```

應顯示 `Python packages OK`。再檢查相依性：

```powershell
.\.venv\Scripts\python.exe -m pip check
```

應顯示 `No broken requirements found.`。完成這兩項才表示專用 Python 環境已準備好。

若之前曾安裝到共用 Python，建立 `.venv` 不會自動還原共用環境已變動的版本；本指南後續一律使用專用環境。

## 7. 準備 Azure 模型設定

請向管理 Azure 的同事取得以下資料：

| 資料 | 說明 |
| --- | --- |
| 租用戶 ID | 模型所在的 Azure 租用戶 |
| 訂用帳戶 ID | 要選用的 Azure 訂用帳戶 |
| Foundry 專案端點 | `https://帳戶名稱.services.ai.azure.com/api/projects/專案名稱` |
| 模型部署名稱 | 已建立的模型部署名稱 |

登入 Azure CLI 的帳號須在模型所在的 Foundry 帳戶上具備 **Foundry User** 角色。只有 Owner 管理角色不足以執行模型推論。安裝包不會自動建立 Foundry 專案或模型。

### 7.1 在瀏覽器開啟 Foundry 專案

這裡的「進入專案」是指在 Foundry 網站選取並開啟雲端專案，不是用 PowerShell 切換本機資料夾。以下操作需要連網。

1. 用 Edge 或 Chrome 開啟 [Microsoft Foundry](https://ai.azure.com/)。
2. 登入要使用的 Azure 帳號，確認選用正確的租用戶。
3. 找到專案選單或專案清單；若已有專案開啟，可點選左上角的專案名稱切換。
4. 點選要使用的專案，例如 `finops-project`。
5. 畫面顯示該專案名稱及專案內容後，才表示已「進入專案」。

若只看到建立專案的選項，請先確認管理者是否已建立專案及授予您存取權。若確實要建立新專案，選擇 **Create new project**，填入名稱，於 **Advanced options** 確認資源群組、區域及訂用帳戶，再選擇 **Create project**。建立新資源需要相應權限。

### 7.2 部署模型或確認既有部署

如果管理者已提供可用的模型部署，直接記下部署名稱，不必重複部署。若尚未部署，請在上一步開啟的 Foundry 專案中操作：

1. 點選 **Build**，再於左側選擇 **Models**。
2. 點選 **Deploy → Deploy a base model**，開啟模型選擇畫面。
3. 選擇符合此應用程式需求、支援 Responses API 與工具呼叫的模型；本儲存庫預設使用 `gpt-6.1-sol`，實際可部署項目取決於區域及配額。
4. 依部署畫面確認版本、部署類型及容量，設定部署名稱，例如 `finops-model`，再提交部署。模型使用會產生 Azure 費用。
5. 等待部署完成，開啟部署詳細資料，記下 **Deployment name（部署名稱）**。這個名稱可能與模型名稱不同。

如果找不到模型或無法部署，請向管理者確認區域、配額及權限。畫面名稱可能隨 Foundry 版本不同；完整操作可參閱 [Microsoft 官方建立專案與部署模型指南](https://learn.microsoft.com/en-us/azure/foundry/tutorials/quickstart-create-foundry-resources)。

### 7.3 複製專案端點並確認權限

1. 在同一專案開啟 **Overview（總覽）** 或專案歡迎頁面。
2. 找到 **Project endpoint（專案端點）**，複製完整網址，不要依顯示名稱自行拼湊。
3. 確認端點包含 `/api/projects/專案名稱`，格式如下：

```text
https://帳戶名稱.services.ai.azure.com/api/projects/專案名稱
```

依本儲存庫的本機設定要求，Azure CLI 登入帳號須在模型所在的 Foundry 帳戶資源上具備 **Foundry User** 角色。請有角色指派權限的管理者在該資源的 **存取控制（IAM）→ 新增角色指派** 中授予此角色。Owner 管理角色本身不足以執行模型推論。

端點用途可參閱 [Microsoft 官方 SDK 與端點說明](https://learn.microsoft.com/en-us/azure/foundry/how-to/develop/sdk-overview)。

### 7.4 填入本機設定

把剛取得的端點與部署名稱填入以下兩行，保留雙引號：

```powershell
$env:AzureOpenAI__Endpoint = "貼上完整的 Foundry 專案端點"
$env:AzureOpenAI__DeploymentName = "填入實際模型部署名稱"
```

這兩行須與 `Dashboard.exe` 在同一個 PowerShell 視窗執行。若程式已啟動，先按 Ctrl+C 停止，設定後再啟動。

第 8 節完成 Azure CLI 登入後，依第 9 節完整設定啟動程式。

### 7.5 驗證 AI 設定與連線

填入上述兩行，是指定程式要連接的 AI 模型；還需要確認以下條件：

- **模型已部署**：在 Foundry 的 **Build → Models** 確認部署可用，並核對 **Deployment name** 與 `AzureOpenAI__DeploymentName` 完全一致。部署名稱不一定等於模型名稱。
- **端點正確**：本指南使用完整的 Foundry 專案端點，網址須包含 `/api/projects/專案名稱`；不要直接以 Azure OpenAI 端點取代。
- **身分與權限可用**：依第 8 節登入 Azure，使用 `az account show` 確認帳號、租用戶及訂用帳戶。該帳號須具有第 7.3 節說明的 Foundry User 權限；登入成功本身不代表可以呼叫模型。
- **使用同一視窗啟動**：環境變數只對目前 PowerShell 工作階段及其啟動的程式生效。依第 9 節在同一視窗設定並啟動；若程式已執行，先按 Ctrl+C，重新設定後再啟動。

啟動前，可在該 PowerShell 視窗查看目前設定，確認已換成實際值，沒有保留提示文字或 `<...>`：

```powershell
$env:AzureOpenAI__Endpoint
$env:AzureOpenAI__DeploymentName
az account show
```

以上只能核對設定與登入狀態。完成第 9 節後，依第 10 節在本機聊天介面輸入「請用一句話介紹 Azure FinOps」，等待完整的 AI 回答，並查看執行程式的 PowerShell 是否有連線或驗證錯誤。這次測試會呼叫模型並產生用量費用。

**實際收到模型回答，才能確認當次本機設定、Azure 驗證與模型存取可用。** 能開啟網頁或看到 `Now listening on` 只表示網站已啟動，不能證明 AI 連線成功。若回答失敗，依錯誤訊息檢查登入與權限、端點與部署名稱，或網路與模型配額；不要僅因網頁可開啟就判定驗證通過。

## 8. 登入 Azure

從這一步開始，需要連線至 Azure。將範例中的 `<...>`（包括尖括號）替換為實際值：

```powershell
az login --tenant <您的租用戶ID>
az account set --subscription <您的訂用帳戶ID>
az account show
```

依提示完成登入與驗證。最後一行會顯示目前帳號與訂用帳戶，請確認環境正確。

## 9. 設定並啟動應用程式

在同一個 PowerShell 視窗操作。以下先回到安裝包根目錄，設定專用 Python，再切換到 `app` 啟動。請將端點與部署名稱換成實際值，保留雙引號。

```powershell
cd D:\FinOps
$env:FINOPS_PYTHON = (Resolve-Path .\.venv\Scripts\python.exe).Path

cd .\app
$env:ASPNETCORE_ENVIRONMENT = "Development"
$env:ASPNETCORE_URLS = "http://localhost:5000"
$env:AzureOpenAI__Endpoint = "https://<帳戶名稱>.services.ai.azure.com/api/projects/<專案名稱>"
$env:AzureOpenAI__DeploymentName = "<模型部署名稱>"

.\Dashboard.exe
```

`FINOPS_PYTHON` 告訴應用程式使用 `.venv` 中的 Python；其他設定指定本機網址及模型。這些環境變數只對目前 PowerShell 工作階段生效。

成功啟動時通常會看到 `Now listening on: http://localhost:5000`。**保持 PowerShell 視窗開啟**；程式正在執行，不會回到輸入提示。

## 10. 開啟網頁並確認功能

在 Edge 或 Chrome 的網址列輸入 **http://localhost:5000**。

1. 確認能看到聊天介面。
2. 輸入一個簡單問題，確認能收到 AI 回答。
3. 上傳一份合成測試檔案，確認檔案分析可用。請使用自己的測試資料或專案提供的示範資料。

能看到網頁只表示本機網站已啟動；AI 回答成功才表示模型連線與權限也正常。

目前設定提供匿名聊天。若要在網頁中登入並查詢自己的 Azure 成本與資源，還須設定 Entra 應用程式註冊及 `Microsoft:ClientId`、`Microsoft:ClientSecret`、`Microsoft:TenantId`。網頁登入與第 8 節的 Azure CLI 登入是分開的設定；詳見 [README_tw.md](README_tw.md)。

## 11. 停止與下次使用

停止程式：回到執行中的 PowerShell，按 **Ctrl+C**。

下次使用：

1. 開啟 PowerShell。
2. 必要時依第 8 節重新登入 Azure。
3. 重新執行第 9 節全部指令，包括 `FINOPS_PYTHON` 與模型設定。
4. 開啟 http://localhost:5000。

不必重裝 Python、Azure CLI 或套件。若更換電腦或移動安裝路徑，請在新位置重新建立 `.venv` 並安裝套件。

## 常見問題

| 狀況 | 處理方式 |
| --- | --- |
| 出現 `>>` 或 `ParserError` | 按 Ctrl+C，只重新貼上指令；不要貼入先前的輸出文字。 |
| 無法辨識 `python` 或 `az` | 重新開啟 PowerShell，確認工具已安裝及加入 PATH。 |
| 輸入 `python` 後開啟 Microsoft Store | 檢查 Windows 的應用程式執行別名，以及已安裝 Python 的 PATH。 |
| `Location .\wheels is ignored` | 目前位置不正確或缺少資料夾；回到第 4 節檢查。 |
| `No matching distribution found` | 確認 wheel 已帶齊，且使用一般版 Python 3.13 x64。 |
| 安裝顯示其他程式的套件衝突 | 確認指令以 `.\.venv\Scripts\python.exe` 開頭。 |
| SHA-256 不符 | 重新取得相符的清單與套件；不要移除檢查碼來略過驗證。 |
| `Resolve-Path` 找不到 Python | 確認在安裝包根目錄執行，且已完成第 5 節。 |
| 5000 連接埠已被使用 | 停止之前的程式，或將網址設定改成 `http://localhost:5001` 並使用新網址。 |
| 網頁開啟但 AI 回答失敗 | 檢查登入、Foundry User 權限、端點、部署名稱及網路。 |

請勿將實際認證資料或客戶識別碼提交至版本控制。提供錯誤訊息供排查前，先遮蔽機密資料。
