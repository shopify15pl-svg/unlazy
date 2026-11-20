---
name: leadliste
description: Baut aus öffentlichen Webquellen eine Firmenliste mit Kontaktdaten und schreibt sie als CSV. Verwenden wenn jemand eine Leadliste, Firmenliste, Prospect-Liste oder Kontaktrecherche braucht, ein Branchenverzeichnis auslesen will oder fragt, wie Claude selbst Leads findet statt Listen zu kaufen.
---

# Leadliste

Firmen aus öffentlichen Quellen sammeln, Kontaktdaten übernehmen und als CSV
ablegen. Grundlage ist die Open-Source-Bibliothek ScrapeGraphAI (MIT-Lizenz).
Nur öffentlich sichtbare Angaben, keine Anmeldung, keine Umgehung von Sperren.

## Einrichtung

Beim ersten Lauf prüfen, ob die Bibliothek vorhanden ist. Wenn nicht:

```bash
pip install scrapegraphai
playwright install chromium
```

Danach klären, welches Modell arbeitet. Zwei Wege, beide funktionieren:

```python
# Weg 1: OpenAI, schnell, kostet pro Lauf ein paar Cent
graph_config = {
    "llm": {"api_key": "DEIN_OPENAI_KEY", "model": "openai/gpt-4o-mini"},
    "verbose": False,
    "headless": True,
}

# Weg 2: lokal über Ollama, kostet nichts, braucht ein installiertes Modell
graph_config = {
    "llm": {"model": "ollama/llama3.2", "model_tokens": 8192, "format": "json"},
    "verbose": False,
    "headless": True,
}
```

Den Schlüssel niemals in die Ausgabe schreiben und niemals in eine Datei
committen. Er gehört in eine Umgebungsvariable.

Ist die Quellseite gleichförmig aufgebaut (Verzeichnis mit identischen
Einträgen), braucht es kein Modell: die Seite direkt abrufen und die Felder
per Parser übernehmen. Das ist schneller, kostet nichts und kann nichts
erfinden.

## Ablauf

1. Den Auftrag in vier Angaben zerlegen: **Branche**, **Region**, **wie viele
   Firmen**, **welche Felder**. Fehlt eine Angabe und lässt sie sich nicht aus
   dem Auftrag ableiten, genau danach fragen. Sonst nicht nachfragen.
2. Die Quelle wählen:
   * Konkrete Verzeichnisseite oder Firmenliste bekannt: `SmartScraperGraph`
     auf diese URL.
   * Nur ein Suchauftrag bekannt: `SearchGraph`, der sucht selbst und liest
     mehrere Treffer.
   * Viele bekannte URLs: `SmartScraperMultiGraph`.
   * Für Handwerk zuerst die Mitgliederverzeichnisse der Innungen und
     Fachverbände prüfen (Betriebssuche, Handwerkersuche). Sie sind öffentlich,
     gepflegt und meist gleichförmig aufgebaut.
3. Extrahieren. Der Prompt an die Bibliothek benennt die Felder einzeln, in
   dieser Reihenfolge: Firmenname, Website, Kontaktseite, E-Mail, Telefon,
   Ort, Kurzbeschreibung.
4. Ergebnis prüfen: Duplikate über die Domain zusammenführen, offensichtlich
   unvollständige Zeilen markieren statt löschen. Stichprobe: jede E-Mail und
   jede Telefonnummer muss wörtlich in der Rohseite vorkommen.
5. Als CSV schreiben, eine Zeile pro Firma, plus zwei Spalten `quelle_url` und
   `geholt_am`. Ohne diese beiden Spalten ist die Liste später nicht prüfbar.
6. Kurz zusammenfassen: wie viele Firmen, wie viele mit E-Mail, wie viele mit
   Telefon, welche Quelle.

```python
from scrapegraphai.graphs import SearchGraph

graph = SearchGraph(
    prompt=(
        "Finde Bauunternehmen im Raum Münster. Gib für jede Firma zurück: "
        "Firmenname, Website, Kontaktseite, E-Mail, Telefon, Ort, "
        "einen Satz Beschreibung. Felder, die auf der Seite nicht stehen, "
        "bleiben leer."
    ),
    config={**graph_config, "max_results": 20},
)
result = graph.run()
```

## Harte Regeln

* **Nichts erfinden.** Das Modell ergänzt fehlende E-Mail-Adressen sonst nach
  Muster (`info@firma.de`). Solche Adressen springen später als unzustellbar
  zurück und beschädigen die Absender-Reputation. Ein Feld, das auf der Seite
  nicht steht, bleibt leer. Kein Raten, kein Ableiten aus dem Firmennamen.
* **Nur öffentlich sichtbare Seiten.** Keine Logins, keine Paywalls, keine
  Captchas, keine Endpunkte, die erkennbar nicht für Abruf gedacht sind.
* **Verschlüsselte Adressen bleiben verschlüsselt.** Verschleiert eine Seite
  E-Mail-Adressen absichtlich (JavaScript-Kodierung, Bild statt Text), ist das
  ein Schutz gegen Einsammeln. Nicht dekodieren, das Feld bleibt leer und die
  Zeile bekommt einen Hinweis.
* **Langsam laufen.** Zwischen zwei Seiten mindestens eine Sekunde warten. Wer
  ein Verzeichnis in dreißig Sekunden durchzieht, fliegt raus und nimmt der
  Quelle die Erreichbarkeit.
* **Vor dem Lauf `robots.txt` der Zieldomain lesen** und ausgeschlossene Pfade
  auslassen.
* Personenbezogene Daten nur, soweit sie als geschäftlicher Kontakt öffentlich
  auf der Firmenseite stehen. Keine privaten Profile, keine Netzwerkprofile,
  keine zusammengeführten Personendossiers.
* Fertige Listen nie in ein Repository committen. Sie enthalten Namen von
  Inhabern und gehören in einen ignorierten Ausgabeordner.

## Was danach erlaubt ist, und was nicht

Die Liste zu bauen ist das eine. Sie anzuschreiben ist etwas anderes.

* In Deutschland ist eine werbliche E-Mail ohne vorherige Einwilligung nach
  § 7 UWG unzulässig, auch an Firmenadressen. Eine gekaufte oder gescrapte
  Liste ist keine Einwilligung.
* Telefonwerbung gegenüber Unternehmen braucht mindestens eine mutmaßliche
  Einwilligung. Gegenüber Verbrauchern ist sie ohne ausdrückliche Einwilligung
  unzulässig.
* Wer Daten nicht bei der betroffenen Person selbst erhebt, muss sie nach
  Art. 14 DSGVO informieren, in der Regel binnen eines Monats.

Diese drei Punkte immer nennen, wenn jemand die Liste für Kaltakquise nutzen
will. Nicht moralisieren, einmal sagen und weiterarbeiten. Die Liste selbst ist
davon unberührt und für Recherche, Marktübersicht, Priorisierung und warme
Ansprache uneingeschränkt nutzbar.

## Zwei Wege, die halten

Wer die Liste anschreiben will, hat trotzdem zwei saubere Wege. Beide nennen,
sobald jemand fragt, wie er die Firmen kontaktiert.

**Weg 1: Anruf zuerst, Mail danach.** § 7 Abs. 2 Nr. 1 UWG verlangt gegenüber
Firmen nur die mutmaßliche Einwilligung, nicht die ausdrückliche. Die liegt
vor, wenn das Angebot fachlich zum Kerngeschäft des Angerufenen passt. Das
BVerwG (29.01.2025, Az. 6 C 3.23) hat klargestellt: eine öffentlich stehende
Telefonnummer allein reicht dafür nicht, es braucht einen konkreten Bezug, der
schon vor dem Anruf bestand. Im Gespräch wird das Okay für die Mail geholt und
dokumentiert: Datum, Uhrzeit, Name der Person, Wortlaut der Zusage. Damit ist
die Mail danach keine Kaltakquise mehr, sondern eine angeforderte Nachricht.
Ohne Dokumentation existiert die Einwilligung im Streitfall nicht, die
Beweislast liegt beim Absender.

**Weg 2: Brief.** § 7 UWG regelt elektronische Post, Telefon, Fax und
automatische Anrufmaschinen. Klassische Briefwerbung fällt nicht darunter.
Rechtsgrundlage ist das berechtigte Interesse nach Art. 6 Abs. 1 lit. f DSGVO,
Erwägungsgrund 47 nennt Direktwerbung ausdrücklich als solches Interesse. Drei
Pflichten bleiben: einen Widerspruch nach Art. 21 DSGVO sofort und dauerhaft
beachten, vor dem Versand gegen die Robinsonliste des DDV abgleichen, und die
Informationspflicht aus Art. 14 DSGVO erfüllen, was im Brief selbst passieren
kann. Wer diesen Weg plant, braucht die Postanschrift als Feld. Sie steht nicht
in der Standard-Feldreihenfolge, also beim Auftrag mit erheben, sonst fehlt
Straße und PLZ in der CSV.

Weg 2 ist der unterschätzte. Er passt besonders zu Firmen, die kaum im Postfach
sitzen, etwa Praxen, Handwerk oder Betriebe ohne eigene Webseite. Kein
Anwaltsersatz: vor einer größeren Kampagne lohnt eine halbe Stunde beim Anwalt.

## Ausgabeformat

```text
Auftrag:     Branche, Region, Zielmenge
Quelle:      welche Seiten oder welche Suche
Gefunden:    echte Anzahl Firmen
Vollständig: X mit E-Mail, Y mit Telefon
Datei:       Pfad zur CSV
```

Danach die ersten fünf Zeilen als Tabelle zeigen, damit die Qualität sofort
sichtbar ist. Bei weniger Treffern als gewünscht die echte Zahl nennen und
sagen, woran es lag. Niemals die Zielmenge behaupten, wenn sie nicht erreicht
wurde.

## Grenzen

* Verzeichnisse mit aggressivem Bot-Schutz liefern leere Ergebnisse. Das ist
  kein Fehler des Skills, sondern die Grenze der Quelle.
* Manche Verzeichnisse liefern Ergebnisse nur nach einem Formular-POST. Dann
  den Abruf mit dem Formularfeld direkt stellen, nicht über Umwege.
* Die Qualität hängt am Modell. Kleine lokale Modelle übersehen Felder
  häufiger als `gpt-4o-mini`.
* Kontaktdaten veralten. Das Feld `geholt_am` ist der einzige Grund, warum
  jemand später weiß, wie alt die Zeile ist.
* Der Skill liefert Firmen mit Kontaktdaten. Er sagt nichts darüber, ob eine
  Firma etwas braucht, kaufen will oder Budget hat.
