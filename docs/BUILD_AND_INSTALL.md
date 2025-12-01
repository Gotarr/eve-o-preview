# Build-Anleitung für VRAM-Optimierungen

## Schnellstart

```powershell
# Im Repository-Root:
cd e:\GutHub\eve-o-preview

# Build für Windows (Ihre Konfiguration):
dotnet build src\Eve-O-Preview\Eve-O-Preview.csproj --configuration Release -p:EVEOTARGET="Windows"
```

## Output-Pfad

Die kompilierte `.exe` findet sich in:
```
e:\GutHub\eve-o-preview\bin\net8.0-windows8.0\
```

## Installation

1. **Stoppen Sie** das aktuell laufende EVE-O-Preview
2. **Backup** Ihrer aktuellen Config:
   ```powershell
   Copy-Item "e:\Games\EVE-O.Preview8\EVE-O-Preview.json" "e:\Games\EVE-O.Preview8\EVE-O-Preview.json.backup"
   ```
3. **Ersetzen** Sie die `EVE-O-Preview.exe`:
   ```powershell
   Copy-Item "e:\GutHub\eve-o-preview\bin\net8.0-windows8.0\EVE-O-Preview.exe" "e:\Games\EVE-O.Preview8\EVE-O-Preview.exe"
   ```

## Konfiguration anpassen

Öffnen Sie `e:\Games\EVE-O.Preview8\EVE-O-Preview.json` und fügen Sie hinzu:

```json
{
  "ForceRefreshCycleThreshold": 3
}
```

**Hinweis**: Die anderen Optimierungen (DWM-Throttling, Font-Caching) sind Code-basiert und automatisch aktiv.

## Testen

1. Starten Sie EVE-O-Preview
2. Starten Sie Ihre 10 Mining-Clients
3. Öffnen Sie **Task-Manager** → **Details** Tab → EVE-O-Preview.exe
4. Rechtsklick → **Spalten auswählen** → "GPU-Motor" aktivieren
5. Beobachten Sie die GPU-Memory-Nutzung

**Erwartung**: ~45-55 MB statt ~60-80 MB

## Fehlerbehebung

### "ForceRefreshCycleThreshold" wird ignoriert

→ Die Config wird nur beim Start geladen. EVE-O-Preview neustarten.

### Thumbnails werden nicht aktualisiert

→ `ForceRefreshCycleThreshold` zu hoch. Reduzieren Sie auf 2.

### Keine Verbesserung sichtbar

→ Prüfen Sie, ob `"WineCompatibilityMode": false` gesetzt ist (DWM-Modus).

## Rollback

Falls Probleme auftreten:

```powershell
# Alte Version wiederherstellen
Copy-Item "e:\Games\EVE-O.Preview8\EVE-O-Preview.exe.old" "e:\Games\EVE-O.Preview8\EVE-O-Preview.exe"

# Alte Config wiederherstellen
Copy-Item "e:\Games\EVE-O.Preview8\EVE-O-Preview.json.backup" "e:\Games\EVE-O.Preview8\EVE-O-Preview.json"
```
