# VRAM-Optimierung für EVE-O Preview

## Aktueller Stand der VRAM-Auslastung

### Zwei Rendering-Modi mit unterschiedlichem VRAM-Verbrauch

#### 1. **LiveThumbnailView** (DWM-basiert) - NIEDRIG (~50 MB für 3 Thumbnails)
- Verwendet Windows Desktop Window Manager (DWM) API
- **Vorteil**: Sehr geringer VRAM-Verbrauch, da DWM die Thumbnail-Verwaltung übernimmt
- **Nachteil**: Funktioniert nicht in RDP-Sessions oder ohne DWM Composition
- Datei: `src/Eve-O-Preview/View/Implementation/LiveThumbnailView.cs`

#### 2. **StaticThumbnailView** (Screenshot-basiert) - HOCH (~180 MB für 3 Thumbnails)
- Erstellt Screenshots der EVE-Fenster und speichert sie als Bitmaps
- **Vorteil**: Funktioniert überall (auch RDP, Wine)
- **Nachteil**: ~3,6x höherer Speicherverbrauch
- Datei: `src/Eve-O-Preview/View/Implementation/StaticThumbnailView.cs`

### Rendering-Auswahl
```csharp
// ThumbnailViewFactory.cs
public IThumbnailView Create(IntPtr id, string title, Size size)
{
    IThumbnailView view = this._enableWineCompatibilityMode
        ? (IThumbnailView)this._controller.Create<StaticThumbnailView>()  // Screenshot-Modus
        : (IThumbnailView)this._controller.Create<LiveThumbnailView>();   // DWM-Modus
}
```

Gesteuert durch `EnableWineCompatibilityMode` (alias `CompatibilityMode` in Config):
- **Windows Standard**: `false` → LiveThumbnailView (niedriger VRAM)
- **Linux/Wine**: `true` → StaticThumbnailView (hoher VRAM)

---

## Identifizierte VRAM-Problembereiche

### 1. **Bitmap-Erstellung ohne explizite Freigabe** ⚠️ KRITISCH
**Datei**: `src/Eve-O-Preview/Services/Implementation/WindowManager.cs` (Zeilen 295-325)

```csharp
public Image GetStaticThumbnail(IntPtr source)
{
    var sourceContext = User32NativeMethods.GetDC(source);
    
    // ... Größenprüfung ...
    
    var destContext = Gdi32NativeMethods.CreateCompatibleDC(sourceContext);
    var bitmap = Gdi32NativeMethods.CreateCompatibleBitmap(sourceContext, width, height);  // ← VRAM-Allokation
    
    var oldBitmap = Gdi32NativeMethods.SelectObject(destContext, bitmap);
    Gdi32NativeMethods.BitBlt(destContext, 0, 0, width, height, sourceContext, 0, 0, Gdi32NativeMethods.SRCCOPY);
    Gdi32NativeMethods.SelectObject(destContext, oldBitmap);
    Gdi32NativeMethods.DeleteDC(destContext);
    User32NativeMethods.ReleaseDC(source, sourceContext);
    
    Image image = Image.FromHbitmap(bitmap);  // ← Erstellt GDI+ Kopie
    Gdi32NativeMethods.DeleteObject(bitmap);   // ← Native Bitmap wird gelöscht, aber...
    
    return image;  // ← GDI+ Image bleibt im VRAM!
}
```

**Problem**: 
- `Image.FromHbitmap()` erstellt eine **GDI+ Kopie** des Bitmaps im Managed Memory
- Diese Kopie bleibt bestehen, bis `Image.Dispose()` aufgerufen wird
- Bei jedem Refresh-Zyklus wird ein neues Image erstellt

### 2. **Alte Bilder werden nicht sofort freigegeben**
**Datei**: `src/Eve-O-Preview/View/Implementation/StaticThumbnailView.cs` (Zeilen 30-43)

```csharp
protected override void RefreshThumbnail(bool forceRefresh)
{
    if (!forceRefresh || this.IsPreventPreviews())
    {
        return;  // ← Kein Refresh = Altes Bild bleibt im Speicher
    }
    
    var thumbnail = this.WindowManager.GetStaticThumbnail(this.Id);
    if (thumbnail != null)
    {
        var oldImage = this._thumbnail.Image;
        this._thumbnail.Image = thumbnail;  // ← Neues Bild wird zugewiesen
        oldImage?.Dispose();  // ← Altes Bild wird entsorgt (GUT!)
    }
}
```

**Gut**: Alte Bilder werden disposed
**Problem**: Nur bei `forceRefresh == true`, nicht bei jedem Tick

### 3. **Refresh-Zyklen-Management**
**Datei**: `src/Eve-O-Preview/Services/Implementation/ThumbnailManager.cs` (Zeilen 291-295, 399-475)

```csharp
private const int FORCED_REFRESH_CYCLE_THRESHOLD = 2;

private void ThumbnailUpdateTimerTick(object sender, EventArgs e)
{
    this.UpdateThumbnailsList();
    this.RefreshThumbnails();  // ← Wird JEDES Mal aufgerufen
}

private void RefreshThumbnails()
{
    // ...
    bool forceRefresh;
    if (this._refreshCycleCount >= ThumbnailManager.FORCED_REFRESH_CYCLE_THRESHOLD)
    {
        forceRefresh = true;
        this._refreshCycleCount = 0;
    }
    else
    {
        forceRefresh = false;
        this._refreshCycleCount++;
    }
    
    // ...
    view.Refresh(forceRefresh);  // ← forceRefresh nur alle 2 Zyklen!
}
```

**Problem**:
- **Timer-Intervall**: 500ms (Windows) / 10ms (Linux) konfigurierbar
- **Force Refresh**: Nur alle 2 Zyklen (= 1 Sekunde bei Windows)
- Bedeutet: Screenshots werden nur 1x pro Sekunde aktualisiert
- Zwischen Updates bleiben alte Bitmaps im VRAM

### 4. **Keine Pooling-Strategie für Bitmaps**
Jedes Mal wird ein komplett neues Bitmap erstellt und das alte disposed:
```csharp
// Kein Wiederverwendungs-Pool
// Kein Bitmap-Recycling
// Ständige Allokation/Deallokation → Fragmentierung
```

### 5. **PictureBox hält Image-Referenzen**
**Datei**: `src/Eve-O-Preview/View/Implementation/StaticThumbnailView.cs` (Zeilen 17-27)

```csharp
this._thumbnail = new StaticThumbnailImage
{
    TabStop = false,
    SizeMode = PictureBoxSizeMode.StretchImage,  // ← Skalierung in GPU!
    Location = new Point(0, 0),
    Size = new Size(this.ClientSize.Width, this.ClientSize.Height)
};
```

**Problem**:
- `PictureBoxSizeMode.StretchImage` bedeutet, dass die GPU die Skalierung übernimmt
- Originalbild bleibt in voller Größe im VRAM
- Zusätzlich: Skalierter Output im VRAM

---

## Optimierungsmöglichkeiten

### 🔴 **Priorität 1: Kritische Speicherlecks beheben**

#### 1.1 Bitmap-Disposal in WindowManager verbessern
**Problem**: Alte Image-Objekte könnten länger im Speicher bleiben als nötig

**Lösung**: Explizites Dispose-Pattern in `GetStaticThumbnail()`:

```csharp
// VORHER (WindowManager.cs):
Image image = Image.FromHbitmap(bitmap);
Gdi32NativeMethods.DeleteObject(bitmap);
return image;

// NACHHER:
// Option A: Caller muss dispose (dokumentieren!)
Image image = Image.FromHbitmap(bitmap);
Gdi32NativeMethods.DeleteObject(bitmap);
return image;  // Caller MUSS dispose!

// Option B: IDisposable-Wrapper erstellen
public class ManagedThumbnail : IDisposable { ... }
```

#### 1.2 Aggressive Garbage Collection für Bitmaps
**Datei**: `StaticThumbnailView.cs`

```csharp
protected override void RefreshThumbnail(bool forceRefresh)
{
    if (!forceRefresh || this.IsPreventPreviews())
    {
        return;
    }
    
    var thumbnail = this.WindowManager.GetStaticThumbnail(this.Id);
    if (thumbnail != null)
    {
        var oldImage = this._thumbnail.Image;
        this._thumbnail.Image = thumbnail;
        
        if (oldImage != null)
        {
            oldImage.Dispose();
            // NEU: Explizite GC-Freigabe für große Objekte
            GC.Collect(GC.MaxGeneration, GCCollectionMode.Optimized);
            GC.WaitForPendingFinalizers();
        }
    }
}
```

**Warnung**: Aggressive GC kann Performance beeinträchtigen! Nur wenn nötig.

### 🟡 **Priorität 2: Bitmap-Pooling implementieren**

#### 2.1 Bitmap-Pool für Wiederverwendung
**Neuer Service**: `IBitmapPool` Interface + Implementation

```csharp
// Neue Datei: Services/Interface/IBitmapPool.cs
public interface IBitmapPool
{
    Bitmap Rent(int width, int height);
    void Return(Bitmap bitmap);
}

// Neue Datei: Services/Implementation/BitmapPool.cs
public class BitmapPool : IBitmapPool
{
    private readonly ConcurrentBag<Bitmap> _pool = new();
    private readonly int _maxPoolSize = 20;  // Konfigurierbar!
    
    public Bitmap Rent(int width, int height)
    {
        // Versuche vorhandenes Bitmap aus Pool zu holen
        if (_pool.TryTake(out var bitmap) && 
            bitmap.Width == width && bitmap.Height == height)
        {
            return bitmap;
        }
        
        // Erstelle neues, wenn Pool leer oder Größe passt nicht
        return new Bitmap(width, height);
    }
    
    public void Return(Bitmap bitmap)
    {
        if (_pool.Count < _maxPoolSize)
        {
            _pool.Add(bitmap);
        }
        else
        {
            bitmap.Dispose();  // Pool voll, dispose
        }
    }
}
```

**Integration in WindowManager**:
```csharp
// WindowManager.cs - Constructor
private readonly IBitmapPool _bitmapPool;

public WindowManager(IThumbnailConfiguration configuration, IBitmapPool bitmapPool)
{
    this._bitmapPool = bitmapPool;
    // ...
}

// GetStaticThumbnail anpassen
public Image GetStaticThumbnail(IntPtr source)
{
    // ... bestehender Code bis bitmap-Erstellung ...
    
    // STATT: Image image = Image.FromHbitmap(bitmap);
    var managedBitmap = this._bitmapPool.Rent(width, height);
    
    using (var graphics = Graphics.FromImage(managedBitmap))
    {
        var hdc = graphics.GetHdc();
        Gdi32NativeMethods.BitBlt(hdc, 0, 0, width, height, sourceContext, 0, 0, Gdi32NativeMethods.SRCCOPY);
        graphics.ReleaseHdc(hdc);
    }
    
    Gdi32NativeMethods.DeleteObject(bitmap);
    return managedBitmap;
}
```

### 🟡 **Priorität 3: Bitmap-Skalierung optimieren**

#### 3.1 Pre-Scaling vor VRAM-Upload
**Problem**: PictureBox speichert Vollbild + skaliert in GPU

**Lösung**: CPU-seitige Skalierung auf Zielgröße

```csharp
// StaticThumbnailView.cs - RefreshThumbnail
protected override void RefreshThumbnail(bool forceRefresh)
{
    if (!forceRefresh || this.IsPreventPreviews())
    {
        return;
    }
    
    var fullSizeThumbnail = this.WindowManager.GetStaticThumbnail(this.Id);
    if (fullSizeThumbnail == null) return;
    
    // NEU: Skalierung auf PictureBox-Größe VOR dem Zuweisen
    var targetSize = this._thumbnail.Size;
    Bitmap scaledBitmap = null;
    
    if (fullSizeThumbnail.Width != targetSize.Width || 
        fullSizeThumbnail.Height != targetSize.Height)
    {
        scaledBitmap = new Bitmap(targetSize.Width, targetSize.Height);
        using (var graphics = Graphics.FromImage(scaledBitmap))
        {
            graphics.CompositingQuality = System.Drawing.Drawing2D.CompositingQuality.HighSpeed;
            graphics.InterpolationMode = System.Drawing.Drawing2D.InterpolationMode.Low;  // Schneller!
            graphics.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.HighSpeed;
            
            graphics.DrawImage(fullSizeThumbnail, 0, 0, targetSize.Width, targetSize.Height);
        }
        fullSizeThumbnail.Dispose();
    }
    else
    {
        scaledBitmap = (Bitmap)fullSizeThumbnail;
    }
    
    // Alte Bitmap entsorgen
    var oldImage = this._thumbnail.Image;
    this._thumbnail.Image = scaledBitmap;
    oldImage?.Dispose();
}
```

**VRAM-Ersparnis**: ~50-70% je nach Original-Größe!

### 🟢 **Priorität 4: Refresh-Rate intelligent anpassen**

#### 4.1 Adaptive Refresh-Rates
**Konfiguration**: `ThumbnailRefreshPeriod` ist bereits vorhanden!

**Empfehlung**:
```json
// EVE-O-Preview.json
{
  "ThumbnailRefreshPeriod": 1000,  // ← Von 500ms auf 1000ms erhöhen (Windows)
  // Für Linux: Minimum ist bereits 10ms
}
```

**Zusätzlich**: Idle-Detection implementieren

```csharp
// ThumbnailManager.cs - ThumbnailUpdateTimerTick
private DateTime _lastUserInteraction = DateTime.UtcNow;
private const int IDLE_THRESHOLD_SECONDS = 30;

private void ThumbnailUpdateTimerTick(object sender, EventArgs e)
{
    var idleTime = (DateTime.UtcNow - _lastUserInteraction).TotalSeconds;
    
    if (idleTime > IDLE_THRESHOLD_SECONDS)
    {
        // Reduziere Refresh-Rate bei Inaktivität
        // z.B. nur alle 5 Sekunden refreshen
        if (this._refreshCycleCount % 10 != 0)
        {
            this._refreshCycleCount++;
            return;
        }
    }
    
    this.UpdateThumbnailsList();
    this.RefreshThumbnails();
}
```

### 🟢 **Priorität 5: Thumbnail-on-Demand**

#### 5.1 Lazy-Loading für nicht sichtbare Thumbnails
```csharp
// ThumbnailView.cs - Refresh
public void Refresh(bool forceRefresh)
{
    // NEU: Nur refreshen wenn sichtbar
    if (!this.Visible) return;
    
    this.RefreshThumbnail(forceRefresh);
    this.HighlightThumbnail(forceRefresh || this._isSizeChanged);
    this.RefreshOverlay(forceRefresh || this._isSizeChanged || this._isLocationChanged);
    
    this._isSizeChanged = false;
}
```

---

## Zusammenfassung der Maßnahmen

### Sofort umsetzbar (Low-Hanging Fruit):

1. **Refresh-Rate erhöhen** (500ms → 1000ms)
   - Datei: `EVE-O-Preview.json`
   - Ersparnis: ~50% weniger Bitmap-Allokationen
   - Risiko: Minimal

2. **Pre-Scaling implementieren** (StaticThumbnailView)
   - Ersparnis: ~50-70% VRAM pro Thumbnail
   - Aufwand: ~2-3 Stunden
   - Risiko: Gering (nur CPU-Load etwas höher)

3. **Sichtbarkeits-Check** (ThumbnailView.Refresh)
   - Ersparnis: Abhängig von versteckten Thumbnails
   - Aufwand: 30 Minuten
   - Risiko: Minimal

### Mittelfristig (höherer Aufwand):

4. **Bitmap-Pooling** (neuer Service)
   - Ersparnis: Reduzierte Allokations-Overhead, weniger GC-Druck
   - Aufwand: ~1 Tag (Design + Implementation + Testing)
   - Risiko: Mittel (komplexe Zustandsverwaltung)

5. **Idle-Detection**
   - Ersparnis: Signifikant bei längerer Inaktivität
   - Aufwand: ~4 Stunden
   - Risiko: Gering

### Langfristig (architektonische Änderungen):

6. **DirectX/Direct2D Integration** (Alternative zu GDI+)
   - Ersparnis: Potenziell massiv (native GPU-Beschleunigung)
   - Aufwand: ~2-3 Wochen
   - Risiko: Hoch (große Code-Änderungen)

---

## Empfohlene Implementierungs-Reihenfolge

1. **Phase 1** (Quick Wins - 1 Tag):
   - Refresh-Rate auf 1000ms erhöhen
   - Pre-Scaling implementieren
   - Sichtbarkeits-Check hinzufügen

2. **Phase 2** (Performance-Boost - 1 Woche):
   - Bitmap-Pooling implementieren
   - Idle-Detection hinzufügen
   - Aggressive GC-Optimierungen (optional)

3. **Phase 3** (Zukunft - bei Bedarf):
   - DirectX/Direct2D Research
   - Hardware-Beschleunigung evaluieren

---

## Messung & Monitoring

### Zu trackende Metriken:
```csharp
// Neue Klasse: PerformanceMonitor.cs
public class PerformanceMonitor
{
    private long _totalBitmapsCreated;
    private long _totalBitmapsDisposed;
    private long _vramEstimate;  // Geschätzt basierend auf Bitmap-Größen
    
    public void RecordBitmapCreation(int width, int height)
    {
        _totalBitmapsCreated++;
        _vramEstimate += width * height * 4;  // RGBA = 4 bytes/pixel
    }
    
    public void RecordBitmapDisposal(int width, int height)
    {
        _totalBitmapsDisposed++;
        _vramEstimate -= width * height * 4;
    }
    
    public string GetReport()
    {
        return $"Created: {_totalBitmapsCreated}, Disposed: {_totalBitmapsDisposed}, " +
               $"Est. VRAM: {_vramEstimate / 1024 / 1024} MB, " +
               $"Leak Potential: {_totalBitmapsCreated - _totalBitmapsDisposed} bitmaps";
    }
}
```

---

## Konfigurationsoptionen für User

### Neue Config-Optionen (empfohlen):
```json
{
  "ThumbnailRefreshPeriod": 1000,           // Erhöhen für weniger VRAM-Nutzung
  "EnableBitmapPooling": true,               // Neu: Bitmap-Wiederverwendung
  "BitmapPoolMaxSize": 20,                   // Neu: Max Bitmaps im Pool
  "EnablePreScaling": true,                  // Neu: CPU-seitige Skalierung
  "IdleRefreshMultiplier": 5,                // Neu: Refresh-Rate bei Idle * 5
  "ForceRefreshCycleThreshold": 2            // Bereits vorhanden
}
```

### User-Szenarien:

**Szenario A: Maximale Performance (niedriger VRAM)**
```json
{
  "EnableWineCompatibilityMode": false,      // LiveThumbnailView nutzen!
  "ThumbnailRefreshPeriod": 1000,
  "ThumbnailSize": "320, 180"                // Kleinere Thumbnails
}
```

**Szenario B: RDP/Wine (hoher VRAM)**
```json
{
  "EnableWineCompatibilityMode": true,       // Screenshot-Modus nötig
  "EnablePreScaling": true,                  // VRAM sparen durch Skalierung
  "EnableBitmapPooling": true,               // Recycling aktivieren
  "ThumbnailRefreshPeriod": 1500             // Längeres Intervall
}
```

---

## Zusätzliche Ressourcen

### Relevante Dateien für Änderungen:
```
src/Eve-O-Preview/
├── Services/
│   ├── Implementation/
│   │   ├── WindowManager.cs              # ← GetStaticThumbnail() optimieren
│   │   └── ThumbnailManager.cs            # ← Refresh-Logik anpassen
│   └── Interface/
│       ├── IWindowManager.cs
│       └── IBitmapPool.cs                 # ← NEU: Pool-Interface
├── View/
│   └── Implementation/
│       ├── StaticThumbnailView.cs         # ← Pre-Scaling implementieren
│       └── LiveThumbnailView.cs           # ← Referenz (low VRAM)
└── Configuration/
    └── Implementation/
        └── ThumbnailConfiguration.cs      # ← Neue Config-Optionen
```

### Testing-Empfehlungen:
1. **Memory Profiler** nutzen (z.B. Visual Studio Diagnostic Tools)
2. **Process Explorer** von Sysinternals für VRAM-Monitoring
3. **Mehrere EVE-Clients simulieren** mit Eve-O-Mock
4. **Langzeit-Tests** (8+ Stunden) auf Memory Leaks prüfen

---

## Fazit

**Hauptproblem**: StaticThumbnailView (Screenshot-Modus) verbraucht ~3,6x mehr Speicher als LiveThumbnailView (DWM-Modus).

**Schnellste Lösung**: 
- Windows-User sollten `"EnableWineCompatibilityMode": false` verwenden (falls nicht in RDP)
- Pre-Scaling implementieren (größter Impact bei geringstem Risiko)
- Refresh-Rate erhöhen (500ms → 1000ms)

**Erwartete VRAM-Reduktion**: 50-70% bei StaticThumbnailView durch Kombination der Maßnahmen.
