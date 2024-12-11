# Changelog

### 2024-12-11

**Gültige AHV Nummern**

Neu werden AHV-Nummern sowohl bei der Ein- als auch bei der Ausgabe validiert.

Ungültige AHV-Nummern werden von der API-Response entfernt. 
Die AHV-Nummern werden bei der Ausgabe ausserdem normalisiert, d.h. sie enthalten keine Trennpunkte.

### 2024-11-26

**Webhooks**

Heureka unterstützt jetzt [Webhooks](./webhooks.md), um Anwendungen aktiv über Änderungen ihrer Berechtigungen zu informieren.
Mehr Informationen dazu in der [Dokumentation](./webhooks.md).
