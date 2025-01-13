Beim Verwenden der Heureka API können diverse Fehler auftreten. Zur Hilfe wurden daher hier die häufigsten dokumentiert.


## HTTP Fehler

Wir benutzen die standard HTTP Status Codes nach [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html), um erfolgreiche oder fehlerhafte Anfragen zu beantworten.
`2xx` Status codes sind erfolgreiche Anfragen, `4xx` sind Client-Fehler und `5xx` sind Server-Fehler.

Fehler der API entsprechen den [FHIRv4 Standard](https://build.fhir.org/operationoutcome.html) und sehen in der Regel wie folgt aus.

``` http
HTTP/1.1 404
Server: nginx
Date: Sun, 01 Jan 2000 12:00:00 GMT
Content-Type: application/fhir+json;charset=UTF-8

{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "processing",
      "diagnostics": "Provided id could not be read"
    }
  ]
}
```

Es kann dazu kommen, dass die Fehlermeldungen nicht diesem Standard entsprechen, sollte die Schnittstelle nicht einen FHIR Endpunkt betreffen.
Ein Beispiel wäre die `api-configuration` Schnittstelle.

Diese übrigen Fehler sind nach [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html) strukturiert.

### 400 Bad Request

Die Anfrage wurde falsch gestellt.
Mehr Informationen sind in der Regel im Body enthalten.

### 401 Unauthorized

Mögliche Ursachen:

- Es wurde kein Token mitgeliefert.
- Das Refreshtoken ist abgelaufen.
- Der Zugriff zur Praxis wurde entzogen.

### 403 Forbidden

Man besitzt keine Berechtigungen, um diese Ressource abfragen zu dürfen.

Mögliche Ursache:

- Die Praxis hat den Zugriff auf diese Ressource entzogen.

### 404 Not Found

Mögliche Ursachen:

- Die Ressource existiert nicht
- Die mitgelieferte ID ist falsch

### 429 Too Many Requests

Es wurden zu viele Anfragen in einem bestimmten Zeitraum gestellt.

### 500 Internal Server Error

Ein Server-interner Fehler ist aufgetreten.
Dieses Verhalten ist nicht erwartet und wird von uns überwacht.

### 502 Bad Gateway

Der Server kann nicht erreicht werden.
Bitte versuchen Sie es in ein paar Minuten erneut.

## Request Timeout

Anfragen an die FHIR-API laufen über einen HTTP-Proxy. Dieser hat ein Timeout von 10 Sekunden konfiguriert, nach welchem die Verbindung terminiert, falls keine Antwort erfolgt.

Wir empfehlen daher das Timeout des verwendeten API-Clients auf einen passenden Wert zu setzen, bzw. den Verbindungsabbruch entsprechend zu behandeln.

### Ursache

Ein Timeout bedeutet meistens, dass lediglich ein Primärsystem zurzeit nicht in der Lage ist zu antworten.
Möglicherweise kann die Anfrage nicht in dem gewünschten Zeitraum bearbeitet werden oder das Primärsystem ist nicht online.

### Mögliche Lösung

Es sollten Vorkehrungen getroffen werden, damit die Anfragen nicht in einer Endlosschleife enden.
Ein Exponential Backoff wäre daher eine mögliche Lösung.
Die Anfrage nach kleineren Datenmengen könnte weiterhin die Antwortzeit verbessern.