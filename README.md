# Azure FinOps Agent — Offline Installation Package

This repository prepares a Windows x64 offline installation package for [Azure FinOps Agent](https://github.com/Azure-Samples/azure-finops-agent), originally developed in the Azure-Samples project. It contains a published application, Python dependencies, and prerequisite installers so the target computer can be set up without downloading packages.

The focus of this repository is packaging and installation. For application source code, architecture, and development instructions, see the [upstream repository](https://github.com/Azure-Samples/azure-finops-agent).

**Offline installation still requires online services at runtime.** AI responses require access to Microsoft Foundry, and Azure sign-in and resource queries require access to the relevant Azure services. The package does not include a local AI model or provision Azure resources.

For detailed instructions in Traditional Chinese, see [Windows 離線安裝與啟動指南](offline.install.md).

## Repository contents

```text
.
├── README.md
├── offline.install.md
├── offline-package.zip
└── offline-package/
    ├── app/
    │   ├── Dashboard.exe
    │   ├── Dashboard.dll
    │   ├── Dashboard.runtimeconfig.json
    │   ├── appsettings.json
    │   ├── wwwroot/                 # Built web interface
    │   ├── frontend/                # Frontend project files
    │   └── ...                      # Application dependencies and .NET runtime
    ├── wheels/
    │   ├── manifest.json            # Wheel versions, sources, sizes, and hashes
    │   └── *.whl                   # 18 Python dependency wheels
    ├── requirements-offline.txt     # Pinned versions and SHA-256 hashes
    ├── installers.sha256           # Python and Azure CLI installer hashes
    ├── python-3.13.16-amd64.exe
    ├── AzureCLI-x64.msi
    └── .venv/                      # Existing machine-specific environment; recreate locally
```

The published application includes .NET 10.0.13 and the built web assets. Target computers do not need Node.js, npm, or a separate .NET SDK/runtime installation. The Python wheels target standard CPython 3.13 on Windows x64.

## Requirements

- Windows x64 with PowerShell and a modern browser.
- Python 3.13 x64 and Azure CLI, installed using the bundled installers if needed.
- An existing Microsoft Foundry project endpoint and model deployment.
- An Azure account authorized to use that model deployment.
- Network access to Azure authentication, the Foundry endpoint, and any services used by the requested features.

The installation guide specifies the **Foundry User** role for model access. Have your Azure administrator confirm the required data-plane permissions; an Azure management role alone does not establish model inference access.

## Install on the target computer

### 1. Extract and install prerequisites

Extract `offline-package.zip`, then copy the **contents of its `offline-package` folder** into your final installation directory, such as `D:\FinOps`. You can also copy the unpacked folder contents directly.

Keep the entire `app` directory, all wheels, both installers, and their manifests together. Exclude the bundled `.venv` when copying: it refers to paths on the preparation computer. If already extracted, rename that `.venv` before creating a fresh one.

Run `python-3.13.16-amd64.exe` and enable **Add python.exe to PATH** and **pip**. Run `AzureCLI-x64.msi`, then reopen PowerShell.

```powershell
python --version
python -c "import struct; print(struct.calcsize('P') * 8)"
az version
```

Confirm Python 3.13 and `64` before continuing. Use your actual installation directory in place of `D:\FinOps` throughout this README.

### 2. Create a local Python environment

```powershell
Set-Location D:\FinOps
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --no-index --find-links .\wheels --require-hashes -r .\requirements-offline.txt
.\.venv\Scripts\python.exe -m pip check
.\.venv\Scripts\python.exe -c "import requests, pandas, numpy, openpyxl, tabulate, dateutil, pdfminer, pyarrow; print('Python packages OK')"
```

`--no-index` prevents package-index access, and `--require-hashes` checks each wheel against the requirements file. Expect `No broken requirements found.` and `Python packages OK`. Environment activation and changes to PowerShell execution policy are unnecessary.

Recreate this environment whenever the installation moves to another computer or directory.

### 3. Sign in to Azure

From this step onward, network connectivity is required. Replace the placeholder values with your own tenant and subscription IDs.

```powershell
az login --tenant "<tenant-id>"
az account set --subscription "<subscription-id>"
az account show
```

### 4. Configure and start the application

Run these commands in the same PowerShell window, replacing the Foundry endpoint and deployment name with your actual values:

```powershell
Set-Location D:\FinOps
$env:FINOPS_PYTHON = (Resolve-Path .\.venv\Scripts\python.exe).Path
$env:ASPNETCORE_ENVIRONMENT = "Development"
$env:ASPNETCORE_URLS = "http://localhost:5000"
$env:AzureOpenAI__Endpoint = "https://<account-name>.services.ai.azure.com/api/projects/<project-name>"
$env:AzureOpenAI__DeploymentName = "<model-deployment-name>"
Set-Location .\app
.\Dashboard.exe
```

This follows the installation guide's local Development configuration. Keep the PowerShell window open and visit **http://localhost:5000**. Verify a chat response, then upload a synthetic test file to check Python-backed file analysis. Press **Ctrl+C** to stop the application.

These environment variables last only for the current PowerShell session. On subsequent launches, sign in again if necessary and repeat the configuration/start commands; dependencies do not need to be reinstalled.

### Optional: web sign-in for Azure data

Azure CLI authentication for the model and browser authentication for your Azure data are separate. The installation guide describes anonymous chat as the initial configuration. Browser sign-in requires an Entra application registration and the following configuration values:

| Environment variable | Purpose |
| --- | --- |
| `Microsoft__ClientId` | Entra application client ID |
| `Microsoft__ClientSecret` | Entra application client secret |
| `Microsoft__TenantId` | Tenant used for browser sign-in |

Configure the registration, redirect URIs, consent, and permissions using the documentation for the [upstream project](https://github.com/Azure-Samples/azure-finops-agent). Keep credentials out of the repository and distribution archive.

## Preparing a distributable package

The repository currently contains a prepared folder and ZIP. It does not contain a repeatable application build or packaging script. Before publishing a release:

1. Record the exact upstream commit/tag used to build `Dashboard.exe`, along with the build command and target runtime.
2. Retain the complete published application and built `wwwroot` assets.
3. Collect all Python dependencies for CPython 3.13 / Windows x64, pin every dependency, and update the wheel hashes and `wheels/manifest.json` together.
4. Include the prerequisite installers and update `installers.sha256` whenever they change. From the package directory, use `Get-FileHash -Algorithm SHA256` to compare installers with the manifest.
5. Assemble a clean distribution folder without `.venv`, credentials, uploaded customer files, or local session data. Include this README, the installation guide, upstream license, and applicable third-party notices.
6. Create a new archive from that folder. Test extraction, prerequisite installation, and Python installation on a clean Windows x64 computer with package-download access disabled. Then enable required Azure connectivity and verify chat and file analysis.
7. Publish the archive with its SHA-256 checksum and a release manifest identifying the upstream revision and bundled component versions. Use a release asset or artifact store that supports its size; the current ZIP is approximately 390 MiB.

The existing ZIP includes a machine-specific `.venv`; regenerate a clean archive before distribution. The upstream build revision is not recorded in the current package.

## Troubleshooting

| Problem | Check |
| --- | --- |
| `python` or `az` is not recognized | Reopen PowerShell after installation and check PATH. |
| No matching Python distribution | Use standard Python 3.13 x64 and ensure all wheels are present. |
| Wheel hash mismatch | Obtain the matching requirements file and wheels; keep hash verification enabled. |
| `FINOPS_PYTHON` cannot be resolved | Create `.venv` in the final installation directory and resolve it from that directory. |
| Port 5000 is occupied | Stop the earlier process or set `ASPNETCORE_URLS` to `http://localhost:5001` and open that URL. |
| Web interface works but AI responses fail | Check Azure CLI sign-in, model permissions, the project endpoint, deployment name, and connectivity. |
| Browser sign-in is unavailable | Configure the Entra registration separately from Azure CLI sign-in. |

## Verification status

Repository review confirmed that all 18 wheel files match the hashes in `requirements-offline.txt` and that both installers match `installers.sha256`. These checks establish consistency with the local manifests. Installation on a clean target computer and live Azure/model operation were not tested during this documentation review.

## Attribution and licensing

The original application is [Azure-Samples/azure-finops-agent](https://github.com/Azure-Samples/azure-finops-agent), which is published under the [MIT License](https://github.com/Azure-Samples/azure-finops-agent/blob/main/LICENSE). Preserve its copyright and license notice when redistributing the application. This repository packages that application for offline installation.

A standalone upstream license file was not found in the current package. Add the license from the exact upstream revision used for the build, together with applicable third-party license notices, before publishing the distribution. Python, Azure CLI, .NET, and bundled dependencies retain their respective licenses.
