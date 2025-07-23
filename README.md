
# Anleitung: Chatbot-Daten für Kodular nutzen

Diese Anleitung beschreibt, wie du die Wissensdatenbank (Regeln, Maps, Sets) dieser Web-Anwendung hostest, um sie dynamisch in deiner Kodular-App zu laden. Es werden zwei Strategien vorgestellt:

1.  **Live-Abruf von GitHub:** Einfach umzusetzen, erfordert aber bei jedem Start eine Internetverbindung.
2.  **Lokales Speichern für Offline-Nutzung:** Etwas aufwendiger, aber leistungsfähiger, da die App nach einem einmaligen Download offline funktioniert.

---

### Schritt 1: GitHub-Repository erstellen und Dateien hochladen

1.  Erstelle ein neues, **öffentliches** Repository auf [GitHub](https://github.com/).
2.  Lade **alle** Dateien aus dem `public`-Verzeichnis dieser Web-App in dein neues Repository hoch. Dazu gehören:
    - `knowledge.json` (oder deine anderen Regeldateien wie `rules.json`)
    - `bot.config.json`
    - `knowledge_index.json`
    - `normalization.json`
    - Alle `.map`-Dateien (z.B. `capitals.map`, `flags.map`)
    - Alle `.set`-Dateien (z.B. `staaten.set`)

---

### Schritt 2: Raw-URLs der Dateien ermitteln

Für den Zugriff über Kodular benötigst du die "rohe" URL zu jeder Datei, nicht die normale GitHub-Link.

1.  Navigiere in deinem GitHub-Repository zu einer deiner hochgeladenen Dateien (z.B. `bot.config.json`).
2.  Klicke auf den **"Raw"**-Button.
3.  Kopiere die URL aus der Adresszeile deines Browsers.

Die URL wird folgendes Format haben:
`https://raw.githubusercontent.com/DEIN_BENUTZERNAME/DEIN_REPO_NAME/main/bot.config.json`

Wiederhole diesen Vorgang, um die Raw-URL für **jede** deiner Wissensdateien zu erhalten.

---

### Schritt 3: Konfigurationsdateien anpassen

Jetzt musst du die lokalen Pfade in deinen Konfigurationsdateien durch die neuen GitHub-Raw-URLs ersetzen.

1.  **`bot.config.json` anpassen:**
    Ersetze den lokalen Pfad bei `rulesFile` durch die Raw-URL deiner Regeldatei.

    **Vorher:**
    ```json
    {
      "rulesFile": "./public/knowledge.json",
      ...
    }
    ```

    **Nachher (Beispiel):**
    ```json
    {
      "rulesFile": "https://raw.githubusercontent.com/DEIN_BENUTZERNAME/DEIN_REPO_NAME/main/knowledge.json",
      ...
    }
    ```

2.  **`knowledge_index.json` anpassen:**
    Ersetze alle lokalen Pfade für Maps und Sets durch die entsprechenden Raw-URLs.

    **Vorher:**
    ```json
    {
      "maps": [ { "name": "capitals", "path": "./public/capitals.map" } ],
      ...
    }
    ```

    **Nachher (Beispiel):**
    ```json
    {
      "maps": [ { "name": "capitals", "path": "https://raw.githubusercontent.com/DEIN_BENUTZERNAME/DEIN_REPO_NAME/main/capitals.map" } ],
      ...
    }
    ```

---

### Schritt 4: Angepasste Konfigurationen hochladen

Lade die beiden soeben geänderten Dateien (`bot.config.json` und `knowledge_index.json`) ebenfalls in dein GitHub-Repository hoch und überschreibe die alten Versionen.

---

### Schritt 5: Umsetzung in Kodular

Du hast nun zwei Möglichkeiten, die Daten in Kodular zu laden.

#### Strategie A: Live-Abruf (Einfach, nur online)

Diese Methode ist einfacher, aber die App muss bei jedem Start alle Daten neu aus dem Internet laden.

1.  **Lade `bot.config.json`:** Wenn die App startet (`Screen.Initialize`), rufe mit der `Web`-Komponente die Raw-URL deiner `bot.config.json` ab.
2.  **Verarbeite Konfiguration:** Im `Web.GotText`-Block, parse die Antwort. Speichere die Bot-Eigenschaften und die URL der Regeldatei in globalen Variablen.
3.  **Lade Regeln & Index:** Rufe nun die Regeldatei und die `knowledge_index.json` mit der `Web`-Komponente ab.
4.  **Lade Maps & Sets:** Parse den Index, den du im vorherigen Schritt geladen hast. Loope durch die Listen der Maps und Sets und lade jede einzelne Datei über ihre URL.
5.  **Nachdem alles geladen ist,** kann dein Chatbot arbeiten.

#### Strategie B: Lokales Speichern (Fortgeschritten, offline-fähig)

Diese Methode ist robuster und performanter. Die Daten werden von GitHub heruntergeladen und im App-spezifischen Verzeichnis (ASD) gespeichert.

**Benötigte Kodular-Komponenten:**
-   `Web`-Komponente
-   `File`-Komponente
-   `Notifier`-Komponente (für Lade-Feedback)

**Logik-Überblick:**

1.  **App-Start (`Screen.Initialize`):**
    -   Prüfe mit der `File`-Komponente, ob eine Kerndatei, z.B. `bot.config.json`, bereits lokal existiert.
    -   **Wenn ja:** Super, die App kann die Daten lokal laden. Erstelle eine Prozedur `LadeLokaleDaten()`.
    -   **Wenn nein (erster Start):** Zeige dem Benutzer eine Nachricht (z.B. mit dem `Notifier`), dass eine Ersteinrichtung (Download) erforderlich ist. Mache deinen "Update"-Button sichtbar.

2.  **Die "Update"-Button Logik (Prozedur `StarteUpdate`):**
    -   **Schritt 2.1: Indexe laden:** Rufe zuerst die `bot.config.json` und `knowledge_index.json` von ihren GitHub-Raw-URLs mit der `Web`-Komponente ab.
    -   **Schritt 2.2: Dateien speichern:** Im `Web.GotText`-Event, nutze `File.SaveFile`, um die heruntergeladenen Inhalte zu speichern.
        -   `fileName`: `//bot.config.json` (der doppelte Schrägstrich stellt sicher, dass es im ASD gespeichert wird)
        -   `text`: `responseContent`
    -   **Schritt 2.3: Dateiliste erstellen:** Nachdem die beiden Index-Dateien geladen und gespeichert sind, parse sie, um eine **komplette Liste aller benötigten Datei-URLs** zu erstellen (Regeldatei + alle Maps + alle Sets).
    -   **Schritt 2.4: Alle Dateien herunterladen:** Loope durch diese Liste. In jeder Iteration:
        -   Rufe die URL mit der `Web`-Komponente ab.
        -   Im `Web.GotText`-Event, speichere die Datei wieder mit `File.SaveFile` und einem passenden Dateinamen (z.B. `//knowledge.json`, `//capitals.map`).
        -   Zeige dem Benutzer den Fortschritt mit dem `Notifier`.
    -   **Schritt 2.5: Abschluss:** Wenn alle Dateien heruntergeladen sind, zeige eine "Update erfolgreich"-Nachricht und rufe die `LadeLokaleDaten()` Prozedur auf.

3.  **Die `LadeLokaleDaten()` Prozedur:**
    -   Diese Prozedur ist ähnlich wie die Live-Strategie, aber statt der `Web`-Komponente nutzt du die `File.ReadFromFile`-Blöcke, um die Inhalte aus dem lokalen Speicher zu lesen und in deinen globalen Variablen (Dictionaries für Maps, Listen für Sets etc.) zu speichern.

**Wichtig:** Der restliche Code deines Chatbots (Pattern-Matching, Tag-Verarbeitung etc.) bleibt fast identisch. Der einzige Unterschied ist die **Datenquelle**: Statt auf eine `Web.GotText`-Antwort zu warten, liest du die Daten aus einer lokalen Datei. Der Code in `index.tsx` dient dir weiterhin als perfekte Vorlage für die Logik.
