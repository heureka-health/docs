## Anwendungs-Registrierung

Um die Heureka API verwenden zu können, benötigt eine Anwendung 

* ein gültiges mTLS Client-Zertifikat 
* eine Freigabe der verwendeten Server-IP.

Ausserdem benötigt wird die Client-ID und es muss eine gültige Redirect-URI für den Authentisierungsvorgang eingerichtet sein, zudem müssen die entsprechenden [Request Context Parameter](./access_logging.md) definiert sein.

### Betriebsumgebungen

Heureka betreibt neben dem Produktivsystem eine Testumgebung gegen welche sämtliche Entwicklungsarbeiten stattfinden sollten.

| Endpunkt                 | Produktivsystem       | Testsystem                    |
|--------------------------|-----------------------|-------------------------------|
| Portal / Authentisierung | portal.heureka.health | portal.testing.heureka.health |
| Token (mTLS)             | token.heureka.health  | token.testing.heureka.health  |
| API (mTLS)               | api.heureka.health    | api.testing.heureka.health    |

## Autorisierung

### Zugriff Beantragen

Die Zugriffsfreigabe findet direkt zwischen einer der Praxis zugeordneten Person (HCP) und der Heureka Anwendung statt.
Dazu bietet Heureka einen OAuth2 kompatiblen Authorization Code Flow an, mit welchem die Datenfreigabe und die damit verbundene Ausstellung der notwendigen Access-Token initiiert wird.

Um den Authentisierungsvorgang einzuleiten, muss der Benutzer auf folgende URL geleitet werden.

```
https://portal.testing.heureka.health/authorization/grant?client_id=CLIENT_ID&state=RANDOM_ANTI_CSRF_STRING&redirect_uri=https://example.com/callback
```

#### Request Parameter

| Name          | Beschreibung                                                                          | 
|---------------|---------------------------------------------------------------------------------------|
| client_id     | Client ID der Anwendung                                                               |   
| state         | Optionales CSRF state token welches nach dem Redirect überprüft werden sollte         |
| redirect_uri  | URI auf welche nach erfolgreicher oder abgelehnter Authorisierung weitergeleitet wird |

#### Redirect Error Codes

Falls bei der Autorisierung ein Fehler auftritt, enthält die `redurect_uri` einen `error` Query-Parameter mit einem entsprechenden [Fehlercode](https://www.oauth.com/oauth2-servers/server-side-apps/possible-errors/).

---

Diese öffnet den Heureka-Authentisierungsdialog, in welchem sich der Praxisbenutzer anmelden und die von der Anwendung benötigten Berechtigungen einsehen und bestätigen kann.

Anschliessend findet ein Redirect auf die im initialen Request angegebene Redirect-URL statt, welche den Code (Query Parameter `code`) enthält, der gegen ein kurzlebiges Access- und ein langlebiges Refresh-Token eingetauscht werden kann.

Der folgende und alle weiteren Aufrufe gegen die APIs von Heureka erfordern eine Authentisierung via mTLS.

```bash
curl -v --cert client.crt --key client.key -X POST "https://token.testing.heureka.health/oauth2/token" -d "grant_type=authorization_code" -d "client_id=CLIENT_ID" -d "redirect_uri=https://example.com/callback" -d "code=AUTH_CODE"
```

#### Request Parameter

| Name         | Beschreibung                                                | 
|--------------|-------------------------------------------------------------|
| client_id    | Client ID der Anwendung                                     |
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
    participant s as Heureka App
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

Das Access-Token enthält einen Claim `sub` welcher die **Installations ID** beinhaltet. Diese ID bezeichnet eindeutig die Freigabe der jeweiligen Praxis für die Anwendung und sollte anwendungsseitig zusammen mit dem Access- und Refresh-Token abgelegt werden. Die Installation ID wird wieder benötigt, um die Berechtigungs-Flows zu starten oder um Webhooks empfangen zu können.

#### Token-Refresh

Access-Tokens sind kurzlebig und müssen daher mithilfe des Refresh-Tokens nach Ablauf erneuert werden.

```bash
curl -v --cert client.crt --key client.key -X POST "https://token.testing.heureka.health/oauth2/token" -d "grant_type=refresh_token" -d "client_id=CLIENT_ID" -d "refresh_token=REFRESH_TOKEN"
```

### Berechtigungen Aktualisieren

Beim Autorisierungsschritt erteilt der/die Benutzer:in der Anwendung Zugriff auf bestimmte Ressourcen (Bspw. Patientendaten, Medikation, etc.). Diese werden im Rahmen des Onboardings zwischen Anwendung & Heureka definiert. Des Weiteren können, auch im Nachhinein, optionale Berechtigungen definiert werden, welche zusätzlich freigegeben werden können.

Wenn die Anwendung nachträglich zusätzliche Berechtigungen anfordern (oder abgeben) möchte ist dies möglich, indem der Benutzer auf die folgende Seite geleitet wird auf der/die Benutzer:in zusätzliche Berechtigungen vergeben oder Berechtigungen entziehen.

```bash 
https://portal.testing.heureka.health/authorization/update?installation_id=INSTALLATION_ID&redirect_uri=https://example.com/callback
```

#### Request Parameter

| Name            | Beschreibung                                           | 
|-----------------|--------------------------------------------------------|
| installation_id | Installation ID aus dem Access-Token                   |   
| redirect_uri    | URI an die nach erfolgter Freigabe weitergeleitet wird |

#### Redirect Error Codes

Falls bei der Autorisierung ein Fehler auftritt, enthält die `redurect_uri` einen `error` Query-Parameter mit einem entsprechenden [Fehlercode](https://www.oauth.com/oauth2-servers/server-side-apps/possible-errors/).

Wird der Vorgang abgebrochen, enthält der `error` Parameter den Wert `cancelled`.

--- 

Benutzer:innen haben immer auch die Möglichkeit, via Heureka Portal einer Anwendung die Berechtigung zu entziehen.

```mermaid
sequenceDiagram
    participant u as User
    participant s as Heureka App
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
| installation_id | Installation ID aus dem Access-Token                           |   
| redirect_uri    | URI an die nach dem Trennen der Verbindung weitergeleitet wird |

#### Redirect Error Codes

Falls bei der Autorisierung ein Fehler auftritt, enthält die `redurect_uri` einen `error` Query-Parameter mit einem entsprechenden [Fehlercode](https://www.oauth.com/oauth2-servers/server-side-apps/possible-errors/).

Wird der Vorgang abgebrochen, enthält der `error` Parameter den Wert `cancelled`.

---

Benutzer:innen haben immer auch die Möglichkeit die Verbindung aus dem Heureka Portal zu trennen.

```mermaid
sequenceDiagram
    participant u as User
    participant s as Heureka App
    participant h as Heureka
    s->>h: /authorization/revoke request
    h->>u: Authentication prompt
    u->>h: Authentication and revoke grants
    h->>s: Redirect to redirect_uri
```

## API Verwendung

Nachdem die Anwendung den Autorisierungsvorgang für eine Praxis erfolgreich abgeschlossen und damit ein gültiges Access-Token erhalten hat können Requests gegen die API ausgeführt werden.

Die Peer-to-peer Architektur von Heureka ermöglicht den direkten und sicheren Zugriff auf die Daten des Praxisinformationssystems in dem alle Anfragen via eines Proxy-Setups direkt auf das Primärsystem gelangen.

### <a name="api_configuration"></a>API Konfiguration

Der API-Konfiguration-Endpunkt liefert zusammen mit der HCP-spezifischen URL für die FHIR-API auch die Proxykonfiguration zurück, welche für Requests gegen die FHIR-API verwendet werden muss.
Ausserdem enthält die API-Konfiguration die aktuell für die Installation gültigen Berechtigungen (Grants).

Dieser Endpunkt liefert immer die **aktuell gültigen** Berechtigungen für das verwendete Token zurück. Er sollte dazu verwendet werden, um sicherzustellen, dass die Awendung die erforderlichen Berechtigungen besitzt.

!!! tip ""

    Als Best Practice wird empfohlen, die gültigen Berechtigungen vor der ersten User-Interaktion zu überprüfen, um damit sicherzustellen, dass nachfolgende API-Zugriffe erlaubt sind. Auch sollte nach dem Ende des Grant-/Update-/Revoke-Flows der aktuelle Stand der Berechtigungen anhand des Endpunkts überprüft werden.

!!! info "Hinweis"

    Es ist in seltenen Fällen möglich, dass eine Differenz zwischen dem Resultat des API-Konfiguration-Endpunkts und den Berechtigungen bei der Abfrage von Daten besteht, da die Berechtigungen nicht unmittelbar synchronisiert werden. Die "Source of Truth" ist in dem Fall immer der API-Konfiguration-Endpunkt und nach spätestens 60 Sekunden sollten die Systeme wieder eine einheitliche Datengrundlage haben.

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

