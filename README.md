# klerk, Windows edition: installers and deployment

This repository holds the installers of the klerk clinic records app (Windows edition) as
[GitHub Releases](https://github.com/MMACABRE/MirKazemi-s.Public-/releases), and the deployment runbook. The source
code is private.

## Status

No installer has been released yet. The first release follows once the installer and the window app are built and
tested on a clean Windows 11 Pro PC.

## Installing (once a release exists)

1. From the newest release, download `klerk-setup-<tag>.exe` and `SHA256SUMS.txt`.
2. Check the file: in PowerShell, `Get-FileHash .\klerk-setup-<tag>.exe` must match its line in `SHA256SUMS.txt`.
3. Run the installer. It is not code-signed, so Windows shows "Windows protected your PC": press "More info", then
   "Run anyway".
4. On the clinic's main PC choose «سرور مطب»; on the other clinic PCs choose «اتصال به سرور».

## Deployment runbook

For the person (or the Claude Code deployer agent) setting up the clinic's host PC, the router and the internet
address:

1. [`docs/DEPLOY-AGENT-WINDOWS-READ-FIRST.md`](docs/DEPLOY-AGENT-WINDOWS-READ-FIRST.md): read this first.
2. [`docs/DEPLOY-AGENT-WINDOWS.md`](docs/DEPLOY-AGENT-WINDOWS.md): the full runbook (first install, the outside
   test, connecting to the internet, workstations, updates, repair, restore).

Where these documents mention `Plan.md` or `docs/specs/windows-edition.md`, those files are in the private
repository.
