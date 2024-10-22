# Testszenarien Für Services

Um User:innen ein möglichst gutes Erlebnis rund um die Verwendung von Heureka zu bieten, ist es sinnvoll verschiedene Szenarien zu berücksichtigen und zu testen.
Dies betrifft vor allem die Authentisierungs-Flows, die sowohl aus dem Heureka Portal als auch serviceseitig her initiiert werden können.
Diese sind nachfolgend beschrieben.

## Initiale Freigabe

Die initiale Freigabe der Praxis für den Service erfolgt immer serviceseitig, indem der/die Benutzer:in auf das Portal geleitet wird, um den Authorization Code Flow zu starten. 

## Trennen Der Verbindung

### Heureka-Portal

Der Service muss auf die Verbindungstrennung entsprechend reagieren und kann dem User beispielsweise die Möglichkeit bieten sich wieder mit Heureka zu verbinden.

Preview: Wenn der User Berechtigungen im Heureka Portal aktualisiert wird der Service darüber via Webhook informiert, wenn eine Webhook URL konfiguriert ist.

### Service

Wird das Trennen der Verbindung vom Service aus initiiert so wird der Benutzer anschliessend wieder an die angebgebe `redirect_uri` zurück zum Service geleitet. Der Service muss anschliessend überprüfen (mithilfe des API Configuration Endpunkts) ob die Verbindung erfolgreich getrennt wurde.


## Berechtigungen Aktualisieren

Wenn ein Service mit optionalen Berechtigungen registriert wurde, hat der Heureka User jederzeit die Möglichkeit dem Service diese Berechtigungen zu gewähren oder zu entziehen. 

### Heureka-Portal

Der Service muss damit umgehen können, dass sich die vergebenen Berechtigungen ändern können (mithilfe des API Configuration Endpunkts können die aktuellen Berechtigungen für die mit dem Accesstoken verbundene Installation abgefragt werden.)

Preview: Wenn der User Berechtigungen im Heureka Portal aktualisiert wird der Service darüber via Webhook informiert, wenn eine Webhook URL konfiguriert ist.

### Service

Wird eine Aktualisierung der Berechtigungen vom Service aus initialisiert, so wird der Benutzer anschliessend wieder via der angegeben `redirect_uri` zurück zum Service geleitet. Der Service muss anschliessend überprüfen (mithilfe des API Configuration Endpunkts) ob die Berechtigungen wie erwartet angepasst wurden.

## Testprotokoll
| # | Beschreibung                                    | Ablauf                                                                                                                                           | Erwartetes Resultat                                                                                                                |
|---|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| 1 | Zugriff erfolgreich beantragen                  | <ul><li>Benutzer wird auf `/authorization/grant` mit den entsprechenden Parametern geleitet.</li><li>User authorisiert Service</li></ul>         | Redirect an Service `redirect_uri` mit `code` Parameter um Access- & Refreshtoken zu lösen                                         |
| 2 | Zugriffsantrag abgebrochen                      | <ul><li>Benutzer wird auf `/authorization/grant` mit den entsprechenden Parametern geleitet.</li><li>User bricht ab</li></ul>                    | Redirect an Service `redirect_uri` mit `error=access_denied` Parameter                                                             |
| 3 | Berechtigungen aktualisieren                    | <ul><li>Benutzer wird auf `/authorization/update` mit den entsprechenden Parametern geleitet.</li><li>User aktualisiert Berechtigungen</li></ul> | Redirect an Service `redirect_uri`                                                                                                 |
| 4 | Aktualisieren der Berechtigungen abgebrochen    | <ul><li>Benutzer wird auf `/authorization/update` mit den entsprechenden Parametern geleitet.</li><li>User bricht ab</li></ul>                   | Redirect an Service `redirect_uri` mit `error=cancelled` Parameter                                                                 |
| 5 | Verbindung trennen                              | <ul><li>Benutzer wird auf `/authorization/revoke` mit den entsprechenden Parametern geleitet.</li><li>User entzieht berechtigungen</li></ul>     | Redirect an Service `redirect_uri`                                                                                                 |
| 6 | Verbindung trennen abgebrochen                  | <ul><li>Benutzer wird auf `/authorization/revoke` mit den entsprechenden Parametern geleitet.</li><li>User bricht ab</li></ul>                   | Redirect an Service `redirect_uri` mit `error=cancelled` Parameter                                                                 |
| 7 | Benutzer aktualisiert Berechtigungen via Portal | <ul><li>Benutzer verwendet Portal um zusätzliche Berechtigungen zu vergeben oder zu entziehen.</li></ul>                                         | Neue Berechtigungen werden nach kurzer Zeit (max. 60 Sekunden) aktiv und können via API Konfigurations Endpunkt abgerufen werden.  |
| 8 | Benutzer trennt Verbindung via Portal           | <ul><li>Benutzer verwendet Portal um die Verbindung vom Service zu trennen.</li></ul>                                                            | Refresh Token verliert Gültigkeit und Berechtigungen für aktives Access Token werden nach kurzer Zeit (max. 60 Sekunden) entzogen. |
