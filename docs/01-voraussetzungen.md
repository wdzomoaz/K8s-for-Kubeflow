[Übersicht](../README.md)

# 1. Voraussetzungen und Kompatibilität prüfen

Aufbau der Befehle (Was, Wo, Erwartet, Prüfen) und Hinweise zur Sprache der Ausgaben: siehe [Übersicht](../README.md#aufbau-jedes-befehls). Quellen: siehe [Übersicht](../README.md#quellen).

Bevor etwas installiert wird, klären wir zwei Fragen: Welche Kubernetes-Version passt zur gewünschten Kubeflow-Version? Und erfüllt der Rechner die Mindestanforderungen?

## 1.1 Passende Versionen bestimmen

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

Die Release-Seite nennt außerdem die Kustomize-Version (für 26.03: `5.8.1`). Sie wird in Abschnitt 5 gebraucht. kubectl enthält eine eigene Kustomize-Version (https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/); bei kubectl 1.36.6 ist es `v5.8.1` (Ausgabe von `kubectl version --client`, siehe [1.2](#12-den-rechner-prüfen)). Ob `kubectl kustomize` das eigenständige Programm `kustomize` ersetzen kann, wird in Abschnitt 5 geprüft.

## 1.2 Den Rechner prüfen

Alle Befehle in diesem Abschnitt ändern nichts am System, sie lesen nur Informationen aus.

Mindestanforderungen laut Kubernetes für kubeadm: 2 CPUs und 2 GB RAM pro Rechner sowie ein eindeutiger Hostname (https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/). Für Kubeflow empfiehlt die Community Distribution 16 GB RAM und 8 CPU-Kerne (https://github.com/kubeflow/community-distribution).

### Betriebssystem

```bash
lsb_release -a
```

- **Was:** zeigt Name und Version der Linux-Distribution. `-a` gibt alle Felder aus (Distributor, Beschreibung, Release, Codename).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Ubuntu-Version mit Langzeitunterstützung (LTS), z. B. `Description: Ubuntu 24.04.x LTS`.
- **Prüfen:** Die Zeile `Description` enthält `LTS`. Die Zeile `No LSB modules are available.` ist normal und kein Fehler.

Beispiel (ruki-ws-02, 03.11.2026):

```
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 24.04.5 LTS
Release:	24.04
Codename:	noble
```

### CPU-Kerne

```bash
nproc
```

- **Was:** gibt die Anzahl der verfügbaren CPU-Kerne als Zahl aus.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zahl, mindestens `2` für kubeadm, besser `8` oder mehr für Kubeflow.
- **Prüfen:** Die Zahl ist mindestens 8.

Beispiel (ruki-ws-02, 03.11.2026): `32`.

### Arbeitsspeicher

```bash
free -h
```

- **Was:** zeigt belegten und freien Arbeitsspeicher sowie Swap. `-h` gibt die Werte lesbar in GiB/MiB aus.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** in der Zeile `Mem:` (deutsch: `Speicher:`) unter `total` (deutsch: `gesamt`) mindestens 16 GiB.
- **Prüfen:** Der Gesamtwert in der Zeile `Mem:` bzw. `Speicher:` ist 16 GiB oder mehr. Die Zeile `Swap:` (deutsch: `Auslager:`) wird im nächsten Befehl genauer geprüft.

Beispiel (ruki-ws-02, 03.11.2026, deutsche Systemsprache):

```
               gesamt       benutzt     frei      gemns.  Puffer/Cache verfügbar
Speicher:       60Gi       4,6Gi        52Gi        42Mi       3,8Gi        55Gi
Auslager:         0B          0B          0B
```

### Swap

```bash
swapon --show
```

- **Was:** listet alle aktiven Swap-Bereiche auf. Ohne aktiven Swap gibt der Befehl nichts aus.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe, oder eine Liste mit Swap-Dateien bzw. -Partitionen.
- **Prüfen:** Ist die Ausgabe leer, ist Swap aus. Steht dort ein Eintrag, muss Swap vor der Installation abgeschaltet werden, solange der kubelet nicht ausdrücklich für Swap konfiguriert ist (Installing kubeadm, Abschnitt *Before you begin*). Das geschieht in Abschnitt [2](02-kubernetes-kubeadm.md).

Beispiel (ruki-ws-02, 03.11.2026): keine Ausgabe. Auf diesem Rechner war nie Swap eingerichtet (es gibt weder `/swap.img` noch einen Swap-Eintrag in `/etc/fstab`), Abschnitt [2.1](02-kubernetes-kubeadm.md#21-swap-ausschalten) entfällt also.

### Freier Speicherplatz

```bash
df -h /
```

- **Was:** zeigt Größe, Belegung und freien Platz des Dateisystems, auf dem `/` liegt. `-h` gibt die Werte lesbar aus. Dort landen Container-Images und die Daten der Kubeflow-Komponenten.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile mit den Spalten `Size`, `Used`, `Avail` (deutsch: `Größe`, `Benutzt`, `Verf.`).
- **Prüfen:** Der freie Platz (`Avail` bzw. `Verf.`) ist deutlich größer als 100 GB. Die Komponententabelle der Community Distribution nennt zusammen rund 65 GB Speicher für persistente Volumes, dazu kommen die Container-Images und eigene Datensätze. Die 100 GB sind eine eigene Faustregel, kein offizieller Wert.

Beispiel (ruki-ws-02, 03.11.2026, deutsche Systemsprache):

```
Dateisystem    Größe Benutzt Verf. Verw% Eingehängt auf
/dev/nvme1n1p1  1,8T    384G  1,4T   23% /
```

### Hostname

```bash
hostname
```

- **Was:** gibt den Namen des Rechners aus. kubeadm verwendet ihn als Namen des Kubernetes-Knotens.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** ein Name wie `ruki-ws-03`.
- **Prüfen:** Der Name enthält nur Kleinbuchstaben, Ziffern, `-` oder `.`. Kubernetes verlangt das für die meisten Objektnamen (https://kubernetes.io/docs/concepts/overview/working-with-objects/names/). Ein Name mit Großbuchstaben muss vorher geändert werden: Auf `RuKI-ws-07` brach `kubeadm init` deshalb ab (eigene Erfahrung im Labor, März 2026).

Beispiel (ruki-ws-02, 03.11.2026): `ruki-ws-02`.

### IP-Adresse: fest oder per DHCP?

Kubernetes empfiehlt eine stabile Netzwerkkonfiguration des Rechners (Creating a cluster with kubeadm, Abschnitt *Network setup*). Im Labor vergibt der DHCP-Server die Adressen, und eine feste Zuordnung ist nicht immer möglich. Ändert sich die Adresse, bleibt der Cluster stehen. Die Anleitung verringert deshalb in [2.6](02-kubernetes-kubeadm.md#26-cluster-initialisieren) und [2.11](02-kubernetes-kubeadm.md#211-abhängigkeit-von-der-ip-adresse-weiter-verringern), was von der Adresse abhängt, und zeigt in [2.12](02-kubernetes-kubeadm.md#212-reparatur-nach-einem-wechsel-der-ip-adresse) die Reparatur.

```bash
ip route show default
```

- **Was:** zeigt die Standardroute: über welche Netzwerkschnittstelle (`dev`) und welches Gateway (`via`) der Rechner nach außen geht, mit welcher eigenen Adresse (`src`) und woher diese Konfiguration stammt (`proto`).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile der Form `default via <Gateway> dev <Schnittstelle> proto dhcp src <eigene IP> metric 100`.
- **Prüfen:** Steht dort `proto dhcp`, kann sich die Adresse ändern: Die Adresse aus `src` notieren, um später eine Änderung zu erkennen, und die Abschnitte [2.11](02-kubernetes-kubeadm.md#211-abhängigkeit-von-der-ip-adresse-weiter-verringern) und [2.12](02-kubernetes-kubeadm.md#212-reparatur-nach-einem-wechsel-der-ip-adresse) beachten. Steht dort `proto static`, ist die Adresse fest eingestellt.

Beispiel (ruki-ws-02, 03.11.2026):

```
default via 143.93.152.10 dev enp14s0 proto dhcp src 143.93.153.200 metric 100
```

Die Adresse kommt per DHCP. Am 30.09.2026 hatte derselbe Rechner noch `143.93.153.5`: Ein Adresswechsel ist im Labor also kein theoretischer Fall.

### Versionen von kubeadm, kubelet und kubectl

kubeadm installiert weder kubelet noch kubectl. Alle drei müssen zur gewählten Kubernetes-Version aus [1.1](#11-passende-versionen-bestimmen) passen (hier 1.36). Auf einem neuen Rechner sind sie meist noch nicht installiert; das geschieht in 2.5.

```bash
kubeadm version -o short
```

- **Was:** zeigt die Version von kubeadm. `-o short` gibt nur die Versionsnummer aus.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Version der Form `v1.36.x`, oder die Meldung, dass der Befehl nicht gefunden wurde.
- **Prüfen:** Die Minor-Version (die mittlere Zahl) entspricht der in [1.1](#11-passende-versionen-bestimmen) gewählten Version. Meldet das System `Command 'kubeadm' not found` (deutsch: `Der Befehl 'kubeadm' wurde nicht gefunden`), ist kubeadm nicht installiert und wird in [2.5](02-kubernetes-kubeadm.md#25-kubeadm-kubelet-und-kubectl-installieren) installiert. Den dabei angezeigten Vorschlag `sudo snap install kubeadm` **nicht** ausführen: Die offizielle Anleitung installiert über die Paketquelle `pkgs.k8s.io`, und Snap-Pakete werden unabhängig von apt aktualisiert, die Sperre aus [2.5](02-kubernetes-kubeadm.md#25-kubeadm-kubelet-und-kubectl-installieren) (`apt-mark hold`) greift für sie nicht. Zeigt kubeadm eine andere Minor-Version, gehörte der Rechner wahrscheinlich schon zu einem Cluster: dann zuerst 1.3.

Beispiel (ruki-ws-02, vor der Installation, 03.11.2026):

```
Der Befehl 'kubeadm' wurde nicht gefunden, kann aber installiert werden mit:
sudo snap install kubeadm
```

```bash
kubelet --version
```

- **Was:** zeigt die Version des kubelet, des Dienstes, der auf dem Knoten die Container startet.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `Kubernetes v1.36.x`, oder die Meldung, dass der Befehl nicht gefunden wurde.
- **Prüfen:** Gleiche Minor-Version wie bei kubeadm. Fehlt der Befehl, wird der kubelet in [2.5](02-kubernetes-kubeadm.md#25-kubeadm-kubelet-und-kubectl-installieren) installiert (auch hier nicht `snap install` verwenden).

```bash
kubectl version --client
```

- **Was:** zeigt die Version von kubectl, dem Kommandozeilenwerkzeug für den Cluster. `--client` fragt nur das lokale Programm ab, weil noch kein Cluster läuft.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile `Client Version: v1.36.x` und eine Zeile `Kustomize Version: ...`, oder die Meldung, dass der Befehl nicht gefunden wurde.
- **Prüfen:** kubectl darf laut Version Skew Policy höchstens eine Minor-Version neuer oder älter sein als der Cluster, also hier 1.35, 1.36 oder 1.37. Fehlt der Befehl, wird kubectl in [2.5](02-kubernetes-kubeadm.md#25-kubeadm-kubelet-und-kubectl-installieren) installiert (nicht über `snap`).

Beispiel (ruki-ws-02, nach der Installation in [2.5](02-kubernetes-kubeadm.md#25-kubeadm-kubelet-und-kubectl-installieren), 03.11.2026): `kubeadm version -o short` ergab `v1.36.6`, `kubelet --version` ergab `Kubernetes v1.36.6`, und `kubectl version --client` ergab:

```
Client Version: v1.36.6
Kustomize Version: v5.8.1
```

### Checkliste

| Prüfpunkt | Anforderung |
|---|---|
| Betriebssystem | Ubuntu LTS |
| CPU-Kerne | mindestens 8 empfohlen |
| Arbeitsspeicher | mindestens 16 GiB empfohlen |
| Swap | aus (sonst in Abschnitt [2](02-kubernetes-kubeadm.md) abschalten) |
| Freier Speicher | deutlich mehr als 100 GB |
| Hostname | nur Kleinbuchstaben, Ziffern, `-`, `.` |
| IP-Adresse | notiert; bei `proto dhcp` die Abschnitte [2.11](02-kubernetes-kubeadm.md#211-abhängigkeit-von-der-ip-adresse-weiter-verringern) und [2.12](02-kubernetes-kubeadm.md#212-reparatur-nach-einem-wechsel-der-ip-adresse) beachten |
| kubeadm, kubelet, kubectl | passend zur Kubernetes-Version aus [1.1](#11-passende-versionen-bestimmen) **oder noch nicht installiert** (wird in [2.5](02-kubernetes-kubeadm.md#25-kubeadm-kubelet-und-kubectl-installieren) installiert) |
| Früherer Cluster | keine Reste (sonst zuerst [1.3](#13-nur-falls-nötig-reste-eines-früheren-clusters-entfernen)) |

Erst wenn alle Punkte erfüllt sind, geht es mit Abschnitt [2](02-kubernetes-kubeadm.md) weiter.

## 1.3 Nur falls nötig: Reste eines früheren Clusters entfernen

Dieser Abschnitt gilt nur, wenn der Rechner schon einmal Teil eines Kubernetes-Clusters war. Anzeichen dafür: Es gibt den Ordner `/etc/kubernetes`, `apt-mark showhold` listet `kubeadm`, `kubelet` oder `kubectl`, oder kubeadm hat in [1.2](#12-den-rechner-prüfen) eine andere Version gezeigt.

> **Achtung:** Die folgenden Befehle löschen die Kubernetes-Konfiguration des Rechners endgültig. Vor jedem Befehl im Prompt prüfen, auf welchem Rechner man gerade arbeitet (z. B. `ruki-labor@ruki-ws-02:~$`). Ein `kubeadm reset` auf dem falschen Rechner löscht dort etcd, alle Zertifikate und Zugangsdateien. Genau das ist im Labor am 03.11.2026 passiert: Der Cluster auf ruki-ws-07 musste aus einer Sicherung wiederhergestellt werden.

> **Achtung:** Nicht `sudo apt autoremove` ausführen. Auf ruki-ws-02 hätte der Befehl nach dem Entfernen der Kubernetes-Pakete auch Pakete der CUDA-Installation entfernt (eigene Beobachtung, 03.11.2026).

**Schritt 1: Den Knoten aus dem alten Cluster abmelden.** Nur nötig, wenn der alte Cluster noch läuft. Die beiden Befehle laufen **auf dem Control-Plane-Rechner des alten Clusters**, nicht auf dem Rechner, der neu aufgesetzt wird.

```bash
kubectl drain <knotenname> --ignore-daemonsets --delete-emptydir-data
```

- **Was:** sperrt den Knoten für neue Pods und verschiebt die laufenden Pods auf andere Knoten (Safely Drain a Node). `--ignore-daemonsets` lässt Pods von DaemonSets stehen, weil sie ohnehin an den Knoten gebunden sind. `--delete-emptydir-data` erlaubt, Pods mit temporären `emptyDir`-Volumes zu entfernen; deren Daten gehen dabei verloren.
- **Wo:** Terminal des Control-Plane-Rechners des **alten** Clusters. `<knotenname>` durch den Namen aus `kubectl get nodes` ersetzen.
- **Erwartet:** `node/<knotenname> cordoned`, dann Zeilen `evicting pod ...` und zum Schluss `node/<knotenname> drained`.
- **Prüfen:** `kubectl get nodes` zeigt den Knoten mit `SchedulingDisabled`.

```bash
kubectl delete node <knotenname>
```

- **Was:** entfernt den Knoten aus der Knotenliste des alten Clusters.
- **Wo:** Terminal des Control-Plane-Rechners des **alten** Clusters.
- **Erwartet:** `node "<knotenname>" deleted`.
- **Prüfen:** `kubectl get nodes` listet den Knoten nicht mehr.

**Schritt 2: Den Rechner selbst aufräumen.** Alle folgenden Befehle laufen auf dem Rechner, der neu aufgesetzt wird.

```bash
sudo kubeadm reset
```

- **Was:** macht rückgängig, was `kubeadm init` oder `kubeadm join` auf diesem Rechner angelegt hat: stoppt den kubelet, löscht `/etc/kubernetes/manifests`, `/etc/kubernetes/pki`, die Zugangsdateien und bei einem Control-Plane-Knoten die lokale etcd-Datenbank. Nicht entfernt werden die CNI-Konfiguration in `/etc/cni/net.d`, die Netzwerkregeln von kube-proxy und `$HOME/.kube` (kubeadm reset, Kubernetes v1.36). Das erledigen die nächsten Befehle.
- **Wo:** Terminal des Rechners, der neu aufgesetzt wird. **Prompt prüfen.**
- **Erwartet:** eine Sicherheitsabfrage `Are you sure you want to proceed? [y/N]`, nach `y` die Liste der gelöschten Dateien und Ordner und Hinweise auf die nicht entfernten Teile.
- **Prüfen:** `ls /etc/kubernetes/manifests` zeigt einen leeren Ordner oder meldet, dass er nicht existiert.

```bash
sudo apt-mark unhold kubelet kubeadm kubectl
```

- **Was:** hebt die Sperre aus [2.5](02-kubernetes-kubeadm.md#25-kubeadm-kubelet-und-kubectl-installieren) auf. Solange die Pakete gesperrt sind, verweigert apt im nächsten Befehl das Entfernen.
- **Wo:** Terminal des Rechners, der neu aufgesetzt wird.
- **Erwartet:** je Paket eine Bestätigungszeile, z. B. `Canceled hold on kubelet.` (deutsch: `Halten-Markierung für kubelet entfernt.`).
- **Prüfen:** `apt-mark showhold` listet die drei Pakete nicht mehr.

Beispiel (ruki-ws-02, 03.11.2026, deutsche Systemsprache):

```
Halten-Markierung für kubelet entfernt.
Halten-Markierung für kubeadm entfernt.
Halten-Markierung für kubectl entfernt.
```

```bash
sudo apt-get purge -y kubeadm kubelet kubectl kubernetes-cni cri-tools
```

- **Was:** entfernt die Kubernetes-Pakete samt ihren Konfigurationsdateien (`purge` statt `remove`). `-y` beantwortet die Rückfrage automatisch mit Ja. Ist ein Paket nicht installiert (z. B. `cri-tools`, das ab Kubernetes 1.36 nicht mehr automatisch mitkommt), meldet apt das nur.
- **Wo:** Terminal des Rechners, der neu aufgesetzt wird.
- **Erwartet:** eine Liste der zu entfernenden Pakete, danach Zeilen zum Entfernen und Löschen der Konfiguration.
- **Prüfen:** `kubeadm version` meldet, dass der Befehl nicht gefunden wurde.

```bash
sudo rm -rf /etc/kubernetes /etc/cni /opt/cni
```

- **Was:** löscht die restlichen Ordner: `/etc/kubernetes` (z. B. alte Sicherungsdateien), `/etc/cni` (CNI-Konfiguration des alten Pod-Netzwerks) und `/opt/cni` (CNI-Programme, werden in [2.5](02-kubernetes-kubeadm.md#25-kubeadm-kubelet-und-kubectl-installieren) mit dem Paket `kubernetes-cni` neu installiert). `-r` löscht Ordner mit Inhalt, `-f` unterdrückt Rückfragen und Fehlermeldungen für nicht vorhandene Pfade.
- **Wo:** Terminal des Rechners, der neu aufgesetzt wird. **Prompt prüfen**, der Befehl fragt nicht nach.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `ls -d /etc/kubernetes /etc/cni /opt/cni` meldet für alle drei, dass sie nicht existieren.

```bash
sudo rm -f /etc/modules-load.d/k8s.conf /etc/apt/sources.list.d/kubernetes.list /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

- **Was:** löscht die Moduldatei aus [2.2](02-kubernetes-kubeadm.md#22-kernel-modul-br_netfilter-laden) sowie Paketquelle und Signaturschlüssel der alten Kubernetes-Version. Alle drei werden in Abschnitt [2](02-kubernetes-kubeadm.md) für die neue Version neu angelegt.
- **Wo:** Terminal des Rechners, der neu aufgesetzt wird.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `ls /etc/apt/sources.list.d/` listet keine `kubernetes.list` mehr.

```bash
grep -n k8s-api /etc/hosts
```

- **Was:** sucht in `/etc/hosts` nach einem Eintrag für den Namen `k8s-api`, den diese Anleitung in [2.6](02-kubernetes-kubeadm.md#26-cluster-initialisieren) verwendet. `-n` zeigt die Zeilennummer.
- **Wo:** Terminal des Rechners, der neu aufgesetzt wird.
- **Erwartet:** keine Ausgabe, oder eine Zeile wie `10:143.93.153.174 k8s-api`.
- **Prüfen:** Gibt es eine Zeile, stammt sie vom alten Cluster und wird mit den nächsten beiden Befehlen entfernt. Ohne Ausgabe weiter mit `~/.kube`.

```bash
sudo cp /etc/hosts /etc/hosts.bak-$(date +%F)
```

- **Was:** sichert `/etc/hosts`. `$(date +%F)` setzt das heutige Datum im Format `JJJJ-MM-TT` in den Dateinamen ein.
- **Wo:** Terminal des Rechners, der neu aufgesetzt wird.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `ls /etc/hosts.bak-*` zeigt die Sicherung.

```bash
sudo sed -i '/k8s-api/d' /etc/hosts
```

- **Was:** löscht in `/etc/hosts` jede Zeile, die `k8s-api` enthält. `-i` ändert die Datei direkt, `/k8s-api/d` bedeutet „Zeilen mit diesem Text löschen“.
- **Wo:** Terminal des Rechners, der neu aufgesetzt wird.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `grep -n k8s-api /etc/hosts` gibt nichts mehr aus.

```bash
mv ~/.kube ~/.kube.bak-$(date +%F)
```

- **Was:** benennt den Ordner `~/.kube` um. Er enthält die Zugangsdatei des alten Clusters (`config`), die `kubeadm reset` nicht löscht. Umbenennen statt Löschen hält die Datei zur Sicherheit noch eine Weile vor; sie gibt aber Administratorrechte auf den alten Cluster und sollte gelöscht werden, sobald sie nicht mehr gebraucht wird.
- **Wo:** Terminal des Rechners, der neu aufgesetzt wird, als normaler Benutzer (ohne `sudo`).
- **Erwartet:** keine Ausgabe, oder `Datei oder Verzeichnis nicht gefunden` (englisch: `No such file or directory`), wenn es keinen solchen Ordner gab.
- **Prüfen:** `ls -d ~/.kube` meldet, dass der Ordner nicht existiert.

```bash
sudo reboot
```

- **Was:** startet den Rechner neu. Dabei verschwinden die Netzwerkregeln und virtuellen Schnittstellen des alten Pod-Netzwerks (z. B. `cni0`, `flannel.1`).
- **Wo:** Terminal des Rechners, der neu aufgesetzt wird. Laufen dort Docker-Container anderer Nutzer, vorher den Hinweis in [2.4](02-kubernetes-kubeadm.md#24-container-runtime-containerd) lesen.
- **Erwartet:** die Verbindung bricht ab, nach einigen Minuten ist der Rechner wieder erreichbar.
- **Prüfen:** `ip link show cni0` meldet nach dem Neustart, dass die Schnittstelle nicht existiert.

Danach mit Abschnitt [2](02-kubernetes-kubeadm.md) beginnen.

Weiter: [2. Kubernetes mit kubeadm](02-kubernetes-kubeadm.md) | [Übersicht](../README.md)
