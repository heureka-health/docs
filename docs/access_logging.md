
### Access Logging

Um gegenüber dem HCP grösstmögliche Transparenz über die Verwendung der Daten zu gewährleisten, muss mit jedem API-Request ein entsprechender Request-Context via HTTP Header Werten mitgesendet werden. Diese Daten sind im Heureka Portal für 180 Tage sichtbar.

Kontext-Typ und Rollen werden als Teil des Onboardings zusammen mit Heureka definiert und API-seitig validiert.

| Header                       | Beschreibung                                                                                    | Beispiel                                          |
|------------------------------|-------------------------------------------------------------------------------------------------|---------------------------------------------------|
| X-HEUREKA-RequestContextId   | UUID v4                                                                                         | 000000000-1111-2222-3333-555555555555             |
| X-HEUREKA-RequestContextType | Beschreibt den fachlichen Kontext in dem der Zugriff stattfindet                                | MEDICATION_CHECK                                  |
| X-HEUREKA-UserRole           | Rolle die beschreibt, wer den Zugriff tätigt (oder SYSTEM für technische Zugriffe ohne User)    | PRACTICE                                          |
| X-HEUREKA-UserName           | Name oder Bezeichnung (oder leer für SYSTEM Zugriffe). Spezialzeichen als UTF-8 URL-encodieren. | `Dr. M%C3%BCller` (vor URL-Encoding: `Dr. Müller` | 


#### Beispiel

**Medikationszugriff**

Eine Anwendung erlaubt es Nutzer:innen nach Patienten zu suchen und deren aktuelle Medikation darzustellen, dazu wird ein Patient anhand der AHV Nummer gesucht und anschliessend die Medikation abgerufen. Da die beiden Requests fachlich zusammen gehören, wird derselbe Request-Context gesetzt.

```bash
GET /Patient?search...
X-HEUREKA-RequestContextId: 000000000-1111-2222-3333-555555555555
X-HEUREKA-RequestContextType: MEDICATION_CHECK
X-HEUREKA-UserRole: PRACTICE
X-HEUREKA-UserName: Dr. M%C3%BCller
```

```bash
GET /Medication?subject=PATIENT_ID
X-HEUREKA-RequestContextId: 000000000-1111-2222-3333-555555555555
X-HEUREKA-RequestContextType: MEDICATION_CHECK
X-HEUREKA-UserRole: PRACTICE
X-HEUREKA-UserName: Dr. M%C3%BCller
```
