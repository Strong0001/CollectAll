# CollectAll

**Verteiltes KI-System zur Edge-basierten Bildsegmentierung und automatisierten Marktwertschätzung per Kamera-Scan.**

## Einleitung

CollectAll ist eine Full-Stack-Anwendung, die On-Device Machine Learning (SAM 2 Tiny) mit einer asynchronen, cloud-basierten Microservice-Architektur kombiniert. Das System segmentiert Gegenstände auf Fotos, schätzt deren Marktwert anhand realer Marktdaten mit trainierten ML-Modellen und speichert sie in einer lokalen Sammlung.

> 🚧 **Projekt-Status:** Der Quellcode dieses Projekts ist aktuell privat. Dieses Repository dient als Architektur-Dokumentation und Portfolio-Showcase für Systemdesign, KI-Integration und CI/CD-Pipelines.

### 🛠 Tech Stack

**KI & Machine Learning**  
![Meta SAM 2](https://img.shields.io/badge/Meta%20SAM2-0668E1?style=flat&logo=meta&logoColor=white)
![ONNX Runtime](https://img.shields.io/badge/ONNX-000000?style=flat&logo=onnx&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Gemini-8E75B2?style=flat&logo=google-gemini&logoColor=white)
![Microsoft Foundry](https://img.shields.io/badge/Microsoft%20Foundry-0078D4?style=flat&logo=microsoft-azure&logoColor=white)
![XGBoost](https://img.shields.io/badge/XGBoost-1D2B44?style=flat&logo=xgboost&logoColor=white)

**Backend**  
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-05998B?style=flat&logo=fastapi&logoColor=white)
![Azure PostgreSQL](https://img.shields.io/badge/Azure%20PostgreSQL-316192?style=flat&logo=postgresql&logoColor=white)
![JWT](https://img.shields.io/badge/JWT-000000?style=flat&logo=jsonwebtokens&logoColor=white)

**Cloud & Infrastruktur**  
![Microsoft Azure](https://img.shields.io/badge/Microsoft%20Azure-0089D6?style=flat&logo=microsoft-azure&logoColor=white)
![Azure Container Apps](https://img.shields.io/badge/Container%20Apps-0078D4?style=flat&logo=microsoft-azure&logoColor=white)
![Azure Container Apps Jobs](https://img.shields.io/badge/Container%20Apps%20Jobs-0078D4?style=flat&logo=microsoft-azure&logoColor=white)
![Azure Machine Learning](https://img.shields.io/badge/Azure%20ML-0078D4?style=flat&logo=microsoft-azure&logoColor=white)
![Azure Queue Storage](https://img.shields.io/badge/Azure%20Queue-0078D4?style=flat&logo=microsoft-azure&logoColor=white)
![Azure Blob Storage](https://img.shields.io/badge/Blob%20Storage-0078D4?style=flat&logo=microsoft-azure&logoColor=white)
![Azure Redis Cache](https://img.shields.io/badge/Azure%20Redis-DC382D?style=flat&logo=redis&logoColor=white)

**DevOps & CI/CD**  
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=flat&logo=githubactions&logoColor=white)

**Frontend**  
![.NET MAUI](https://img.shields.io/badge/.NET%20MAUI-512BD4?style=flat&logo=.net&logoColor=white)
![C#](https://img.shields.io/badge/C%23-239120?style=flat&logo=csharp&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-07405E?style=flat&logo=sqlite&logoColor=white)

---

## App Demo
https://github.com/user-attachments/assets/91729379-4464-4555-9b41-2fad0eb0a9d6
> Im Video sind für diesen User-Account ausschließlich die Kategorien *Lego Sets* und *Videospiele* aktiv.

---

## Systemarchitektur & Datenfluss

Das System basiert auf einer strikten Trennung zwischen mobilem Client, Backend-Services und asynchronen Workern, um Skalierbarkeit und Ausfallsicherheit zu gewährleisten.

<p align="center">
  <img src="assets/architecture-diagramm.jpg"><br>
  <sub>High-Level Architektur: Client, Cloud Compute, Data Layer & Pipelines</sub>
</p>

---

### Architektur & Design-Entscheidungen

Dieses Projekt wurde mit Fokus auf Kostenoptimierung, Latenzreduzierung und Ausfallsicherheit entworfen:

* **Edge-Compute & Asynchrone Parallelisierung (SAM 2 via ONNX):**<br>
Die rechenintensive Generierung der Bild-Embeddings für die Segmentierung wurde als ONNX-Modell (Encoder/Decoder) auf das Endgerät des Nutzers ausgelagert. Diese lokale Edge-Inferenz läuft asynchron und parallel zur KI-Analyse im Backend.<br>
**Problem & Lösung:** Um den Scan-Vorgang durch überlappende Ladezeiten zu beschleunigen und nutzerproportionale Compute-Ressourcen (GPUs) im Azure-Backend zu vermeiden, wird die schwere Rechenlast stabil auf die Client-Geräte verteilt.

* **LLM-Skalierung: Context Caching & Semantic Vector Matching:**<br>
Um das System bei einer wachsenden Anzahl an Kategorien kosteneffizient und performant zu halten, nutzt die Architektur eine zweigleisige LLM-Strategie. Der System-Prompt ist strikt segmentiert: Der statische Payload (System-Prompt & Schablonen) triggert ab dem Erreichen der Token-Mindestgröße automatisch das Context Caching der API (Gemini). Bei jedem Scan werden dann nur noch minimale Parameter (aktive Kategorien und das Bild) dynamisch injiziert.<br>
**Problem & Lösung:** Die direkte Übergabe aller in der Datenbank bereits bekannten Gegenstandsnamen (z. B. zehntausende Filmtitel oder Lego-Sets) im Prompt würde das Context Window sprengen und hohe Latenzen verursachen. Auf den Versand dieser Listen an die KI wird daher verzichtet. Die Schablonen weisen die KI stattdessen an, eine freie Namensvorhersage zu generieren. Das Backend vektorisiert diesen Text-Output zur Laufzeit über ein lokal ausgeführtes Sentence-Transformer-Modell und nutzt eine Vektor-Suche in Azure Redis, um die KI-Schätzung semantisch und fehlertolerant auf den korrekten Datenbankeintrag zu mappen.

* **Distributed Caching & Concurrency Control (Azure Redis):**<br>
Um Latenzen zu minimieren und denselben Datenstand über mehrere zustandslose Backend-Replikate hinweg zu garantieren, werden Systemdaten in über 10 logisch getrennten Cache-Bereichen in Azure Redis vorgehalten.<br>
**Problem & Lösung:** Bei gleichzeitig eintreffenden Scans unbekannter Gegenstände durch verschiedene Nutzer drohen Race Conditions und damit mögliche Duplikate in der Datenbank. Das System nutzt deshalb einen Cache-First Lookup mit transaktionalem Re-Check: Die rechenintensive Embedding-Generierung und der erste Lookup erfolgen lock-free gegen den Redis-Cache. Nur bei einem Cache Miss wird eine kurze PostgreSQL-Transaktion (Pessimistic Locking) geöffnet, ein zweiter Check direkt auf der Datenbank ausgeführt und erst dann persistiert. Dadurch werden konkurrierende Schreibvorgänge kontrolliert, ohne die Performance für bereits bekannte Gegenstände auszubremsen.

* **Deterministische Preisfindung & LLM-Daten-Normalisierung (XGBoost):**<br>
Für die finale Wertschätzung kommen eigens trainierte XGBoost-Modelle zum Einsatz, um deterministische und reproduzierbare Modellergebnisse auf Basis echter Marktdaten zu erhalten, statt sich auf generative LLM-Schätzungen zu stützen.<br>
**Problem & Lösung:** Ein XGBoost-Modell erwartet strikt tabellarische, identisch benannte Features. Um den unstrukturierten Output der Bildanalyse-KI (Gemini) in dieses ML-Format zu überführen, ist das Prompt-Engineering und das nachgelagerte Daten-Mapping darauf optimiert, dass semantisch gleiche Merkmale konsistent benannt und formatiert an die Inferenz-Pipeline übergeben werden (Data Normalization).

* **Hybride Marktdaten-Beschaffung (Tiered Fallback) & Asynchrones ML-Training:**<br>
Um die Kosten iterativer KI-Websuchen zu minimieren, nutzt das System eine gestufte hybride Architektur in autarken Azure Container Apps Jobs: Zunächst fragt der Worker strukturierte Rohdaten in hoher Stückzahl über die eBay API ab. Ein Analyse-Agent ordnet die Ergebnisse exakt den vorgegebenen Merkmalen zu. Nur bei unzureichender API-Ausbeute triggert der Job als Fallback einen autonomen KI-Agenten für eine offene Websuche.<br>
**Problem & Lösung:** Um das Frontend nicht zu blockieren, werden Suchaufträge zunächst in PostgreSQL geschrieben und in eine Azure Queue überführt. Diese triggert asynchron einen Azure Container Apps Job. Die so validierten Preisdaten fließen in ein entkoppeltes Batch-Verfahren, das stündlich vollautomatisiert neue XGBoost-Modelle trainiert und dem System bereitstellt.

* **Dynamic Data Modeling & Zero-Downtime Deployments:**<br>
Das System ermöglicht die vollständige Erstellung und Modifikation neuer Gegenstandskategorien und deren Merkmale zur Laufzeit über ein Admin-Panel. Bei der Freischaltung führt das Backend automatisierte Schema-Änderungen (DDL) aus, legt Tabellenstrukturen an, baut In-Memory-Caches neu auf und aktualisiert das globale App-Schema.<br>
**Problem & Lösung:** Um starre App-Store-Releasezyklen zu umgehen, nutzt das System eine Server-Driven Architecture. Konfigurationsdaten (UI-Sichtbarkeit, zulässige Werte, KI-Analyse-Regeln) werden dynamisch über das App-Schema an die Clients gepusht. Dies erfordert eine präzise Cache-Invalidierung im Backend, ermöglicht es aber, das System agil zu skalieren und neue Kategorien ohne Client-Updates weltweit auszurollen.

### DevOps & CI/CD Pipeline

Die Deployment-Pipeline wird durch einen Merge in den `main`-Branch getriggert und durchläuft eine dreistufige Validierung, um fehlerhafte Deployments zu verhindern:

* **Phase 1: Unit Tests & Image Build (Sandbox):** In einer geschlossenen Build-Stage wird der Code validiert. Um unbemerkt teure API-Aufrufe zu blockieren, sperrt `pytest-socket` sämtliche Netzwerkschnittstellen. Externe Abhängigkeiten werden gemockt (SQLite In-Memory). Bei 100 % Test-Erfolg wird das finale Docker-Image gebaut.
* **Phase 2: Automated Integration Testing (Azure Container Apps Jobs):** Die Pipeline triggert automatisch einen Azure Container Apps Job, der das Image im Zusammenspiel mit einer separaten Test-Datenbank validiert (Datenbank-Migrationen und Service-Kommunikation).
* **Phase 3: Production Deployment (Revision Gating):** Nach erfolgreichem Integrationstest rollt die Pipeline das validierte Image als neue **Azure Container App Revision** aus (Zero-Downtime).

---

## Hauptfunktionen & UX

### 1. Kamera-Scan & Interaktive Erkennung

<p align="center">
  <img src="assets/scan-example.PNG" width="250"><br>
  <sub>Live-Segmentierung via SAM 2</sub>
</p>

Die App erkennt Gegenstände in Echtzeit und legt interaktive SVG-Masken über das Bild. Ein Klick auf die Maske öffnet die Detailansicht mit erkannten Merkmalen und einer ersten Wertschätzung.

### 2. Sammlungsverwaltung & Navigation

<p align="center">
  <img src="assets/category-page.jpg" width="250"><br>
  <sub>Kachel-Ansicht der Sammlung</sub>
</p>

Eine hierarchische Struktur sorgt für Übersichtlichkeit: Globale Übersicht für alle aktiven Kategorien, Kategorie-Ansicht als Kachel-Liste und eine Detail-Ansicht für vollständige Daten (Bild, Name, anpassbare Merkmale) pro Gegenstand.

### 3. Automatisierte Markt-Recherche & Preis-Updates

Die App hält das Portfolio auf dem neuesten Stand. Im Hintergrund sucht das System nach echten, merkmalsbasierten Marktpreisen. Die Wertschätzungen aller gespeicherten Gegenstände werden vollautomatisiert und kontinuierlich aktualisiert.

### 4. Wertentwicklung & Datenvisualisierung

<p align="center">
  <img src="assets/Valuation.jpg" width="250"><br>
  <sub>Wertermittlung Beispiel 7 Tage</sub>
</p>

Nutzer können den historischen Preis-Trend (Portfolio-Graphen) über die gesamte Sammlung hinweg verfolgen oder die Preisentwicklung auf Ebene einzelner Kategorien und spezifischer Gegenstände analysieren.

### 5. Dynamische Kategorien-Wahl & Abonnements

<p align="center">
  <img src="assets/Category_Change.jpg" width="250"><br>
  <sub>Kategorien Übersicht</sub>
</p>

Nutzer können ihr Set an aktiven Kategorien aus einem Pool live geschalteter Sammelgebiete frei zusammenstellen.

### 6. Mehrsprachigkeit (On-the-Fly)

Nahtlose Sprachumschaltung (Deutsch/Englisch) der Benutzeroberfläche und aller dynamischen Inhalte direkt in den Einstellungen, ohne Neustart der App.

### 7. User-Accounts & Data Security

<p align="center">
  <img src="assets/Register.jpg" width="250"><br>
  <sub>Sichere Nutzer-Registrierung</sub>
</p>

Cloud-Synchronisierung und ein sicheres Authentifizierungssystem ermöglichen den nahtlosen Zugriff auf den eigenen Bestand über verschiedene Geräte hinweg.

### 8. Admin-Dashboard & Live-Analytics

<table align="center">
  <tr>
    <td align="center" style="border: none;">
      <img src="assets/scan-statistic.jpg" width="230"><br>
      <sub>Live Scan-Statistik</sub>
    </td>
    <td align="center" style="border: none;">
      <img src="assets/token-statistic.jpg" width="230"><br>
      <sub>API Kosten-Monitoring</sub>
    </td>
  </tr>
</table>

Autorisierte Accounts können neue Kategorien, Merkmale und UI-Labels zur Laufzeit erstellen und freischalten. Das Dashboard bietet zudem Live-Analytics zur Überwachung von Systemdaten wie durchschnittlicher Scan-Dauer, Verarbeitungsvolumen und exakten API-Kosten in Echtzeit.

---
