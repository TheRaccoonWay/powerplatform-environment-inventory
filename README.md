# Environment Inventory

Inventarisiert alle Power-Platform-Umgebungen eines Tenants (Typ, Managed ja/nein, Ersteller, Solutions, lose Objekte außerhalb von Solutions) und soll ausgewählte Solutions später in ein Azure-DevOps-Repo sichern. Mandantenfähig: Zugangsdaten kommen aus einer Variable Group (bisher in einem Tenant getestet).

## Struktur

```
.
├── README.md
└── pipelines/
    ├── inventory.yml     ← liest Umgebungen und Solutions und schreibt sie in die Listen
    └── setup-lists.yml   ← legt die SharePoint-Listen "Environments" und "Solutions" an (zuerst ausführen)
```

Umgesetzt: Umgebungen in die Liste `Environments` und Solutions in die Liste `Solutions` schreiben, Inhalt der Solutions, Zugriffsentscheidung (Application User) und Sicherungsentscheidung über die Listen. Geplant: `backup.yml` für den Export ins Repo (noch nicht umgesetzt), Umgebungen ohne Dataverse (Apps und Flows über die Admin-APIs von Power Apps und Power Automate, der Zugriff der App ist vorher zu prüfen), später eine Liste "Publisher".

## Einrichtung

1. Variable Group **`Inventory-Secrets`** (Pipelines › Library) mit `TenantId`, `ClientId`, `ClientSecret` (als Secret) und `SiteId` (SharePoint-Site, Format `hostname,site-collection-guid,web-guid`) anlegen.
2. Die App Registration braucht die Registrierung als **Power-Platform-Management-App** (`pac admin application register --application-id <ClientId>`, ausgeführt von einem Administrator, [Doku](https://learn.microsoft.com/power-platform/admin/powershell-create-service-principal); die App darf damit die Umgebungen des Tenants wie ein Power Platform Administrator verwalten), die Graph-Berechtigung **User.Read.All** (Application, mit Admin-Zustimmung, für den Owner) und auf der SharePoint-Site **Sites.Selected** mit der Site-Rolle **`manage`** (siehe Abschnitt "Voraussetzung: Site-Berechtigung"). Für `inventory.yml` allein genügt später `write`.
3. Pipelines anlegen: Pipelines › New pipeline › Azure Repos Git › Existing YAML file › `/pipelines/inventory.yml` bzw. `/pipelines/setup-lists.yml`. Die Pipelines laufen in Azure DevOps. Liegt der Code auf GitHub, wählst du bei "Where is your code?" stattdessen **GitHub** (Azure Pipelines braucht dann Zugriff auf das Repo) oder kopierst die YAML-Dateien in ein Azure-DevOps-Repo.
4. `inventory.yml`: erst `setup-lists.yml` ausführen, damit die Listen existieren. Dann läuft `inventory.yml` und trägt jede Umgebung in die Liste `Environments` und jede Solution in die Liste `Solutions` ein (Parameter `writeToLists`, Standard an). Der Lauf ist wiederholbar: Er aktualisiert nur die Spalten der Pipeline und lässt "Zugriffsentscheidung", "Sicherungsentscheidung", "Maßnahme", "Rolle" und "Notiz" unberührt. Umgebungen und Solutions, die es nicht mehr gibt, bleiben mit "Vorhanden = Nein" stehen. Bei Umgebungen ohne Zugriff bleiben die Solution-Zeilen unverändert, weil dort nicht bekannt ist, was in ihnen liegt. Bei geänderten Listennamen die Parameter `environmentsListName` und `solutionsListName` setzen.
5. `setup-lists.yml`: erst als Trockenlauf, dann mit `dryRun` = aus die Listen anlegen. Pro Liste gibt es ein Häkchen (Environments, Solutions). Der Lauf ist wiederholbar und ergänzt nur fehlende Listen und Spalten. Kommt später eine Spalte dazu (z. B. "Art" in der Liste Solutions), `setup-lists.yml` einfach noch einmal laufen lassen. Jede Spalte hat eine Beschreibung (Listeneinstellungen › Spalte öffnen): wofür sie ist, wer sie schreibt (Pipeline, Mensch, später Backup) und wo sie verwendet wird. Der Lauf gleicht die Beschreibungen bestehender Spalten an, sie werden also zentral in `setup-lists.yml` gepflegt (Parameter `-Desc` der Spalte). Von Hand geänderte Beschreibungen werden dabei überschrieben.

Details zu Parametern, Voraussetzungen und Grenzen stehen im Kopf von `pipelines/inventory.yml`.

## Die Listen im Alltag

**Solutions** hat eine Zeile pro Solution und Umgebung: eigene (unmanaged), managed Solutions, die nicht von Microsoft stammen, und die automatisch angelegte "Common Data Services Default Solution" (Art "Standard"). Nicht aufgenommen werden managed Solutions von Microsoft (Publisher-Prefix `msdyn`, einstellbar über den Parameter `ignorePrefixes`, dazu Publisher-Namen, die mit "Microsoft" oder "Dynamics 365" beginnen) und die System-Solutions (Standardlösung, Aktive Lösung, Basislösung). Sie stehen weiterhin im Pipeline-Log und in `inventory.json`. Zeilen von früheren Läufen, die jetzt ausgeblendet sind, fasst die Pipeline nicht mehr an, sie lassen sich in der Liste löschen.

- **Inhalt** zeigt je Typ die Objekte der Solution, zum Beispiel `Flows: Flow1, Flow2`, `Canvas Apps: …`, `Umgebungsvariablen: …`, `Tabellen: …`, dazu `Sonstige` als Anzahl. Höchstens 25 Namen je Typ. **Inhalt (JSON)** enthält dieselben Daten vollständig für die Power App (`ParseJSON`). Beide Spalten schreibt die Pipeline bei jedem Lauf neu, eigene Anmerkungen gehören in "Notiz". Bei der Standard-Solution sieht man, was Leute ohne eigene Solution gebaut haben.
- **Quelle** (nur bei managed Solutions): Die Pipeline sucht in allen lesbaren Umgebungen eine eigene, unmanaged Solution mit gleichem eindeutigen Namen und gleichem Publisher-Prefix. "Gefunden" heißt, die Solution stammt aus dem eigenen Tenant, in "Quelle in" stehen Umgebung und Version. Liegt die Quelle in einer Umgebung mit Handlungsbedarf (zum Beispiel einer unmanaged Developer-Umgebung), steht das dabei. "Keine gefunden" heißt, die Solution wurde von außen importiert: Für eine Neu-Installation braucht man den Export der managed Solution. "Nicht prüfbar" heißt, es gibt Umgebungen ohne Zugriff, in denen die Quelle liegen könnte. Gleicher Name und Prefix sind ein starkes Indiz, aber kein Beweis.
- **Sicherungsentscheidung** (Offen, Sichern, Nicht sichern) setzt die Pipeline einmalig auf "Offen", mit einem Hinweis, bei eigenen Solutions und bei managed Solutions ohne gefundene Quelle. Hat eine managed Solution eine Quelle, wird die Quelle gesichert und die managed Kopie nicht.

**Environments** hat eine Zeile pro Umgebung. Die Spalte "Dataverse" zeigt, ob die Umgebung Dataverse hat. Nur dann kann die Pipeline Solutions, Apps und Flows lesen. Umgebungen ohne Dataverse erscheinen mit Typ, Owner und Hinweis, "Eigene Solutions" und "Lose Objekte" bleiben leer (nicht 0), und das Zugriff-Log sagt, dass dort nichts erfasst wird. "Lose Objekte (Namen)" nennt die Flows, Apps und Tabellen, die in keiner echten Solution liegen (die Standard-Solution und die System-Solutions zählen dabei nicht als Solution). Apps, die in keiner Solution liegen, stehen nicht in Dataverse; die Pipeline holt sie zusätzlich über die Admin API von Power Apps. Klappt das nicht (fehlende Berechtigung), gibt es eine Warnung im Log und die Zahl der losen Apps ist zu niedrig.

- **Zugriffsentscheidung:** Fehlt der App der Zugriff auf eine Umgebung, setzt die Pipeline die Entscheidung auf "Offen" und trägt eine Maßnahme ein. Bei "Freigeben" legt der nächste Lauf den Application User an (Rolle System Administrator, über die Admin API von Power Platform) und setzt die Entscheidung danach selbst auf "Freigegeben". Geht der Zugriff später verloren, steht die Entscheidung wieder auf "Offen". Bei "Nicht freigeben" bleibt die Umgebung unberührt, und es wird nicht erneut nachgefragt. Der Parameter `grantAccess` entfällt. Nach dem Update `setup-lists.yml` einmal laufen lassen, damit der Wert "Freigegeben" in der Spalte ergänzt wird.
- Die früheren Spalten "Zugriff freigeben" und "Sichern" werden nicht mehr beschrieben. In bestehenden Listen kann man sie löschen.

## Voraussetzung: Site-Berechtigung (`manage`)

Damit `setup-lists.yml` Listen anlegen kann, braucht die App auf der SharePoint-Site die Rolle **`manage`**. Mit `write` reicht es nicht: Das Lesen klappt, das Anlegen einer Liste endet mit `403 accessDenied`. Zum reinen Schreiben von Einträgen genügt später `write`, die Rolle kann nach dem Anlegen der Listen wieder zurückgesetzt werden.

Für Site-Berechtigungen kennt Graph die Rollen `read`, `write`, `manage` und `fullcontrol`. Die Rolle `owner` wird auf der Site nicht angenommen, der Eintrag steht danach auf `none` und die App hat gar keinen Zugriff mehr.

Die Berechtigung setzt ein Admin im [Graph Explorer](https://developer.microsoft.com/graph/graph-explorer) (angemeldet, mit Zustimmung zu `Sites.FullControl.All`):

1. Vorhandene Einträge ansehen: `GET https://graph.microsoft.com/v1.0/sites/{SiteId}/permissions`
2. Neu setzen: `POST https://graph.microsoft.com/v1.0/sites/{SiteId}/permissions` mit dem Body

   ```json
   {
     "roles": ["manage"],
     "grantedToIdentities": [
       { "application": { "id": "{ClientId}", "displayName": "{App-Name}" } }
     ]
   }
   ```

   Die Antwort muss `"roles": ["manage"]` enthalten. Gibt es schon einen Eintrag für die App, ändert ein `PATCH` auf `…/permissions/{permission-id}` mit `{ "roles": ["manage"] }` die Rolle.

Änderungen an Berechtigungen können einige Minuten brauchen, bis sie greifen.

## Sicherheit und Berechtigungen

Die App Registration bekommt weitreichende Rechte. Bitte vor dem Einsatz lesen:

- Als Management-App darf die App alle Umgebungen des Tenants verwalten. `inventory.yml` liest nur, mit einer Ausnahme: dem Anlegen des Application Users (nächster Punkt).
- Bei der Zugriffsentscheidung "Freigeben" legt die Pipeline in der Umgebung einen Application User mit der Rolle **System Administrator** an. Das ändert die Umgebung und geschieht nur auf die ausdrückliche Wahl eines Menschen in der Liste. Wer die Liste bearbeiten darf, kann also Zugriff freigeben: die Berechtigungen auf der Liste entsprechend eng halten. Wer das nicht will, wählt "Nicht freigeben".
- `ClientSecret` gehört als Secret in die Variable Group und nie ins Repo. Secrets laufen ab: Ablaufdatum notieren und rechtzeitig erneuern.
- `User.Read.All` (Application) erlaubt der App, Benutzer des Tenants zu lesen. Es wird nur für den Owner-Namen gebraucht. Ohne die Berechtigung steht der Ersteller in der Spalte.
- Die Site-Rolle `manage` nach dem Anlegen der Listen auf `write` zurücksetzen.

## Stand und Hinweise

- Bisher in einem Tenant getestet. Bitte zuerst in einem Test-Tenant ausprobieren, `setup-lists.yml` zuerst als Trockenlauf.
- Der Risikohinweis für unmanaged Developer-Umgebungen nutzt die von Microsoft angekündigten Termine (Sperre ab 15.12.2026, Löschung ab 15.01.2027, Stand November 2026, [Quelle](https://learn.microsoft.com/power-platform/important-changes-coming#deprecation-of-unmanaged-developer-environments-and-the-power-apps-developer-plan)). Sie stehen fest in `inventory.yml` und sind bei Änderungen dort anzupassen.
- Die Pipeline liest Felder der Power Platform Admin API (`environmentSku`, `governanceConfiguration.protectionLevel`, `createdBy`). Zeigt eine Umgebung "unbekannt", hilft der Parameter `dumpRaw`: Er schreibt die Rohantwort der ersten Umgebung ins Log.
- Die Dateien sind auf Deutsch.

## Lizenz

[MIT](LICENSE) © 2026 Daniela Zurwerra. Bereitstellung ohne Gewähr, bitte zuerst in einem Test-Tenant ausprobieren.
