# Heureka Health

Heureka erlaubt Drittanbietern (_Heureka Services_) den sicheren Zugriff auf eine [FHIR (v4)](https://www.hl7.org/fhir/R4/) kompatible API Schnittstelle, um auf freigegebene Daten aus dem Primärsystem einer  Arztpraxis (Healthcare Provider, _HCP_) zuzugreifen.

## Service Registrierung

Um die Heureka API verwenden zu können, benötigt ein Service ein gültiges mTLS Client-Zertifikat sowie eine Freigabe der verwendeten Server-IP.
Ausserdem benötigt wird die Service-ID und es muss eine gültige Redirect-URI für den Authentisierungsvorgang eingerichtet sein, zudem müssen die entsprechenden [Request Context Parameter](#access_logging) definiert sein.

### Betriebsumgebungen

Heureka betreibt neben dem Produktivsystem eine Testumgebung gegen welche sämtliche Entwicklungsarbeiten stattfinden sollten.

| Endpunkt                 | Produktivsystem       | Testsystem                    |
|--------------------------|-----------------------|-------------------------------|
| Portal / Authentisierung | portal.heureka.health | portal.testing.heureka.health |
| Token (mTLS)             | token.heureka.health  | token.testing.heureka.health  |
| API (mTLS)               | api.heureka.health    | api.testing.heureka.health    |

## Autorisierung

### Zugriff Beantragen

Die Zugriffsfreigabe findet direkt zwischen einer der Praxis zugeordneten Person (HCP) und dem Heureka Service statt.
Dazu bietet Heureka einen OAuth2 kompatiblen Authorization Code Flow an, mit welchem die Datenfreigabe und die damit verbundene Ausstellung der notwendigen Access-Token initiiert wird.

Um den Authentisierungsvorgang einzuleiten, muss der Benutzer auf folgende URL geleitet werden.

```
https://portal.testing.heureka.health/authorization/grant?client_id=CLIENT_ID&state=RANDOM_ANTI_CSRF_STRING&redirect_uri=https://example.com/callback
```

#### Request Parameter

| Name          | Beschreibung                                                                          | 
|---------------|---------------------------------------------------------------------------------------|
| client_id     | Client ID des Service                                                                 |   
| state         | Optionales CSRF state token welches nach dem Redirect überprüft werden sollte         |
| redirect_uri  | URI auf welche nach erfolgreicher oder abgelehnter Authorisierung weitergeleitet wird |

#### Redirect Error Codes

Falls bei der Autorisierung ein Fehler auftritt, enthält die `redurect_uri` einen `error` Query-Parameter mit einem entsprechenden [Fehlercode](https://www.oauth.com/oauth2-servers/server-side-apps/possible-errors/).

---

Diese öffnet den Heureka-Authentisierungsdialog, in welchem sich der Praxisbenutzer anmelden und die vom Service benötigten Berechtigungen einsehen und bestätigen kann.

Anschliessend findet ein Redirect auf die im initialen Request angegebene Redirect-URL statt, welche den Code (Query Parameter `code`) enthält, der gegen ein kurzlebiges Access- und ein langlebiges Refreshtoken eingetauscht werden kann.

Der folgende und alle weiteren Aufrufe gegen die APIs von Heureka erfordern eine Authentisierung via mTLS.

```bash
curl -v --cert client.crt --key client.key -X POST "https://token.testing.heureka.health/oauth2/token" -d "grant_type=authorization_code" -d "client_id=CLIENT_ID" -d "redirect_uri=https://example.com/callback" -d "code=AUTH_CODE"
```

#### Request Parameter

| Name         | Beschreibung                                                | 
|--------------|-------------------------------------------------------------|
| client_id    | Client ID des Service                                       |
| grant_type   | Aktuell unterstützt wird `authorization_code`               |
| redirect_uri | Redirect URL die für den initialen request verwendet wurde. |
| code         | Authorization code aus dem vorherigen Schritt               |
 
#### Response

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJiMDI3ZmMwOS0xMjM0LTQzMjEtOGZhOS05YjM1NDJiZmJhNzEiLCJuYmYiOiIxNzI3MTg2NjkyIiwic2NvcGUiOiJbXCJvZmZsaW5lX2FjY2Vzc1wiXSIsImlzcyI6Imh0dHBzOi8vdG9rZW4udGVzdGluZy5oZXVyZWthLmhlYWx0aCIsImV4cCI6IjE3MjcxODY5OTIiLCJpYXQiOiIxNzI3MTg2NjkyIiwianRpIjoiMjgxNzBlNTQtMjA5YS00MjU0LTg3NWMtZmMwMDBlMjcwNzY2In0.Qx3XVXcOIc-h-ckLBW1hgsIJsFQUDMxiJzwwqfJihgo",
  "refresh_token": "MTHOCCRvbVcQUQNWALfjZCqD-G9Axv8Z78fYvRxEc7RIQ7-XYZ-DnICIFH7wxE7MIcE67ejvwiHavyzbRm3V_iYvdZOZqhGctXjvo4fCwFfqo2om2-OgP132FugDw6bO",
  "scope": "offline_access",
  "token_type": "Bearer",
  "expires_in": 299
}
```

```mermaid
sequenceDiagram
    participant u as User
    participant s as Heureka Service
    participant h as Heureka
    participant a as Heureka FHIR API
    s->>h: /authorization/grant request
    h->>u: Authentication prompt
    u->>h: Authentication and consent
    h->>s: Authorization code response
    s->>h: Request token using auth code & mTLS certificate
    h->>s: Access & request token
    s->>s: store request token together with installation ID (sub claim from access token)
    s->>a: requests via access token & mTLS certificate
    a->>s: response
```

#### Installation ID

Das Accesstoken enthält einen Claim `sub` welcher die **Installations ID** beinhaltet. Diese ID bezeichnet eindeutig die Freigabe der jeweiligen Praxis für den Service und sollte serviceseitig zusammen mit dem Access- und Refreshtoken abgelegt werden. Die Installation ID wird wieder benötigt, um die Berechtigungs-Flows zu starten oder um Webhooks empfangen zu können.

#### Token-Refresh

Access Tokens sind kurzlebig und müssen daher mithilfe des Refresh-Tokens nach Ablauf erneuert werden.

```bash
curl -v --cert client.crt --key client.key -X POST "https://token.testing.heureka.health/oauth2/token" -d "grant_type=refresh_token" -d "client_id=SERVICE_ID" -d "refresh_token=REFRESH_TOKEN"
```

### Berechtigungen Aktualisieren

Beim Autorisierungsschritt erteilt der/die Benutzer:in dem Service Zugriff auf bestimmte Ressourcen (Bspw. Patientendaten, Medikation, etc.). Diese werden im Rahmen des Onboarding zwischen Service & Heureka definiert. Des Weiteren können, auch im Nachhinein, optionale Berechtigungen definiert werden, welche zusätzlich freigegeben werden können.

Wenn der Service nachträglich zusätzliche Berechtigungen anfordern (oder abgeben) möchte ist dies möglich, indem der Benutzer auf die folgende Seite geleitet wird auf der/die Benutzer:in zusätzliche Berechtigungen vergeben oder Berechtigungen entziehen.

```bash 
https://portal.testing.heureka.health/authorization/update?installation_id=INSTALLATION_ID&redirect_uri=https://example.com/callback
```

#### Request Parameter

| Name            | Beschreibung                                           | 
|-----------------|--------------------------------------------------------|
| installation_id | Installation ID aus dem Access Token                   |   
| redirect_uri    | URI an die nach erfolgter Freigabe weitergeleitet wird |

#### Redirect Error Codes

Falls bei der Autorisierung ein Fehler auftritt, enthält die `redurect_uri` einen `error` Query-Parameter mit einem entsprechenden [Fehlercode](https://www.oauth.com/oauth2-servers/server-side-apps/possible-errors/).

Wird der Vorgang abgebrochen, enthält der `error` Parameter den Wert `cancelled`.

--- 

Benutzer:innen haben immer auch die Möglichkeit, via Heureka Portal einem Service die Berechtigung zu entziehen.

```mermaid
sequenceDiagram
    participant u as User
    participant s as Heureka Service
    participant h as Heureka
    s->>h: /authorization/update request
    h->>u: Authentication prompt
    u->>h: Authentication and update grants
    h->>s: Redirect to redirect_uri
    s->>h: check current grants via api configuration endpoint
    h->>s: response
```

### Verbindung Trennen

Um den Benutzer:innen die Möglichkeit zu geben den Zugriff auf freigegebene Daten vollständig zu entziehen, kann auf folgende Seite weitergeleitet werden.

```bash
https://portal.testing.heureka.health/authorization/revoke?installation_id=INSTALLATION_ID&redirect_uri=https://example.com/callback
```

#### Request Parameter

| Name            | Beschreibung                                                   | 
|-----------------|----------------------------------------------------------------|
| installation_id | Installation ID aus dem Access Token                           |   
| redirect_uri    | URI an die nach dem Trennen der Verbindung weitergeleitet wird |

#### Redirect Error Codes

Falls bei der Autorisierung ein Fehler auftritt, enthält die `redurect_uri` einen `error` Query-Parameter mit einem entsprechenden [Fehlercode](https://www.oauth.com/oauth2-servers/server-side-apps/possible-errors/).

Wird der Vorgang abgebrochen, enthält der `error` Parameter den Wert `cancelled`.

---

Benutzer:innen haben immer auch die Möglichkeit die Verbindung aus dem Heureka Portal zu trennen.

```mermaid
sequenceDiagram
    participant u as User
    participant s as Heureka Service
    participant h as Heureka
    s->>h: /authorization/revoke request
    h->>u: Authentication prompt
    u->>h: Authentication and revoke grants
    h->>s: Redirect to redirect_uri
```

## API Verwendung

Nachdem der Service den Autorisierungsvorgang für eine Praxis erfolgreich abgeschlossen und damit ein gültiges Accesstoken erhalten hat können Requests gegen die API ausgeführt werden.

Die Peer-to-peer Architektur von Heureka ermöglicht den direkten und sicheren Zugriff auf die Daten des Praxisinformationssystems in dem alle Anfragen via eines Proxy-Setups direkt auf das Primärsystem gelangen.


### API Konfiguration

Der API-Konfigurationsendpunkt liefert zusammen mit der HCP-spezifischen URL für die FHIR API auch die Proxykonfiguration zurück, welche für Requests gegen die FHIR-API verwendet werden muss.
Ausserdem enthält die API-Konfiguration die aktuell für die Installation gültigen Berechigungen (Grants).

```json
{
    "fhirEndpoint": "https://6a5da575-6ac0-4e06-b9a5-a2210e30337c.api.testing.heureka.health/fhir",
    "proxy": "http://tunnel.testing.heureka.health:7000",
    "grants": {
        "PATIENT": [
          "READ",
          "SEARCH"
        ],
        "MEDICATION_STATEMENT": [
          "SEARCH"
        ]
    }
}
```

### <a name="access_logging"></a>Access Logging

Um gegenüber dem HCP grösstmögliche Transparenz über die Verwendung der Daten zu gewährleisten, muss mit jedem API-Request ein entsprechender Request-Context via HTTP Header Werten mitgesendet werden. Diese Daten sind im Heureka Portal für 180 Tage sichtbar.

Kontext-Typ und Rollen werden als Teil des Onboardings zusammen mit Heureka definiert und API-seitig validiert.

| Header                       | Beschreibung                                                                                 | Beispiel                              |
|------------------------------|----------------------------------------------------------------------------------------------|---------------------------------------|
| X-HEUREKA-RequestContextId   | UUID v4                                                                                      | 000000000-1111-2222-3333-555555555555 |
| X-HEUREKA-RequestContextType | Beschreibt den fachlichen Kontext in dem der Zugriff stattfindet                             | MEDICATION_CHECK                      |
| X-HEUREKA-UserRole           | Rolle die beschreibt, wer den Zugriff tätigt (oder SYSTEM für technische Zugriffe ohne User) | PRACTICE                              |
| X-HEUREKA-UserName           | Name oder Bezeichnung (oder leer für SYSTEM Zugriffe)                                        | Dr. Müller                            | 


#### Beispiel

**Medikationszugriff**

Ein Service erlaubt es Nutzer:innen nach Patienten zu suchen und deren aktuelle Medikation darzustellen, dazu wird ein Patient anhand der AHV Nummer gesucht und anschliessend die Medikation abgerufen. Da die beiden Requests fachlich zusammen gehören, wird derselbe Request-Context gesetzt.

```bash
GET /Patient?search...
X-HEUREKA-RequestContextId: 000000000-1111-2222-3333-555555555555
X-HEUREKA-RequestContextType: MEDICATION_CHECK
X-HEUREKA-UserRole: PRACTICE
X-HEUREKA-UserName: Dr. Müller
```

```bash
GET /Medication?subject=PATIENT_ID
X-HEUREKA-RequestContextId: 000000000-1111-2222-3333-555555555555
X-HEUREKA-RequestContextType: MEDICATION_CHECK
X-HEUREKA-UserRole: PRACTICE
X-HEUREKA-UserName: Dr. Müller
```

## Webhooks (Preview)

Um Services über Änderungen ihrer Installationen zu informieren, wird Heureka zukünftig Webhooks implementieren mit deren Hilfe Events wir das Updaten/Entziehen der Berechtigungen in Echtzeit an den Service übermittelt werden.


## Client-Zertifikate & Private Schlüssel

Die Heureka API erfordert eine mTLS Verbindung. Das dazu notwendige Zertifikat wird von Heureka im Austausch für einen CSR (Certificate Signing Request) erstellt.

### Privaten Schlüssel Erstellen

```bash
openssl genrsa -out my-heureka-service.key 4096
```

### Privaten Schlüssel Aufbewahren

Der private Schlüssel ist zentral für die sichere Kommunikation mit Heureka. Es ist deshalb wichtig, dass dieser möglichst sicher und idealerweise nur auf dem System, auf welchem er verwendet wird, zugänglich ist.

### Certificate Signing Request (CSR) Erstellen

```bash
openssl req -new -key my-heureka-service.key -nodes -out my-heureka-service.csr -subj "/CN=My Heureka Service"
```


## Ressourcen

* [Postman collection](resources/heureka_service.postman_collection.json)


