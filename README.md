# K8s-for-Kubeflow

Installationsanleitung für Kubernetes und Kubeflow auf einem einzelnen Linux-Rechner im Labor (kein verteilter Cluster). Am Ende läuft eine Kubeflow-Pipeline mit einem GPU-Schritt.

Diese Anleitung legt keine feste Version fest. Sie zeigt, wie man prüft, welche Kubernetes-Version zu welcher Kubeflow-Version passt. Die Versionsangaben in den Beispielen sind eine Momentaufnahme (Stand 02.11.2026) und müssen vor jeder neuen Installation erneut geprüft werden.

## Inhalt

1. [Voraussetzungen und Kompatibilität prüfen](#1-voraussetzungen-und-kompatibilität-prüfen)
2. Kubernetes mit kubeadm (folgt)
3. GPU-Check und NVIDIA Device Plugin (folgt)
4. Speicher: Local Path Provisioner als Default StorageClass (folgt)
5. Kubeflow installieren (folgt)
6. Zugriff auf das Dashboard (folgt)
7. Erster Test: Pipeline mit GPU-Schritt (folgt)

## Aufbau jedes Befehls

Zu jedem Befehl stehen vier Angaben:

- **Was:** was der Befehl tut und was die Optionen bedeuten.
- **Wo:** auf welchem Rechner bzw. in welchem Terminal er ausgeführt wird.
- **Erwartet:** welches Ergebnis man sehen sollte.
- **Prüfen:** woran man erkennt, dass der Schritt geklappt hat.

## 1. Voraussetzungen und Kompatibilität prüfen

Bevor etwas installiert wird, klären wir zwei Fragen: Welche Kubernetes-Version passt zur gewünschten Kubeflow-Version? Und erfüllt der Rechner die Mindestanforderungen?

### 1.1 Passende Versionen bestimmen

Kubeflow wird als Sammlung von Manifesten veröffentlicht (Kubeflow Community Distribution). Jede Release-Seite nennt die Kubernetes-Version, die das Kubeflow-Team geprüft hat. Kubernetes wiederum pflegt immer nur die drei neuesten Minor-Versionen. Gesucht ist eine Kubernetes-Version, die beide Bedingungen erfüllt.

**Schritt 1: Kubeflow-Release wählen und die geprüfte Kubernetes-Version ablesen.**

- Übersicht aller Releases: https://www.kubeflow.org/docs/releases/
- Auf der Seite des neuesten Releases steht unter *Dependency Versions (Manifests)* die Zeile *Kubernetes*. Für 26.03 steht dort `1.34+`, mit dem Hinweis, dass andere Versionen funktionieren können, aber nicht validiert sind: https://www.kubeflow.org/docs/releases/kubeflow-26.03/
- Patch-Releases (z. B. 26.03.1) stehen in den Release Notes auf GitHub. Dort steht auch, gegen welche Kubernetes-Version die automatischen Tests (CI) laufen. Für 26.03.1: *Kubernetes 1.36 CI tests*. https://github.com/kubeflow/community-distribution/releases

**Schritt 2: Prüfen, ob die Kubernetes-Version noch gepflegt wird.**

- Die Version Skew Policy nennt die drei Minor-Versionen, die noch Patches erhalten: https://kubernetes.io/releases/version-skew-policy/
- Eine Version außerhalb dieser drei bekommt keine Sicherheitsupdates mehr und ist für eine neue Installation ungeeignet.

**Schritt 3: Schnittmenge bilden.**

Man nimmt die neueste Kubernetes-Version, die in Schritt 1 geprüft bzw. getestet ist und in Schritt 2 noch gepflegt wird. Eine neuere Kubernetes-Version, gegen die Kubeflow noch nicht getestet wurde, ist ein Risiko.

Ergebnis am 02.11.2026 (Beispiel, nicht übernehmen ohne neue Prüfung):

| Frage | Quelle | Ergebnis |
|---|---|---|
| Neuestes Kubeflow-Release | Kubeflow-Releaseseite, GitHub Releases | 26.03.1 |
| Kubernetes, vom Kubeflow-Team validiert bzw. getestet | Release-Seite 26.03, Release Notes 26.03.1 | 1.34+ validiert, CI-Tests auf 1.36 |
| Gepflegte Kubernetes-Versionen | Version Skew Policy | 1.37, 1.36, 1.35 |
| Gewählt | Schnittmenge | Kubernetes 1.36 mit Kubeflow 26.03.1 |

Die Release-Seite nennt außerdem die Kustomize-Version (für 26.03: `5.8.1`). Sie wird in Abschnitt 5 gebraucht.

### 1.2 Den Rechner prüfen

Alle Befehle in diesem Abschnitt ändern nichts am System, sie lesen nur Informationen aus.

Mindestanforderungen laut Kubernetes für kubeadm: 2 CPUs und 2 GB RAM pro Rechner sowie ein eindeutiger Hostname (https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/). Für Kubeflow empfiehlt die Community Distribution 16 GB RAM und 8 CPU-Kerne (https://github.com/kubeflow/community-distribution).

#### Betriebssystem

```bash
lsb_release -a
```

- **Was:** zeigt Name und Version der Linux-Distribution. `-a` gibt alle Felder aus (Distributor, Beschreibung, Release, Codename).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Ubuntu-Version mit Langzeitunterstützung (LTS), z. B. `Description: Ubuntu 24.04.x LTS`.
- **Prüfen:** Die Zeile `Description` enthält `LTS`.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

#### CPU-Kerne

```bash
nproc
```

- **Was:** gibt die Anzahl der verfügbaren CPU-Kerne als Zahl aus.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zahl, mindestens `2` für kubeadm, besser `8` oder mehr für Kubeflow.
- **Prüfen:** Die Zahl ist mindestens 8.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

#### Arbeitsspeicher

```bash
free -h
```

- **Was:** zeigt belegten und freien Arbeitsspeicher sowie Swap. `-h` gibt die Werte lesbar in GiB/MiB aus.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** in der Zeile `Mem:` unter `total` mindestens 16 GiB.
- **Prüfen:** `total` in der Zeile `Mem:` ist 16 GiB oder mehr. Die Zeile `Swap:` wird im nächsten Befehl genauer geprüft.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

#### Swap

```bash
swapon --show
```

- **Was:** listet alle aktiven Swap-Bereiche auf. Ohne aktiven Swap gibt der Befehl nichts aus.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe, oder eine Liste mit Swap-Dateien bzw. -Partitionen.
- **Prüfen:** Ist die Ausgabe leer, ist Swap aus. Steht dort ein Eintrag, muss Swap vor der Installation abgeschaltet werden, solange der kubelet nicht ausdrücklich für Swap konfiguriert ist (Installing kubeadm, Abschnitt *Before you begin*). Das geschieht in Abschnitt 2.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

#### Freier Speicherplatz

```bash
df -h /
```

- **Was:** zeigt Größe, Belegung und freien Platz des Dateisystems, auf dem `/` liegt. `-h` gibt die Werte lesbar aus. Dort landen Container-Images und die Daten der Kubeflow-Komponenten.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile mit den Spalten `Size`, `Used`, `Avail`.
- **Prüfen:** `Avail` ist deutlich größer als 100 GB. Die Komponententabelle der Community Distribution nennt zusammen rund 65 GB Speicher für persistente Volumes, dazu kommen die Container-Images und eigene Datensätze. Die 100 GB sind eine eigene Faustregel, kein offizieller Wert.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

#### Hostname

```bash
hostname
```

- **Was:** gibt den Namen des Rechners aus. kubeadm verwendet ihn als Namen des Kubernetes-Knotens.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** ein Name wie `ruki-ws-03`.
- **Prüfen:** Der Name enthält nur Kleinbuchstaben, Ziffern, `-` oder `.`. Kubernetes verlangt das für die meisten Objektnamen (https://kubernetes.io/docs/concepts/overview/working-with-objects/names/). Ein Name mit Großbuchstaben muss vorher geändert werden: Auf `RuKI-ws-07` brach `kubeadm init` deshalb ab (eigene Erfahrung im Labor, März 2026).

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

#### Versionen von kubeadm, kubelet und kubectl

kubeadm installiert weder kubelet noch kubectl. Alle drei müssen zur gewählten Kubernetes-Version aus 1.1 passen (hier 1.36).

```bash
kubeadm version -o short
```

- **Was:** zeigt die Version von kubeadm. `-o short` gibt nur die Versionsnummer aus.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Version der Form `v1.36.x`.
- **Prüfen:** Die Minor-Version (die mittlere Zahl) entspricht der in 1.1 gewählten Version.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

```bash
kubelet --version
```

- **Was:** zeigt die Version des kubelet, des Dienstes, der auf dem Knoten die Container startet.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `Kubernetes v1.36.x`.
- **Prüfen:** Gleiche Minor-Version wie bei kubeadm. Fehlt der Befehl, ist der kubelet nicht installiert und wird in Abschnitt 2 nachinstalliert.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

```bash
kubectl version --client
```

- **Was:** zeigt die Version von kubectl, dem Kommandozeilenwerkzeug für den Cluster. `--client` fragt nur das lokale Programm ab, weil noch kein Cluster läuft.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile `Client Version: v1.36.x`.
- **Prüfen:** kubectl darf laut Version Skew Policy höchstens eine Minor-Version neuer oder älter sein als der Cluster, also hier 1.35, 1.36 oder 1.37.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

#### Checkliste

| Prüfpunkt | Anforderung |
|---|---|
| Betriebssystem | Ubuntu LTS |
| CPU-Kerne | mindestens 8 empfohlen |
| Arbeitsspeicher | mindestens 16 GiB empfohlen |
| Swap | aus (sonst in Abschnitt 2 abschalten) |
| Freier Speicher | deutlich mehr als 100 GB |
| Hostname | nur Kleinbuchstaben, Ziffern, `-`, `.` |
| kubeadm, kubelet, kubectl | passend zur Kubernetes-Version aus 1.1 |

Erst wenn alle Punkte erfüllt sind, geht es mit Abschnitt 2 weiter.

## Quellen

- Kubeflow Releases: https://www.kubeflow.org/docs/releases/
- Kubeflow Community Distribution 26.03: https://www.kubeflow.org/docs/releases/kubeflow-26.03/
- Release Notes der Community Distribution: https://github.com/kubeflow/community-distribution/releases
- Kubeflow Community Distribution (Repository): https://github.com/kubeflow/community-distribution
- Kubernetes Version Skew Policy: https://kubernetes.io/releases/version-skew-policy/
- Installing kubeadm (Kubernetes v1.36): https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Object Names and IDs: https://kubernetes.io/docs/concepts/overview/working-with-objects/names/

Alle Links geprüft am 02.11.2026.
