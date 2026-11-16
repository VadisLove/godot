# UMA-Textur-Ladepfad – Projektnotizen (Bachelorarbeit)

Branch `uma-textures` im Fork https://github.com/VadisLove/godot

## Forschungsfrage (Kurzfassung)

Um wie viel senkt ein an Apple-Silicon-Unified-Memory angepasster Textur-Ladepfad im
Metal-Backend von Godot den maximalen Prozessspeicher und die Zeit bis zur ersten
korrekten Darstellung – bei gleicher Darstellungsqualität? Methode: Design Science Research.

## Festgelegte Godot-Version

- Basis: `master` Commit **`6ef60dc279`** (02.09.2026, 4.8-dev).
- Warum nicht `4.7.2-stable`: Der Buffer-Sonderweg `_get_buffer_alloc_type`, auf den sich
  die Motivation stützt, kam erst mit PR #120088 (gemergt 18.08.2026) und ist in keinem
  stabilen Release enthalten.
- In den Texten `d824ca9` durch `6ef60dc279` ersetzen (d824ca9 existiert lokal nicht).

## Idee

```
Standard:  Image (RAM) --memcpy--> Staging-Buffer --Blit (GPU)--> private MTLTexture
UMA-Pfad:  Image (RAM) --replaceRegion (CPU)------------------> shared  MTLTexture
```

Staging-Buffer, Blit-Befehl und Warten auf den Transfer-Worker fallen weg. Preis: Die
Textur ist `Shared` statt `Private`, Metal darf sie weniger für die GPU optimieren.
Diesen Zielkonflikt misst die Arbeit.

## Session 16./17.11.2026 – was geändert wurde und wozu

Zeilen beziehen sich auf den Stand nach Commit „Meilenstein 1“.

### 1. Schnittstelle (für alle Treiber)

| Datei | Zeile | Änderung | Wozu |
|---|---|---|---|
| `servers/rendering/rendering_device_commons.h` | 445 | `TextureFormat::is_gpu_mappable = false` | „Bestellzettel“-Feld: diese Textur soll die CPU direkt beschreiben |
| `servers/rendering/rendering_device_commons.h` | 1042 | Feature `SUPPORTS_GPU_MAPPABLE_TEXTURE` | RenderingDevice kann den Treiber fragen, ob er das kann |
| `servers/rendering/rendering_device_driver.h` | 293–294 | virtuelle Methode `texture_write_direct(...)`, Standard `ERR_UNAVAILABLE` | Vulkan/D3D12 erben die leere Version und bleiben unverändert |

### 2. Metal-Treiber

| Datei | Zeile | Änderung | Wozu |
|---|---|---|---|
| `drivers/metal/rendering_device_driver_metal.h` | 256 | Deklaration `texture_write_direct ... override final` | Metal überschreibt die leere Basisversion |
| `drivers/metal/rendering_device_driver_metal.cpp` | 350–353 | `texture_create`: `else if (p_format.is_gpu_mappable)` → `StorageModeShared` | Textur wird CPU-beschreibbar angelegt (nicht linear, Mipmaps bleiben möglich) |
| `drivers/metal/rendering_device_driver_metal.cpp` | 690–703 | neue Methode `texture_write_direct` | prüft Shared + `p_data`, schreibt eine Mipmap-Stufe per `MTL::Texture::replaceRegion` |
| `drivers/metal/rendering_device_driver_metal.cpp` | 2840–2841 | `has_feature`: `SUPPORTS_GPU_MAPPABLE_TEXTURE` → `device->hasUnifiedMemory()` | Schnellweg nur, wenn die Hardware wirklich Unified Memory meldet |

### 3. RenderingDevice (verbindet alles)

| Datei | Zeile | Änderung | Wozu |
|---|---|---|---|
| `servers/rendering/rendering_device.cpp` | 1766–1767 | `format.is_gpu_mappable = feature && Startdaten && keine GPU-Schreib-Bits` | Entscheidung: welche Texturen den UMA-Pfad nehmen (Render-Ziele, Storage usw. bleiben Private) |
| `servers/rendering/rendering_device.h` | 375 | `Texture::is_gpu_mappable = false` | Textur „merkt“ sich die Entscheidung |
| `servers/rendering/rendering_device.cpp` | 1786 | `texture.is_gpu_mappable = format.is_gpu_mappable;` | Wert vom Bestellzettel in die Textur übernehmen |
| `servers/rendering/rendering_device.cpp` | 2203–2228 | `_texture_initialize`: Schnellweg mit Mipmap-Schleife → `driver->texture_write_direct(...)`, dann `return OK` | **Herzstück:** Staging-Weg wird für diese Texturen übersprungen |
| `servers/rendering/rendering_device.cpp` | 2226 | `print_verbose("UMA direct upload: ...")` | Nachweis im Log (nur mit `--verbose`) |

## Erste Ergebnisse (Meilenstein 1: Funktionsnachweis)

Testumgebung: Apple M4 Pro (Apple9), Metal 4.0, Forward+, TPS-Demo im Editor,
Debug-Build (`dev_build=yes`, `-O0`), Build `99b65a97d` + Änderungen aus Punkt 3.

- Demo startet ohne Fehler, keine Meldungen aus `texture_write_direct`.
- **Texturen sehen korrekt aus** (visuell geprüft, keine Streifen/Farbfehler).
- **219 Texturen** liefen über den UMA-Pfad (Zählung der `UMA direct upload`-Zeilen):

| Größe | Mipmaps | Anzahl |
|---|---|---|
| 4096×4096 | 13 | 12 |
| 2048×2048 | 12 | 39 |
| 1024×1024 | 11 | 47 |
| 512×512 | 10 | 10 |
| ≤ 256 (diverse) | 1–9 | 49 |
| 4×4 (Godot-Standardtexturen) | 1 | 62 |

- Die Demo-Texturen sind **S3TC (blockkomprimiert)**. Die Pitch-Berechnung funktioniert
  also auch für Blockformate. Laut Apple („Optimizing texture data“) wendet Metal auf
  blockkomprimierte Texturen ohnehin keine verlustfreie Kompression an → Hypothese:
  Render-Nachteil von Shared ist hier gering.
- Beobachtung: max. ~43 FPS. **Nicht aussagekräftig**, da Debug-Build (`-O0`) im Editor
  und noch kein Vergleich mit dem Standardpfad.

Log der Demo: `~/Library/Application Support/Godot/app_userdata/Third-Person Shooter Demo/logs/godot.log`

## Offen / nächste Schritte

1. **Punkt 4 – Schalter:** Standard- vs. UMA-Pfad in derselben Binärdatei umschaltbar
   (z. B. Projekteinstellung oder Kommandozeile).
2. **Punkt 5 – Messaufbau:** optimierter Build (`dev_build=no optimize=speed` bzw.
   `target=template_release`), Spitzen-Speicher (`/usr/bin/time -l`, Instruments) und
   Zeit bis zum ersten korrekten Bild, mehrere Wiederholungen.
3. Variante V2 prüfen: zusätzlich `optimizeContentsForGPUAccess` nach dem Schreiben.
4. Aufräumen: Platzhalter-Kommentar `// ??? ← HIER...` (rendering_device.cpp:2218) ersetzen,
   Kommentare auf Englisch, Einrückung 2205–2227, `driver->`/`texture->` ohne Leerzeichen,
   `clang-format`.
5. `texture_write_direct` (Metal): Prüfung auf `rasterization_rate_map` ergänzen;
   Rückgabewert in `_texture_initialize` auswerten.
6. Hilfsmittel: `compile_commands.json` neu erzeugen mit
   `scons platform=macos target=editor dev_build=yes compiledb=yes compiledb_gen_only=yes`.
