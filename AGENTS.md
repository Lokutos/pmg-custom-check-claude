# Projekt: pmg-custom-check-claude

Gemeinsame Anleitung für alle Agents.

## Kontext

- Custom-Check-Skript für den Proxmox Mail Gateway (PMG) zur Spam- und Phishing-Bewertung mit Claude (Anthropic API). Gibt PMG einen Score zurück (`SCORE: <Zahl>`), PMG addiert ihn als `CustomCheck(<Zahl>)`. Details in `README.md`, Skript `usr/local/bin/pmg-custom-check-claude`.
- Produktiv auf `mg247.rz1.swissnwx.ch` (PMG 9.1, venv `/opt/pmg-custom-check/venv`, Who-Objekt `pmg-custom-check`). Zugang und Regeln für den Server stehen im Projekt `rz1.swissnwx.ch` (`scripts\sx.ps1`, Schlüssel `joel_snwx-mg247`).
- Git: `https://github.com/Lokutos/pmg-custom-check-claude.git`, öffentlich, Joels GitHub-Konto `Lokutos`. Push nur nach Freigabe.

## Regeln

- Der Anthropic-Key steht nur in `/etc/pmg/anthropic.conf` auf dem PMG (root:pmg-smtp-filter, 0640). Nie ins Repo, nie ausgeben.
- Das Skript läuft produktiv im Mailfluss. Fehler dürfen keine Mails blockieren oder verlieren: jeder Fehlerfall (Rate-Limit, Overload, Timeout, ungültige Antwort) ergibt `OK` ohne Score. Gesamtdauer unter rund 15 s halten.
- Repo und Server gleich halten: Änderungen zuerst im Repo, ausrollen auf mg247 nur nach Freigabe. Auf dem Server nichts direkt testen oder ändern, ohne vorher nachzufragen.
