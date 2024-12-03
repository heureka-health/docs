
## Webhooks

Um Anwendungen über Änderungen ihrer Installationen zu informieren, verwendet Heureka Webhooks.

Webhooks sind POST-Requests an einen vorher festgelegten Endpunkt.

Für den Empfang von Webhooks von Heureka könnte die URL dazu beispielsweise so aussehen: https://www.example.com/heureka/webhooks/.

Die erfolgreiche Verarbeitung eines Webhooks wird durch die Rückgabe einer 2xx-Antwort (Statuscode 200-299) auf die Webhook-Nachricht innerhalb von maximal 15 Sekunden angezeigt.

Ein wichtiger Aspekt bei der Handhabung von Webhooks ist die Überprüfung der Signatur und des Zeitstempels bei der Verarbeitung der Nachricht.

Mehr dazu im Abschnitt [Verifikation](#verifikation).

### Setup

Um Webhooks verwenden zu können, muss ein Endpunkt bei uns registriert werden. Dazu einfach eine E-Mail an [support@heureka.health](mailto:support@heureka.health) mit der entsprechenden Ziel-URL und dem Namen der Anwendung senden. Wir aktivieren daraufhin die Webhooks für die Anwendung und stellen das Signing-Secret für die Verifikation der Daten zur Verfügung.

### Events

Die Eventtypen, die von Heureka via Webhook versendet werden, sind hier dokumentiert:

https://www.svix.com/event-types/eu/org_2onSGfPZRBtRVpRKxm1v1jATLgs/

### Verifikation

Mit der Überprüfung der Webhook-Signatur kann festgestellt werden, dass Webhook-Nachrichten tatsächlich von uns und nicht von einem böswilligen Akteur gesendet wurden.

Eine detailliertere Erklärung und Links zur Anleitung findet sich in diesem Artikel über [warum Sie Webhooks verifizieren sollten](https://docs.svix.com/receiving/verifying-payloads/why).

### Fehlerbehandlungen

Wenn ein Webhook nicht mit einem 2xx-Statuscode quittiert wird, wird nach einem festgelegten Zeitplan versucht die Nachricht erneut zuzustellen. Dabei ist zu beachten, dass Nachrichten nach einer bestimmten Zeit für die Anwendung nicht mehr relevant sein könnten. Die Nachrichten enthalten einen Zeitstempel, anhand dessen das Alter der Nachricht überprüft werden kann.

Der Zeitplan für die wiederholten Zustellversuche:

-   Erster Versuch
-   5 Sekunden
-   5 Minuten
-   30 Minuten
-   2 Stunden
-   5 Stunden
-   10 Stunden
-   10 Stunden

Wenn alle Nachrichten an einen Endpunkt an 5 aufeinanderfolgenden Tagen nicht zugestellt werden können, wird der Endpunkt automatisch deaktiviert.
