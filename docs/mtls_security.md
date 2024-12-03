
## Client-Zertifikate & Private Schlüssel

Die Heureka API erfordert eine mTLS Verbindung. Das dazu notwendige Zertifikat wird von Heureka im Austausch für einen CSR (Certificate Signing Request) erstellt.

### Privaten Schlüssel Erstellen

```bash
openssl genrsa -out my-heureka-app.key 4096
```

### Privaten Schlüssel Aufbewahren

Der private Schlüssel ist zentral für die sichere Kommunikation mit Heureka. Es ist deshalb wichtig, dass dieser möglichst sicher und idealerweise nur auf dem System, auf welchem er verwendet wird, zugänglich ist.

### Certificate Signing Request (CSR) Erstellen

```bash
openssl req -new -key my-heureka-app.key -nodes -out my-heureka-app.csr -subj "/CN=My Heureka App"
```