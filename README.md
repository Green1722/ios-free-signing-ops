# iOS Free-Signing Ops (Windows)

**Keep a sideloaded iOS app alive on Windows without paying for a developer account — automated status checks, one-command re-signing, and the error codes that actually matter.**

一个面向 Windows 的 iOS 免费签名运维工具集：把「7 天到期 → 重新签名 → 覆盖安装」这件事变成一条命令，并在到期前自动提醒。

> Built from a real migration (macOS/Xcode → Windows 11) of a location-spoofing app with a free Apple ID. The findings below were verified on device, not guessed.

---

## Why this exists

A free Apple ID signs apps for **7 days**. Sideloading tools handle the *first* install, but nobody handles the boring part: knowing when it expires, and re-signing before the app silently stops launching.

This repo is that part: 4 small PowerShell scripts + a config file.

| File | Purpose |
|---|---|
| `check-status.ps1` | Read-only health check: days left, device over USB, app present, anisette endpoint reachable. Exit codes `0/1/2`. |
| `renew.ps1` | Automated re-sign + reinstall through [iLoader](https://github.com/nab138/iloader): selects the device, clicks *Import IPA*, fills the path, waits for `Operation completed`, updates the config. |
| `backup.ps1` | Backs up config, iLoader session, logs, IPA and pairing file. |
| `install-schedule.ps1` | Registers a daily scheduled task that pops a notification when ≤ 2 days remain. |

---

## The three findings that cost the most time

If you are fighting free-signing on Windows, these save you hours:

1. **Use the phone-number form of your Apple ID (`+86…`).**
   With an email alias, both Sideloadly and iLoader fail *at the password step* with `-22406 Enter the correct password`, even though the same credentials work on apple.com. Switching the login name to `+86` + number passed immediately. Reproduced with two different accounts.

2. **Non-Mac anisette is rejected by Apple since 2026-10-21.**
   Sideloadly reports it verbatim:
   `-80046 … Since 2026-10-21 Apple rejects this anisette type (non-Mac) when issuing app tokens, which affects all sideloading tools.`
   Local anisette and third-party servers fail at the **app-token** stage. Sideloadly's `Remote Anisette` (Mac type) is a Patreon feature; the free route that worked here is **iLoader + `StikStore`**.
   Symptom map: `-22406` = wrong login-name form · `-22421` = that anisette service unusable (try `StikStore` → `iloader` → `Jkcoxson`) · `-80046` = non-Mac anisette.

3. **A phone hotspot can silently break the flow.**
   Measured against `gsa.apple.com:443`: 6 TCP attempts → 4 OK, 2 hung for 6 s. That surfaces as `WinError 10060` at `GsService2/validate`, right after the 2FA code. Signing/installing uses USB, so the PC should be on a stable wired/Wi-Fi network.

---

## Requirements

- Windows 10/11, PowerShell 5.1+
- iPhone with **Developer Mode** on (Settings → Privacy & Security)
- [iLoader](https://github.com/nab138/iloader) 2.4.x, logged in once by hand
- Python + [`pymobiledevice3`](https://github.com/doronz88/pymobiledevice3) in a venv (used only to ask the device what is installed)
- An IPA you are allowed to install

## Setup

```powershell
git clone <this repo> ios-free-signing-ops
cd ios-free-signing-ops
Copy-Item config.example.json config.json
notepad config.json          # workRoot, ipaPath, bundleId, appleId, teamId, anisetteServer, signedOn
```

Set `anisetteServer` to the service that works for you (here: `ani3.stikstore.app`), and open iLoader once to pick it (`Anisette Server: StikStore`), then log in with the `+86` phone number and finish 2FA.

First check:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\check-status.ps1 -Deep
```

## Daily use

Register the reminder (once):

```powershell
.\install-schedule.ps1            # daily 20:00, notifies when <= 2 days remain
```

## Renewing (every 7 days)

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\renew.ps1 -DryRun   # inspect only
powershell -NoProfile -ExecutionPolicy Bypass -File .\renew.ps1           # re-sign + install
```

`renew.ps1` never touches your password. If the iLoader session expired it exits with code `7` and asks you to log in by hand, then you rerun it. On success it stamps a new `signedOn` into `config.json`, so the 7-day counter restarts and the app keeps its data (same bundle ID, overwrite install).

Exit codes: `3` no device · `4` no IPA · `7` not logged in · `10` file dialog missing · `12` iLoader error · `13` timeout.

## Troubleshooting

| Symptom | Meaning | Fix |
|---|---|---|
| `-22406 Enter the correct password` | login name is an email alias | use `+86` + phone number |
| `-22421` at *proof login* | anisette service unusable | switch service, then `Reset Anisette State` and redo 2FA |
| `-80046 … non-Mac anisette` | non-Mac anisette | don't use Sideloadly local anisette; iLoader + `StikStore` |
| `WinError 10060` / timeouts | flaky network | use stable broadband |
| `file dialog did not appear` (exit 10) | device dropped / card not selected | replug USB, unlock phone, ensure the device card shows `Selected` |
| icon crashes / "Untrusted Developer" | trust revoked or signature expired | Settings → General → VPN & Device Management → trust; or renew |
| app missing from device | expired and removed | rerun `renew.ps1` |

## Security notes

- `config.json`, `backups/` and `*.ipa` are **git-ignored on purpose**: they contain your phone number, iLoader session and pairing material. Never commit or share them.
- `renew.ps1` drives the iLoader GUI. It never reads, stores or transmits credentials.
- Nothing here bypasses Apple's limits; it only automates the legal 7-day renewal cycle.

## License

MIT — see [LICENSE](LICENSE).
