# PMG Custom Check mit Claude

Custom-Check-Skript für den Proxmox Mail Gateway (PMG). Es bewertet eingehende Mails mit Claude (Anthropic API) auf Spam, Phishing, Betrug und Schadlinks. PMG addiert den Score als `CustomCheck(<Score>)` zur SpamAssassin-Bewertung.

## Funktion

- Prüft nur Mails an Empfänger im PMG-Who-Objekt `pmg-custom-check` (Domains, Adressen oder Regex). Fehlt das Objekt oder ist es leer, bleibt die Prüfung aus.
- Schickt Absender, Reply-To, Return-Path, Betreff, Anhangnamen, bis zu 25 Links und höchstens 4000 Zeichen Text an Claude.
- Claude antwortet als JSON mit Score (0.0 bis 10.0) und einem kurzen deutschen Satz als Begründung. Beides landet im Syslog.
- Jeder Fehler (fehlender Key, Timeout, Rate-Limit, ungültige Antwort) ergibt `OK`. Eine Mail wird nie aufgehalten oder verworfen, sie läuft dann nur ohne KI-Score durch.
- Gesamtdauer bleibt unter rund 15 s (Timeout 12 s, keine Wiederholungen), weil manche Absender nach etwa 30 s abbrechen und die Mail erneut zustellen.
- Bei Überlast weicht die API serverseitig auf ein Ersatzmodell aus (Beta `server-side-fallback-2026-07-01`).

Score-Skala im Prompt:

- 0 bis 1: unbedenklich
- 2 bis 4: Werbung oder unerwünscht
- 5 bis 7: wahrscheinlich Spam
- 8 bis 10: Phishing, Betrug, Schadsoftware

## Voraussetzungen

- Proxmox Mail Gateway 8.x oder 9.x (getestet mit 9.1, Python 3.13)
- API-Key von Anthropic (https://console.anthropic.com/)
- Python-Paket `anthropic` ab 2.7 in einem eigenen venv

## Installation

Alle Schritte als root auf dem PMG.

### 1. Python-Umgebung

```bash
apt install python3-venv
python3 -m venv /opt/pmg-custom-check/venv
/opt/pmg-custom-check/venv/bin/pip install "anthropic>=2.7"
```

Das Skript startet über `#!/opt/pmg-custom-check/venv/bin/python3`. Liegt das venv woanders, die erste Zeile anpassen.

### 2. API-Key

```bash
cat > /etc/pmg/anthropic.conf << 'EOF'
ANTHROPIC_API_KEY=DEIN_KEY
EOF
chown root:pmg-smtp-filter /etc/pmg/anthropic.conf
chmod 0640 /etc/pmg/anthropic.conf
```

Weitere Einstellungen in derselben Datei (optional):

| Schlüssel | Standard | Bedeutung |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-opus-5-5` | Modell |
| `ANTHROPIC_EFFORT` | `low` | Reasoning-Stufe |
| `ANTHROPIC_TIMEOUT` | `12` | Timeout in Sekunden |

### 3. Skript

```bash
wget -O /usr/local/bin/pmg-custom-check-claude https://raw.githubusercontent.com/Lokutos/pmg-custom-check-claude/main/usr/local/bin/pmg-custom-check-claude
chmod 0755 /usr/local/bin/pmg-custom-check-claude
```

### 4. Rechte für pmgsh

Der Filter läuft als `pmg-smtp-filter` und braucht für das Who-Objekt `pmgsh`. Das Skript versucht es zuerst direkt, sonst über `sudo -n`.

```bash
echo 'pmg-smtp-filter ALL=(ALL) NOPASSWD: /usr/bin/pmgsh' > /etc/sudoers.d/pmg-custom-check
chmod 0440 /etc/sudoers.d/pmg-custom-check

mkdir -p /etc/systemd/system/pmg-smtp-filter.service.d
cat > /etc/systemd/system/pmg-smtp-filter.service.d/pmg-custom-check.conf << 'EOF'
[Service]
NoNewPrivileges=no
EOF
systemctl daemon-reload
```

Ohne `NoNewPrivileges=no` scheitert `sudo` im Dienst.

### 5. Custom Check aktivieren

In `/etc/pmg/pmg.conf`, Abschnitt `admin`:

```ini
section: admin
	custom_check 1
	custom_check_path /usr/local/bin/pmg-custom-check-claude
```

```bash
systemctl restart pmg-smtp-filter
```

### 6. Who-Objekt

In der PMG-Oberfläche unter Mail Filter -> Who Objects ein Objekt `pmg-custom-check` anlegen und die Domains oder Adressen eintragen, deren Mails geprüft werden sollen. Das Skript liest das Objekt alle 5 Minuten neu (Cache `/tmp/pmg_custom_check_who.cache`).

## Logs

```bash
journalctl -u pmg-smtp-filter -f | grep pmg-custom-check
```

Beispiel:

```text
pmg-custom-check: Claude claude-opus-5-5 2.4 s, Tokens 933/101, Score 8.0: Typische Spam-Werbung von einer Wegwerf-Domain.
pmg-custom-check: Score 8.0 für user@example.com von Shop <info@spam.example>
pmg-smtp-filter: ... hits=CustomCheck(8.0) ...
```

Fehler erscheinen mit Präfix `Claude:` (z. B. `Claude: Timeout nach 12.0 s, Mail ohne KI-Bewertung`).

## Test von Hand

```bash
sudo -u pmg-smtp-filter /usr/local/bin/pmg-custom-check-claude v1 /pfad/zur/mail.eml
```

Ausgabe `v1` und danach `OK` oder `SCORE: <Zahl>`. Der Empfänger der Test-Mail muss im Who-Objekt stehen.

## Lizenz

[MIT](LICENSE)
