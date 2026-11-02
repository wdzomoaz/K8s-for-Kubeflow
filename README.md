# K8s-for-Kubeflow

Installationsanleitung für Kubernetes und Kubeflow auf einem einzelnen Linux-Rechner im Labor (kein verteilter Cluster). Am Ende läuft eine Kubeflow-Pipeline mit einem GPU-Schritt.

Diese Anleitung legt keine feste Version fest. Sie zeigt, wie man prüft, welche Kubernetes-Version zu welcher Kubeflow-Version passt. Die Versionsangaben in den Beispielen sind eine Momentaufnahme (Stand 02.11.2026) und müssen vor jeder neuen Installation erneut geprüft werden.

## Inhalt

1. [Voraussetzungen und Kompatibilität prüfen](#1-voraussetzungen-und-kompatibilität-prüfen)
2. [Kubernetes mit kubeadm](#2-kubernetes-mit-kubeadm)
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

## 2. Kubernetes mit kubeadm

In diesem Abschnitt wird der Rechner vorbereitet und ein Kubernetes-Cluster mit genau einem Knoten erstellt. Dieser Knoten ist gleichzeitig Control Plane (verwaltet den Cluster) und Worker (führt die Pods aus).

Alle Befehle laufen im Terminal des Laborrechners. Befehle mit `sudo` brauchen Administratorrechte.

### 2.1 Swap ausschalten

Nur nötig, wenn `swapon --show` in Abschnitt 1.2 etwas ausgegeben hat.

```bash
sudo swapoff -a
```

- **Was:** schaltet alle aktiven Swap-Bereiche sofort ab. `-a` steht für alle Einträge. Die Änderung gilt nur bis zum nächsten Neustart.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `swapon --show` gibt nichts mehr aus.

```bash
sudo sed -i '/\sswap\s/ s/^/#/' /etc/fstab
```

- **Was:** kommentiert in `/etc/fstab` jede Zeile aus, die einen Swap-Bereich einbindet, damit Swap auch nach einem Neustart aus bleibt. `sed -i` ändert die Datei direkt, `/\sswap\s/` wählt die Zeilen mit dem Wort `swap`, `s/^/#/` setzt ein `#` an den Zeilenanfang.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `grep swap /etc/fstab` zeigt die Swap-Zeile nur noch mit `#` am Anfang (unter Ubuntu meist `#/swap.img ...`).

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

### 2.2 Kernel-Modul br_netfilter laden

Flannel (das Pod-Netzwerk aus 2.8) braucht das Kernel-Modul `br_netfilter`. kubeadm prüft es seit Version 1.30 nicht mehr, ohne das Modul startet Flannel nicht richtig (https://github.com/flannel-io/flannel).

```bash
echo br_netfilter | sudo tee /etc/modules-load.d/k8s.conf
```

- **Was:** schreibt den Modulnamen in eine Datei unter `/etc/modules-load.d/`. systemd lädt alle dort eingetragenen Module bei jedem Systemstart. `sudo tee` wird gebraucht, weil eine einfache Umleitung (`>`) keine Root-Rechte hat.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** die Ausgabe `br_netfilter`.
- **Prüfen:** `cat /etc/modules-load.d/k8s.conf` zeigt `br_netfilter`.

```bash
sudo modprobe br_netfilter
```

- **Was:** lädt das Modul sofort, ohne Neustart.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `lsmod | grep br_netfilter` zeigt eine Zeile mit `br_netfilter`.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

### 2.3 IP-Weiterleitung einschalten

Pods kommunizieren über Netzwerkpakete, die der Rechner weiterleiten muss. Kubernetes verlangt dafür `net.ipv4.ip_forward = 1` (Container Runtimes, Abschnitt *Network configuration*).

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/k8s.conf
```

- **Was:** legt eine Konfigurationsdatei an, die die IP-Weiterleitung bei jedem Systemstart einschaltet.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** die Ausgabe `net.ipv4.ip_forward = 1`.
- **Prüfen:** `cat /etc/sysctl.d/k8s.conf` zeigt dieselbe Zeile.

```bash
sudo sysctl --system
```

- **Was:** liest alle sysctl-Konfigurationsdateien neu ein und wendet sie sofort an, ohne Neustart.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Liste der eingelesenen Dateien, darunter `* Applying /etc/sysctl.d/k8s.conf ...`.
- **Prüfen:** `sysctl net.ipv4.ip_forward` gibt `net.ipv4.ip_forward = 1` aus.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

### 2.4 Container Runtime: containerd

Kubernetes startet Container nicht selbst, sondern über eine Container Runtime. Wir verwenden containerd. Für Kubernetes 1.36 muss es containerd 2.x sein: Erst ab containerd 2.0 kann der kubelet den cgroup-Treiber automatisch von der Runtime abfragen, und Kubernetes hat die Unterstützung für containerd 1.x auslaufen lassen (Kubernetes-Blog vom 12.09.2025, https://v1-36.docs.kubernetes.io/blog/2025/09/12/kubernetes-v1-34-cri-cgroup-driver-lookup-now-ga/).

```bash
containerd --version
```

- **Was:** zeigt, ob containerd installiert ist und in welcher Version.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile wie `containerd containerd.io 2.x.x ...`.
- **Prüfen:** Die Version beginnt mit `2.`. Fehlt containerd oder ist es Version 1.x: Docker Engine nach der offiziellen Anleitung installieren bzw. aktualisieren (https://docs.docker.com/engine/install/ubuntu/). Sie bringt das Paket `containerd.io` mit; die containerd-Dokumentation nennt diesen Weg als eine der Installationsmöglichkeiten (https://containerd.io/docs/getting-started/).

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

Das Paket `containerd.io` von Docker liefert eine Konfiguration, in der die Kubernetes-Schnittstelle (CRI) abgeschaltet ist. Deshalb ersetzen wir sie durch die Standardkonfiguration von containerd. Vorher sichern wir die alte Datei.

```bash
sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
```

- **Was:** kopiert die bestehende Konfiguration in eine Sicherungsdatei.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe. Meldet `cp`, dass die Datei nicht existiert, gibt es nichts zu sichern, und man macht einfach weiter.
- **Prüfen:** `ls /etc/containerd/` zeigt `config.toml.bak`.

```bash
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

- **Was:** `containerd config default` gibt die vollständige Standardkonfiguration aus, `sudo tee` schreibt sie nach `/etc/containerd/config.toml`. `> /dev/null` unterdrückt die lange Bildschirmausgabe.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `grep disabled_plugins /etc/containerd/config.toml` zeigt `disabled_plugins = []`, also ist `cri` nicht abgeschaltet.

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

- **Was:** stellt containerd auf den cgroup-Treiber `systemd` um. Kubernetes empfiehlt ihn, wenn das System cgroup v2 verwendet, wie aktuelle Ubuntu-Versionen (Container Runtimes, Abschnitt *cgroup drivers*).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `grep SystemdCgroup /etc/containerd/config.toml` zeigt `SystemdCgroup = true`. Zeigt `grep` gar nichts, steht der Schlüssel nicht in der Standardkonfiguration und muss von Hand im Abschnitt `[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]` ergänzt werden.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

```bash
sudo systemctl restart containerd
```

- **Was:** startet containerd neu, damit die neue Konfiguration gilt.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `systemctl is-active containerd` gibt `active` aus.

### 2.5 Kubernetes-Pakete gegen Updates sperren

kubeadm, kubelet und kubectl dürfen nicht durch ein normales `apt upgrade` auf eine andere Version springen. Ein Kubernetes-Upgrade braucht ein eigenes Vorgehen (Installing kubeadm).

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

- **Was:** markiert die drei Pakete als „gehalten“. apt aktualisiert sie dann nicht mehr automatisch.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** je Paket eine Zeile wie `kubelet set on hold.`; war es schon gesperrt, steht dort `kubelet was already set on hold.`
- **Prüfen:** `apt-mark showhold` listet `kubeadm`, `kubectl` und `kubelet`.

### 2.6 Cluster initialisieren

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

- **Was:** erstellt die Control Plane: Zertifikate, etcd (Datenbank des Clusters), API-Server, Scheduler und Controller Manager. Vorher prüft kubeadm den Rechner (Preflight-Checks) und bricht bei Fehlern ab. `--pod-network-cidr=10.244.0.0/16` legt den Adressbereich für Pod-IPs fest; Flannel erwartet genau diesen Bereich (https://github.com/flannel-io/flannel/blob/master/Documentation/kubernetes.md).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** nach einigen Minuten die Zeile `Your Kubernetes control-plane has initialized successfully!` und darunter die Befehle für 2.7.
- **Prüfen:** Die Erfolgszeile steht in der Ausgabe. Bricht der Befehl mit `[ERROR ...]` ab, nennt die Meldung den fehlgeschlagenen Preflight-Check (z. B. Swap, Hostname, containerd).

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

### 2.7 kubectl für den eigenen Benutzer einrichten

kubeadm legt die Zugangsdaten zum Cluster in `/etc/kubernetes/admin.conf` ab. Damit kubectl ohne `sudo` funktioniert, kopieren wir sie ins eigene Home-Verzeichnis. Diese Datei gibt volle Administratorrechte auf den Cluster und darf nicht weitergegeben oder in ein Git-Repository eingecheckt werden.

```bash
mkdir -p $HOME/.kube
```

- **Was:** legt den Ordner `.kube` im Home-Verzeichnis an. `-p` verhindert einen Fehler, falls er schon existiert.
- **Wo:** Terminal des Laborrechners, als normaler Benutzer (ohne `sudo`).
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `ls -d $HOME/.kube` zeigt den Ordner.

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

- **Was:** kopiert die Zugangsdatei nach `~/.kube/config`, wo kubectl sie standardmäßig sucht. `-i` fragt nach, bevor eine vorhandene Datei überschrieben wird.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe, oder eine Rückfrage, falls schon eine alte `config` existiert.
- **Prüfen:** `ls -l $HOME/.kube/config` zeigt die Datei, noch mit Besitzer `root`.

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- **Was:** macht den eigenen Benutzer zum Besitzer der Datei. `$(id -u)` und `$(id -g)` setzen die eigene Benutzer- und Gruppennummer ein.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `kubectl get nodes` zeigt den Knoten, zunächst mit Status `NotReady`, weil das Pod-Netzwerk noch fehlt.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

### 2.8 Pod-Netzwerk: Flannel

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

- **Was:** lädt das Manifest der neuesten Flannel-Version und legt die darin beschriebenen Objekte im Cluster an (Namespace `kube-flannel`, Berechtigungen, Konfiguration, DaemonSet). Der Befehl stammt aus der Flannel-Dokumentation (https://github.com/flannel-io/flannel).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** mehrere Zeilen der Form `... created`, z. B. `namespace/kube-flannel created` und `daemonset.apps/kube-flannel-ds created`.
- **Prüfen:** `kubectl get pods -n kube-flannel` zeigt nach etwa einer Minute einen Pod mit Status `Running`. Danach zeigt `kubectl get nodes` den Status `Ready`.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

### 2.9 Control Plane für normale Pods freigeben

kubeadm versieht den Control-Plane-Knoten mit einem Taint (einer Sperre), damit dort keine normalen Anwendungen laufen. Bei einem einzigen Rechner gibt es aber keinen anderen Knoten, also muss die Sperre weg (Creating a cluster with kubeadm, Abschnitt *Control plane node isolation*).

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

- **Was:** entfernt den Taint `node-role.kubernetes.io/control-plane` von allen Knoten. Das `-` am Ende bedeutet „entfernen“.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `node/<hostname> untainted`.
- **Prüfen:** `kubectl describe node | grep Taints` zeigt `Taints: <none>`.

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

### 2.10 Endkontrolle

```bash
kubectl get pods -A
```

- **Was:** listet alle Pods in allen Namespaces (`-A`).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** Pods in `kube-system` (u. a. `coredns`, `etcd`, `kube-apiserver`, `kube-proxy`) und in `kube-flannel`.
- **Prüfen:** Alle Pods haben den Status `Running`. Bleibt `coredns` auf `Pending` oder `ContainerCreating`, fehlt meist das Pod-Netzwerk (2.8) oder `br_netfilter` (2.2).

<!-- TODO: Ausgabe vom Laborrechner einfügen (Test 03.11.2026) -->

## Quellen

- Kubeflow Releases: https://www.kubeflow.org/docs/releases/
- Kubeflow Community Distribution 26.03: https://www.kubeflow.org/docs/releases/kubeflow-26.03/
- Release Notes der Community Distribution: https://github.com/kubeflow/community-distribution/releases
- Kubeflow Community Distribution (Repository): https://github.com/kubeflow/community-distribution
- Kubernetes Version Skew Policy: https://kubernetes.io/releases/version-skew-policy/
- Installing kubeadm (Kubernetes v1.36): https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Object Names and IDs: https://kubernetes.io/docs/concepts/overview/working-with-objects/names/
- Container Runtimes: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- Kubernetes v1.34: Autoconfiguration for Node Cgroup Driver Goes GA (Blog): https://v1-36.docs.kubernetes.io/blog/2025/09/12/kubernetes-v1-34-cri-cgroup-driver-lookup-now-ga/
- Getting started with containerd: https://containerd.io/docs/getting-started/
- Install Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Creating a cluster with kubeadm (Kubernetes v1.36): https://v1-36.docs.kubernetes.io/docs/setup/independent/create-cluster-kubeadm
- Flannel: https://github.com/flannel-io/flannel
- Flannel mit Kubernetes: https://github.com/flannel-io/flannel/blob/master/Documentation/kubernetes.md

Alle Links geprüft am 02.11.2026.
