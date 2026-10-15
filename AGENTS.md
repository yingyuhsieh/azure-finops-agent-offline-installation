# Repository Guidelines

## Project Structure & Module Organization

This repository packages Azure FinOps Agent for Windows x64 offline installation. Application development belongs in the upstream `Azure-Samples/azure-finops-agent` repository.

- `README.md`: English installation instructions and release preparation checklist.
- `offline.install.md`: detailed Traditional Chinese installation guide.
- `offline-package/app/`: published Dashboard executable, bundled .NET runtime, and web assets in `wwwroot/`.
- `offline-package/wheels-py312/` and `offline-package/wheels-py313/`: version-specific Python wheels and `manifest.json` with dependency metadata.
- `requirements-offline.txt` and `installers.sha256` inside `offline-package/`: pinned dependency versions and installer checksums.
- `offline-package.zip`: distribution archive. Git LFS tracks this archive and the Azure CLI MSI.

## Build, Test, and Development Commands

Run `git lfs install` and `git lfs pull` after cloning to obtain complete binary artifacts. There is no repository-level application build or packaging script; use upstream instructions to rebuild the application.

From `offline-package/`, using CPython 3.13 x64:

```powershell
py -3.13 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --no-index --find-links .\wheels-py313 -r .\requirements-offline.txt
.\.venv\Scripts\python.exe -m pip check
```

These commands create a local environment, install pinned offline dependencies, and check dependency compatibility. Follow `README.md` to configure environment variables and run `.\Dashboard.exe` from `app/`.

## Coding Style & Naming Conventions

Use descriptive Markdown headings, fenced `powershell` examples, and explicit working directories. Keep English and Traditional Chinese instructions consistent. Preserve published filenames and wheel naming conventions. No repository-wide formatter or linter is configured.

## Testing Guidelines

No repository-level automated test suite or coverage threshold is defined. Validate changes on clean Windows x64 with package downloads disabled. Check dependency imports, compare installer hashes using `Get-FileHash -Algorithm SHA256`, then enable Azure connectivity and verify chat and synthetic-file analysis. Record checks performed and any untested behavior.

## Commit & Pull Request Guidelines

History includes timestamp-style subjects and descriptive imperative subjects; no single convention is established. Prefer concise imperative messages such as `Update offline dependency manifest`. PRs should explain the change, identify affected component versions, link relevant issues, and report validation. Include screenshots for visible interface changes.

## Security & Release Hygiene

Never commit credentials, customer files, or machine-specific `.venv` directories. Update wheels, pinned requirements, and manifests together. Regenerate a clean archive after package changes; record the upstream revision, preserve license notices, and publish its checksum.
