Zurück: [1. Voraussetzungen und Kompatibilität prüfen](01-voraussetzungen.md) | [Übersicht](../README.md)

# 2. Kubernetes mit kubeadm

Aufbau der Befehle (Was, Wo, Erwartet, Prüfen) und Hinweise zur Sprache der Ausgaben: siehe [Übersicht](../README.md#aufbau-jedes-befehls). Quellen: siehe [Übersicht](../README.md#quellen).

In diesem Abschnitt wird der Rechner vorbereitet und ein Kubernetes-Cluster mit genau einem Knoten erstellt. Dieser Knoten ist gleichzeitig Control Plane (verwaltet den Cluster) und Worker (führt die Pods aus).

Alle Befehle laufen im Terminal des Laborrechners. Befehle mit `sudo` brauchen Administratorrechte. Vor `kubeadm init` und vor jedem Befehl, der etwas löscht, im Prompt den Rechnernamen prüfen.

## 2.1 Swap ausschalten

Nur nötig, wenn `swapon --show` in Abschnitt [1.2](01-voraussetzungen.md#12-den-rechner-prüfen) etwas ausgegeben hat.

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
- **Prüfen:** `grep swap /etc/fstab` zeigt die Swap-Zeile nur noch mit `#` am Anfang (unter Ubuntu meist `#/swap.img ...`). Gibt `grep` gar nichts aus, bindet `/etc/fstab` keinen Swap ein, und es ist nichts zu tun.

Beispiel (ruki-ws-02, 03.11.2026): `grep swap /etc/fstab` gibt nichts aus. Der Rechner wurde ohne Swap installiert, `sed` hatte nichts zu ändern.

## 2.2 Kernel-Modul br_netfilter laden

Flannel (das Pod-Netzwerk aus [2.8](#28-pod-netzwerk-flannel)) braucht das Kernel-Modul `br_netfilter`. kubeadm prüft es seit Version 1.30 nicht mehr, ohne das Modul startet Flannel nicht richtig (https://github.com/flannel-io/flannel).

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
- **Prüfen:** `lsmod | grep br_netfilter` zeigt zwei Zeilen: eine für `br_netfilter` und eine für das Modul `bridge`, das `br_netfilter` benutzt (letzte Spalte `br_netfilter`).

Beispiel für `lsmod | grep br_netfilter` (ruki-ws-02, 03.11.2026):

```
br_netfilter           32768  0
bridge                421888  1 br_netfilter
```

## 2.3 IP-Weiterleitung einschalten

Pods kommunizieren über Netzwerkpakete, die der Rechner weiterleiten muss. Kubernetes verlangt dafür `net.ipv4.ip_forward = 1` (Container Runtimes, Abschnitt *Network configuration*).

Ist Docker installiert, ist die Weiterleitung oft schon an: `sysctl net.ipv4.ip_forward` zeigt dann bereits `= 1`. Die Datei aus dem nächsten Befehl wird trotzdem angelegt. Docker schaltet die Weiterleitung beim Start nur ein, wenn sie noch aus ist, und setzt in diesem Fall die Standardregel der iptables-Kette `FORWARD` auf `DROP` (Docker-Dokumentation, *Packet filtering and firewalls*). Mit der Datei ist die Weiterleitung schon beim Systemstart an, bevor Docker startet.

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
- **Erwartet:** eine Liste der eingelesenen Dateien, darunter `* Applying /etc/sysctl.d/k8s.conf ...` (deutsch: `* /etc/sysctl.d/k8s.conf wird angewendet …`), und am Ende die gesetzten Werte.
- **Prüfen:** `sysctl net.ipv4.ip_forward` gibt `net.ipv4.ip_forward = 1` aus.

Beispiel (ruki-ws-02, Docker vorinstalliert, 03.11.2026): Schon vor diesem Schritt zeigte `sysctl net.ipv4.ip_forward` den Wert `1`. `sudo sysctl --system` gab unter anderem `* /etc/sysctl.d/k8s.conf wird angewendet …` und `net.ipv4.ip_forward = 1` aus.

## 2.4 Container Runtime: containerd

Kubernetes startet Container nicht selbst, sondern über eine Container Runtime. Wir verwenden containerd. Für Kubernetes 1.36 muss es containerd 2.x sein: Erst ab containerd 2.0 kann der kubelet den cgroup-Treiber automatisch von der Runtime abfragen, und Kubernetes hat die Unterstützung für containerd 1.x auslaufen lassen (Kubernetes-Blog vom 12.09.2025, https://v1-36.docs.kubernetes.io/blog/2025/09/12/kubernetes-v1-34-cri-cgroup-driver-lookup-now-ga/).

```bash
containerd --version
```

- **Was:** zeigt, ob containerd installiert ist und in welcher Version.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile wie `containerd containerd.io v2.x.x ...`, oder die Meldung, dass der Befehl nicht gefunden wurde.
- **Prüfen:** Die Versionsnummer beginnt mit `v2.`. Je nach Ergebnis geht es mit einem der drei Fälle weiter.

Beispiel (ruki-ws-02, Docker vorinstalliert, 03.11.2026): `containerd containerd.io v2.2.4 193637f7…`

| Fall | Ergebnis von `containerd --version` | Vorgehen |
|---|---|---|
| A | Befehl nicht gefunden | Paketquelle von Docker einrichten (https://docs.docker.com/engine/install/ubuntu/, Abschnitt *Install using the apt repository*, Schritt 1) und dann nur das Paket `containerd.io` installieren: `sudo apt-get install -y containerd.io`. Die containerd-Dokumentation nennt dieses Paket als eine der Installationsmöglichkeiten (https://containerd.io/docs/getting-started/). Danach weiter wie Fall B. *Noch nicht auf einem frischen Rechner getestet.* |
| B | `v2.x.x` (z. B. weil Docker installiert ist) | Nichts installieren, nur wie unten beschrieben konfigurieren. Getestet auf ruki-ws-02 am 03.11.2026. |
| C | `v1.x.x` | containerd über die Docker-Paketquelle auf 2.x aktualisieren (Docker-Anleitung, Abschnitt *Upgrade Docker Engine*), dann weiter wie Fall B. *Noch nicht getestet.* |

**Laufen auf dem Rechner Docker-Container?** Docker verwendet dasselbe containerd. Ein Neustart von containerd am Ende dieses Abschnitts kann laufende Container betreffen. Deshalb vorher nachsehen:

```bash
sudo docker ps
```

- **Was:** listet die laufenden Docker-Container mit Name, Image und Status.
- **Wo:** Terminal des Laborrechners. Ist Docker nicht installiert, entfällt dieser Schritt.
- **Erwartet:** eine Kopfzeile und eine Zeile pro laufendem Container, oder nur die Kopfzeile.
- **Prüfen:** Steht dort nur die Kopfzeile, ist nichts zu tun. Gehören Container anderen Nutzern, diese vorher informieren und die Container gemeinsam mit ihnen stoppen: `sudo docker stop <name>`. Damit sie nicht bei jedem Neustart automatisch wieder starten: `sudo docker update --restart=no <name>` (Docker-Dokumentation, *Start containers automatically*). Vorher die aktuelle Neustart-Regel notieren (`sudo docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' <name>`), um sie später wiederherzustellen.

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

Die Standardkonfiguration von containerd 2.x bindet zusätzlich alle Dateien aus `/etc/containerd/conf.d/` ein. Eine dort abgelegte Datei, z. B. von einer früheren Installation des NVIDIA Container Toolkit, bleibt also wirksam. Für Abschnitt [3](03-gpu-device-plugin.md) ist das wichtig. `grep -nE 'imports|default_runtime_name' /etc/containerd/config.toml` zeigt beides.

Beispiel (ruki-ws-02, 03.11.2026, NVIDIA Container Toolkit vorinstalliert, eigene Feststellung):

```
8:imports = ['/etc/containerd/conf.d/*.toml']
80:      default_runtime_name = 'runc'
```

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

- **Was:** stellt containerd auf den cgroup-Treiber `systemd` um. Kubernetes empfiehlt ihn, wenn das System cgroup v2 verwendet, wie aktuelle Ubuntu-Versionen (Container Runtimes, Abschnitt *cgroup drivers*).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `grep -n SystemdCgroup /etc/containerd/config.toml` zeigt `SystemdCgroup = true`. Zeigt `grep` gar nichts, steht der Schlüssel nicht in der Standardkonfiguration und muss von Hand im Abschnitt `[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]` ergänzt werden.

Beispiel (ruki-ws-02, 03.11.2026): `109:            SystemdCgroup = true`

```bash
sudo systemctl restart containerd
```

- **Was:** startet containerd neu, damit die neue Konfiguration gilt.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `systemctl is-active containerd docker` gibt für beide Dienste `active` aus (ohne Docker nur `containerd` abfragen).

## 2.5 kubeadm, kubelet und kubectl installieren

Die drei Programme kommen aus der Paketquelle des Kubernetes-Projekts. Für jede Minor-Version gibt es eine eigene Paketquelle; die Befehle unten sind für 1.36 (Installing kubeadm, Kubernetes v1.36). Für eine andere Version überall `v1.36` durch die gewählte Version ersetzen. Beim Wechsel der Version auf einem Rechner, der schon einen Cluster hatte, zuerst Abschnitt [1.3](01-voraussetzungen.md#13-nur-falls-nötig-reste-eines-früheren-clusters-entfernen) ausführen.

```bash
sudo apt-get update
```

- **Was:** lädt die Paketlisten aller eingetragenen Paketquellen neu.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** Zeilen mit `Get:`/`Hit:` (deutsch: `Holen:`/`OK:`) und am Ende `Reading package lists... Done` (deutsch: `Paketlisten werden gelesen… Fertig`).
- **Prüfen:** Keine Zeile beginnt mit `E:` (Fehler).

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

- **Was:** installiert die Programme, die für die Kubernetes-Paketquelle gebraucht werden: `curl` zum Herunterladen des Schlüssels, `gpg` zum Umwandeln, `ca-certificates` für HTTPS. `apt-transport-https` ist auf aktuellen Systemen nur noch ein leeres Übergangspaket. `-y` beantwortet Rückfragen mit Ja.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** die Pakete werden installiert, oder apt meldet, dass sie schon in der neuesten Version vorhanden sind.
- **Prüfen:** `gpg --version` und `curl --version` geben je eine Versionsnummer aus.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

- **Was:** lädt den öffentlichen Signaturschlüssel der Kubernetes-Pakete herunter und speichert ihn so, dass apt ihn verwenden kann. `curl -fsSL`: bei HTTP-Fehlern abbrechen (`-f`), keine Fortschrittsanzeige (`-s`), Fehler aber anzeigen (`-S`), Weiterleitungen folgen (`-L`). `gpg --dearmor` wandelt den Schlüssel aus dem Textformat in das Binärformat um, `-o` legt die Zieldatei fest. Laut Dokumentation ist der Schlüssel für alle Versionen gleich.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe. Existiert die Datei schon, fragt gpg, ob sie überschrieben werden soll: mit `y` bestätigen. Fehlt der Ordner `/etc/apt/keyrings` (nur bei Ubuntu vor 22.04), vorher `sudo mkdir -p -m 755 /etc/apt/keyrings` ausführen.
- **Prüfen:** `ls -l /etc/apt/keyrings/kubernetes-apt-keyring.gpg` zeigt die Datei. Die kubectl-Anleitung ergänzt `sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg`, damit auch apt-Programme ohne Root-Rechte den Schlüssel lesen können (Install and Set Up kubectl on Linux).

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

- **Was:** trägt die Paketquelle für Kubernetes 1.36 ein. `signed-by` legt fest, dass nur Pakete akzeptiert werden, die mit dem eben gespeicherten Schlüssel signiert sind. `tee` überschreibt eine vorhandene `kubernetes.list`; so wird beim Versionswechsel die alte Paketquelle ersetzt.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** die eingetragene Zeile wird noch einmal ausgegeben.
- **Prüfen:** `cat /etc/apt/sources.list.d/kubernetes.list` zeigt genau eine Zeile mit `v1.36`.

```bash
sudo apt-get update
```

- **Was:** lädt die Paketlisten neu, jetzt einschließlich der Kubernetes-Paketquelle.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** wie oben, zusätzlich eine Zeile mit `pkgs.k8s.io`.
- **Prüfen:** Die Zeile mit `core:/stable:/v1.36` erscheint ohne Fehler.

Beispiel (ruki-ws-02, 03.11.2026): `Holen:11 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.36/deb InRelease`

```bash
sudo apt-get install -y kubelet kubeadm kubectl
```

- **Was:** installiert die drei Programme in der neuesten Patch-Version von 1.36. Als Abhängigkeit kommt `kubernetes-cni` mit (die CNI-Programme in `/opt/cni/bin`, die Flannel in [2.8](#28-pod-netzwerk-flannel) braucht). `cri-tools` (`crictl`) kam auf ruki-ws-02 mit 1.36 nicht mehr automatisch mit (eigene Beobachtung, 03.11.2026).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** apt nennt die neuen Pakete `kubeadm kubectl kubelet kubernetes-cni` und installiert sie.
- **Prüfen:** Die Versionsbefehle aus [1.2](01-voraussetzungen.md#12-den-rechner-prüfen) (`kubeadm version -o short`, `kubelet --version`, `kubectl version --client`) zeigen alle `v1.36.x`.

Beispiel (ruki-ws-02, 03.11.2026): installiert wurden `kubeadm`, `kubectl`, `kubelet` in Version `1.36.6-1.1` und `kubernetes-cni` in Version `1.9.1-1.1`.

kubeadm, kubelet und kubectl dürfen nicht durch ein normales `apt upgrade` auf eine andere Version springen. Ein Kubernetes-Upgrade braucht ein eigenes Vorgehen (Installing kubeadm).

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

- **Was:** markiert die drei Pakete als „gehalten“. apt aktualisiert sie dann nicht mehr automatisch. Soll später eine andere Version installiert oder ein Paket entfernt werden, muss die Sperre zuerst mit `sudo apt-mark unhold kubelet kubeadm kubectl` aufgehoben werden (siehe [1.3](01-voraussetzungen.md#13-nur-falls-nötig-reste-eines-früheren-clusters-entfernen)).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** je Paket eine Zeile wie `kubelet set on hold.` (deutsch: `kubelet auf Halten gesetzt.`).
- **Prüfen:** `apt-mark showhold` listet `kubeadm`, `kubectl` und `kubelet`.

Beispiel (ruki-ws-02, 03.11.2026, deutsche Systemsprache):

```
kubelet auf Halten gesetzt.
kubeadm auf Halten gesetzt.
kubectl auf Halten gesetzt.
```

```bash
sudo systemctl enable --now kubelet
```

- **Was:** sorgt dafür, dass der kubelet bei jedem Systemstart startet (`enable`), und startet ihn sofort (`--now`).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe, wenn das Paket den Dienst schon aktiviert hat (so auf ruki-ws-02 am 03.11.2026).
- **Prüfen:** `systemctl is-enabled kubelet` zeigt `enabled`. Bis zu `kubeadm init` in [2.6](#26-cluster-initialisieren) startet der kubelet alle paar Sekunden neu, weil er noch keine Konfiguration hat. Das ist laut Dokumentation normal (Installing kubeadm) und kein Fehler.

## 2.6 Cluster initialisieren

Ohne weitere Angaben schreibt `kubeadm init` die aktuelle IP-Adresse des Rechners an viele Stellen der Konfiguration. Bei einer per DHCP vergebenen Adresse (siehe [1.2](01-voraussetzungen.md#12-den-rechner-prüfen)) ist das ein Risiko. Diese Anleitung verwendet deshalb eine Konfigurationsdatei, die zwei Dinge anders macht:

- etcd lauscht nur auf `127.0.0.1`. Bei einem einzigen Control-Plane-Knoten gibt es keine anderen etcd-Mitglieder, die es über das Netz erreichen müssten.
- Der Cluster wird unter dem Namen `k8s-api` angesprochen (`controlPlaneEndpoint`), nicht unter der IP-Adresse. Dieser Name wird in `/etc/hosts` auf `127.0.0.1` gesetzt.

Den Namen auf `127.0.0.1` zeigen zu lassen, ist eine eigene Entscheidung für den Ein-Rechner-Betrieb und nicht offiziell dokumentiert. Getestet auf ruki-ws-02 am 03.11.2026. Für einen Cluster mit mehreren Rechnern passt sie nicht, weil die anderen Knoten den Namen dann nicht erreichen.

Auch mit dieser Datei bleibt ein Teil an die IP-Adresse gebunden (API-Server, einige Zugangsdateien). Abschnitt [2.11](#211-abhängigkeit-von-der-ip-adresse-weiter-verringern) verringert das weiter, Abschnitt [2.12](#212-reparatur-nach-einem-wechsel-der-ip-adresse) zeigt die Reparatur nach einem Adresswechsel.

```bash
grep -n k8s-api /etc/hosts
```

- **Was:** prüft, ob es in `/etc/hosts` schon einen Eintrag für `k8s-api` gibt.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** Gibt es schon eine Zeile, stammt sie wahrscheinlich von einem früheren Cluster: zuerst die Schritte für `/etc/hosts` aus [1.3](01-voraussetzungen.md#13-nur-falls-nötig-reste-eines-früheren-clusters-entfernen) ausführen.

```bash
echo "127.0.0.1 k8s-api" | sudo tee -a /etc/hosts
```

- **Was:** hängt den Eintrag `127.0.0.1 k8s-api` an `/etc/hosts` an. `-a` (append) hängt an, statt die Datei zu überschreiben. Ohne `-a` wäre der gesamte Inhalt von `/etc/hosts` verloren.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** die Ausgabe `127.0.0.1 k8s-api`.
- **Prüfen:** `getent hosts k8s-api` gibt `127.0.0.1       k8s-api` aus.

```bash
curl -fsSLO https://raw.githubusercontent.com/wdzomoaz/K8s-for-Kubeflow/main/kubeadm-config.yaml
```

- **Was:** lädt die Konfigurationsdatei aus diesem Repository ins aktuelle Verzeichnis. `-O` speichert sie unter ihrem ursprünglichen Namen. Die Kommentare in der Datei erklären jede Einstellung.
- **Wo:** Terminal des Laborrechners, im Home-Verzeichnis.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `grep kubernetesVersion kubeadm-config.yaml` zeigt `kubernetesVersion: v1.36.6`. Wurde in [1.1](01-voraussetzungen.md#11-passende-versionen-bestimmen) eine andere Version gewählt, diese Zeile anpassen (Version wie bei `kubeadm version -o short`).

```bash
sed -i "s/NODE_NAME/$(hostname)/" kubeadm-config.yaml
```

- **Was:** ersetzt den Platzhalter `NODE_NAME` in der letzten Zeile der Datei durch den Namen des Rechners. etcd braucht dort den Namen des eigenen Mitglieds, und der muss dem Knotennamen entsprechen. Die doppelten Anführungszeichen sind nötig, damit die Shell `$(hostname)` vorher einsetzt.
- **Wo:** Terminal des Laborrechners, im selben Verzeichnis wie die Datei.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `tail -1 kubeadm-config.yaml` zeigt `value: <hostname>=https://127.0.0.1:2380`, z. B. `value: ruki-ws-02=https://127.0.0.1:2380`.

```bash
kubeadm config validate --config kubeadm-config.yaml
```

- **Was:** prüft die Datei, ohne etwas am System zu ändern: Syntax, bekannte Felder, gültige Werte (kubeadm config validate). Ein Tippfehler fällt so vor `kubeadm init` auf.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** die Ausgabe `ok`.
- **Prüfen:** Steht etwas anderes da, nennt die Meldung das fehlerhafte Feld.

> **Achtung:** Vor dem nächsten Befehl im Prompt prüfen, dass man auf dem richtigen Rechner ist.

```bash
sudo kubeadm init --config kubeadm-config.yaml 2>&1 | tee ~/kubeadm-init-$(date +%F).log
```

- **Was:** erstellt die Control Plane: Zertifikate, etcd (Datenbank des Clusters), API-Server, Scheduler und Controller Manager. Vorher prüft kubeadm den Rechner (Preflight-Checks) und bricht bei Fehlern ab. `--config` übergibt die Konfigurationsdatei; darin steht auch der Adressbereich für Pod-IPs (`podSubnet: 10.244.0.0/16`), den Flannel erwartet. `2>&1 | tee …` zeigt die Ausgabe am Bildschirm und speichert sie gleichzeitig in einer Datei, mit dem heutigen Datum im Namen.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** nach einigen Minuten die Zeile `Your Kubernetes control-plane has initialized successfully!`, darunter die Befehle für [2.7](#27-kubectl-für-den-eigenen-benutzer-einrichten) und zwei `kubeadm join k8s-api:6443 ...`-Befehle: einer mit `--control-plane` für weitere Control-Plane-Knoten, einer für Worker-Knoten. Für einen Cluster mit nur einem Rechner werden sie nicht gebraucht.
- **Prüfen:** Die Erfolgszeile steht in der Ausgabe. Bricht der Befehl mit `[ERROR ...]` ab, nennt die Meldung den fehlgeschlagenen Preflight-Check (z. B. Swap, Hostname, containerd).

Die gespeicherte Ausgabe enthält in den `join`-Befehlen einen Zugangsschlüssel (Token), der 24 Stunden gültig ist. Die Datei gehört nicht in ein Git-Repository und wird nicht weitergegeben.

Beispiel (ruki-ws-02, 03.11.2026): `kubeadm init` lief erfolgreich durch und zeigte zwei Befehle `kubeadm join k8s-api:6443 ...` (Control Plane und Worker).

## 2.7 kubectl für den eigenen Benutzer einrichten

kubeadm legt die Zugangsdaten zum Cluster in `/etc/kubernetes/admin.conf` ab. Damit kubectl ohne `sudo` funktioniert, kopieren wir sie ins eigene Home-Verzeichnis. Diese Datei gibt volle Administratorrechte auf den Cluster und darf nicht weitergegeben oder in ein Git-Repository eingecheckt werden.

```bash
mkdir -p $HOME/.kube
```

- **Was:** legt den Ordner `.kube` im Home-Verzeichnis an. `-p` verhindert einen Fehler, falls er schon existiert.
- **Wo:** Terminal des Laborrechners, als normaler Benutzer (ohne `sudo`).
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `ls -la $HOME/.kube` zeigt den Ordner mit dem eigenen Benutzer als Besitzer. Liegt darin schon eine Datei `config`, stammt sie von einem früheren Cluster (siehe [1.3](01-voraussetzungen.md#13-nur-falls-nötig-reste-eines-früheren-clusters-entfernen)).

Beispiel (ruki-ws-02, 03.11.2026, deutsche Systemsprache):

```
insgesamt 8
drwxrwxr-x  2 ruki-labor ruki-labor 4096 Nov  3 12:48 .
drwxr-x--- 29 ruki-labor ruki-labor 4096 Nov  3 12:48 ..
```

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

- **Was:** kopiert die Zugangsdatei nach `~/.kube/config`, wo kubectl sie standardmäßig sucht. `-i` fragt nach, bevor eine vorhandene Datei überschrieben wird.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe, oder eine Rückfrage, falls schon eine alte `config` existiert. Diese gehört zu einem früheren Cluster; mit `y` bestätigen.
- **Prüfen:** `ls -l $HOME/.kube/config` zeigt die Datei, noch mit Besitzer `root`.

Beispiel (ruki-ws-02, 03.11.2026): `-rw------- 1 root root 5631 Nov  3 12:49 /home/ruki-labor/.kube/config`

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- **Was:** macht den eigenen Benutzer zum Besitzer der Datei. `$(id -u)` und `$(id -g)` setzen die eigene Benutzer- und Gruppennummer ein. Die Rechte `-rw-------` bleiben: Nur der Besitzer darf die Datei lesen.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `ls -l $HOME/.kube/config` zeigt den eigenen Benutzer als Besitzer, und `kubectl get nodes` zeigt den Knoten, zunächst mit Status `NotReady`, weil das Pod-Netzwerk noch fehlt.

Beispiel für `kubectl get nodes` (ruki-ws-02, 03.11.2026):

```
NAME         STATUS     ROLES           AGE     VERSION
ruki-ws-02   NotReady   control-plane   5m48s   v1.36.6
```

## 2.8 Pod-Netzwerk: Flannel

Die Anleitung legt die Flannel-Version fest, damit jede Installation dasselbe Ergebnis liefert. Stand 03.11.2026 ist das `v0.28.10`. Eine neuere Version kann man auf der Release-Seite (https://github.com/flannel-io/flannel/releases) auswählen und die Versionsnummer in der Adresse ersetzen; sie ist dann aber nicht mit dieser Anleitung getestet.

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/download/v0.28.10/kube-flannel.yml
```

- **Was:** lädt das Manifest, das zum Release `v0.28.10` von Flannel gehört, und legt die darin beschriebenen Objekte im Cluster an (Namespace `kube-flannel`, Berechtigungen, Konfiguration, DaemonSet). Die Flannel-Dokumentation empfiehlt das Manifest aus einem Release (https://github.com/flannel-io/flannel). Es erwartet den Pod-Adressbereich `10.244.0.0/16`, den `kubeadm-config.yaml` setzt.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** sechs Zeilen der Form `... created`.
- **Prüfen:** `kubectl get pods -n kube-flannel` zeigt nach etwa einer Minute einen Pod `kube-flannel-ds-…` mit `1/1` und `Running`. Danach zeigt `kubectl get nodes` den Status `Ready`.

Beispiel (ruki-ws-02, 03.11.2026):

```
namespace/kube-flannel created
serviceaccount/flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

Der Flannel-Pod war nach 19 Sekunden `1/1 Running`, der Knoten danach `Ready`. Die installierte Version lässt sich so ablesen:

```bash
kubectl get daemonset kube-flannel-ds -n kube-flannel -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

- **Was:** gibt nur das Container-Image des Flannel-DaemonSets aus. `-o jsonpath=…` wählt ein einzelnes Feld des Objekts aus, `{"\n"}` hängt einen Zeilenumbruch an.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile mit Image und Versionsnummer.
- **Prüfen:** Die Versionsnummer entspricht der Version in der Adresse oben.

Beispiel (ruki-ws-02, 03.11.2026): `ghcr.io/flannel-io/flannel:v0.28.10`

## 2.9 Control Plane für normale Pods freigeben

kubeadm versieht den Control-Plane-Knoten mit einem Taint (einer Sperre), damit dort keine normalen Anwendungen laufen. Bei einem einzigen Rechner gibt es aber keinen anderen Knoten, also muss die Sperre weg (Creating a cluster with kubeadm, Abschnitt *Control plane node isolation*).

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

- **Was:** entfernt den Taint `node-role.kubernetes.io/control-plane` von allen Knoten. Das `-` am Ende bedeutet „entfernen“.
- **Wo:** Terminal des Laborrechners. `--all` wirkt auf alle Knoten des Clusters, an dem kubectl gerade angemeldet ist: Prompt prüfen.
- **Erwartet:** `node/<hostname> untainted`.
- **Prüfen:** `kubectl describe node <hostname> | grep Taints` zeigt `Taints: <none>`.

Beispiel (ruki-ws-02, 03.11.2026): `node/ruki-ws-02 untainted`, danach `Taints:             <none>`.

## 2.10 Endkontrolle

```bash
kubectl get pods -A
```

- **Was:** listet alle Pods in allen Namespaces (`-A`).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** Pods in `kube-system` (`coredns` zweimal, `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-proxy`, `kube-scheduler`) und in `kube-flannel`.
- **Prüfen:** Alle Pods haben `1/1` und den Status `Running`. Bleibt `coredns` auf `Pending` oder `ContainerCreating`, fehlt meist das Pod-Netzwerk ([2.8](#28-pod-netzwerk-flannel)) oder `br_netfilter` ([2.2](#22-kernel-modul-br_netfilter-laden)).

Beispiel (ruki-ws-02, 03.11.2026, etwa 9 Minuten nach `kubeadm init`):

```
NAMESPACE      NAME                                 READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-7bqd9                1/1     Running   0          2m45s
kube-system    coredns-589f44dc88-g4v8q             1/1     Running   0          9m11s
kube-system    coredns-589f44dc88-kfgsw             1/1     Running   0          9m11s
kube-system    etcd-ruki-ws-02                      1/1     Running   0          9m18s
kube-system    kube-apiserver-ruki-ws-02            1/1     Running   0          9m18s
kube-system    kube-controller-manager-ruki-ws-02   1/1     Running   0          9m20s
kube-system    kube-proxy-7nh7p                     1/1     Running   0          9m11s
kube-system    kube-scheduler-ruki-ws-02            1/1     Running   0          9m18s
```

## 2.11 Abhängigkeit von der IP-Adresse weiter verringern

Nur nötig, wenn die IP-Adresse per DHCP vergeben wird (siehe [1.2](01-voraussetzungen.md#12-den-rechner-prüfen)). Zuerst wird gemessen, in welchen Dateien die Adresse noch steht.

```bash
sudo grep -rn <eigene-IP> /etc/kubernetes
```

- **Was:** sucht die IP-Adresse des Rechners in allen Dateien unter `/etc/kubernetes`. `-r` durchsucht auch Unterordner, `-n` zeigt Zeilennummern. `sudo` ist nötig, weil einige Dateien nur für root lesbar sind. `<eigene-IP>` durch die Adresse aus [1.2](01-voraussetzungen.md#12-den-rechner-prüfen) (`src`) ersetzen.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Liste der Form `Datei:Zeile:Inhalt`.
- **Prüfen:** siehe Beispiel und Tabelle unten.

Beispiel (ruki-ws-02, Kubernetes 1.36.6, 03.11.2026, eigene Messung):

```
/etc/kubernetes/controller-manager.conf:5:    server: https://143.93.153.200:6443
/etc/kubernetes/manifests/kube-apiserver.yaml:5:    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 143.93.153.200:6443
/etc/kubernetes/manifests/kube-apiserver.yaml:15:    - --advertise-address=143.93.153.200
/etc/kubernetes/manifests/kube-apiserver.yaml:47:        host: 143.93.153.200
/etc/kubernetes/manifests/kube-apiserver.yaml:62:        host: 143.93.153.200
/etc/kubernetes/manifests/kube-apiserver.yaml:74:        host: 143.93.153.200
/etc/kubernetes/manifests/etcd.yaml:5:    kubeadm.kubernetes.io/etcd.advertise-client-urls: https://143.93.153.200:2379
/etc/kubernetes/scheduler.conf:5:    server: https://143.93.153.200:6443
/etc/kubernetes/kubelet.conf:5:    server: https://143.93.153.200:6443
```

| Datei | Bedeutung |
|---|---|
| `kube-apiserver.yaml` | Adresse, die der API-Server bekannt gibt, und Ziel seiner drei Gesundheitsprüfungen. Muss die echte Adresse des Rechners sein; nach einem Adresswechsel ist sie zu reparieren ([2.12](#212-reparatur-nach-einem-wechsel-der-ip-adresse)). |
| `etcd.yaml` | Nur eine Anmerkung (Annotation). etcd selbst lauscht dank `kubeadm-config.yaml` nur auf `127.0.0.1`. |
| `kubelet.conf`, `controller-manager.conf`, `scheduler.conf` | Adresse, über die kubelet, Controller Manager und Scheduler den API-Server erreichen. kubeadm 1.36.6 hat hier die lokale Adresse des API-Servers eingetragen, nicht `controlPlaneEndpoint` (eigene Messung). Für `kubelet.conf` erklärt das Feature Gate `ControlPlaneKubeletLocalMode` diesen Eintrag; es ist laut kubeadm-Referenz für 1.35 standardmäßig aktiv (KEP-4471). Für die beiden anderen Dateien fehlt noch eine offizielle Quelle. |

Die drei Zugangsdateien lassen sich auf den Namen `k8s-api` umstellen. Er steht im Zertifikat des API-Servers und zeigt auf `127.0.0.1`. Die Verbindung bleibt also lokal, und das Zertifikat muss nicht neu erstellt werden. Nach einem Adresswechsel ist dann nur noch `kube-apiserver.yaml` zu reparieren. Auf ruki-ws-07 waren alle fünf Zugangsdateien bereits so eingestellt, und dort genügte am 03.11.2026 diese eine Reparatur. Die Umstellung ist ein eigener Eingriff, kein Standardschritt von kubeadm. Getestet auf ruki-ws-02 am 03.11.2026.

> **Achtung:** Laut KEP-4471 soll `kubeadm upgrade` die Datei `kubelet.conf` auf die lokale Adresse des API-Servers einstellen; die Umstellung kann also bei einem Upgrade verloren gehen. Ob `kubeadm certs renew` die Zeile `server:` verändert, ist nicht dokumentiert. Nach jedem `kubeadm upgrade` und jedem `kubeadm certs renew` deshalb den `grep`-Befehl oben wiederholen und diesen Abschnitt bei Bedarf noch einmal ausführen.

```bash
sudo cp -a /etc/kubernetes ~/kubernetes-backup-$(date +%F)
```

- **Was:** sichert den ganzen Ordner `/etc/kubernetes` vor der Änderung, wie es auch die Dokumentation zur Umkonfiguration empfiehlt (Reconfiguring a kubeadm cluster). `-a` kopiert rekursiv und behält Besitzer, Rechte und Zeitstempel bei; die privaten Schlüssel in `pki/` bleiben so nur für root lesbar. Die Sicherung bleibt auf dem Rechner und gehört nicht in ein Git-Repository.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `sudo diff -r /etc/kubernetes ~/kubernetes-backup-$(date +%F) && echo identisch` gibt `identisch` aus.

```bash
sudo sed -i 's#server: https://<eigene-IP>:6443#server: https://k8s-api:6443#' /etc/kubernetes/kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf
```

- **Was:** ersetzt in den drei Zugangsdateien die Adresse des API-Servers durch den Namen `k8s-api`. Als Trennzeichen dient `#` statt `/`, weil die Adresse selbst `/` enthält. `<eigene-IP>` durch die Adresse ersetzen; die Punkte darin am besten mit `\.` schreiben, damit sie nur für einen Punkt stehen (z. B. `143\.93\.153\.200`). Die laufenden Komponenten bemerken die Änderung erst nach einem Neustart.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `sudo grep -n "server:" /etc/kubernetes/*.conf` zeigt für alle fünf Dateien (`admin.conf`, `controller-manager.conf`, `kubelet.conf`, `scheduler.conf`, `super-admin.conf`) `server: https://k8s-api:6443`.

```bash
sudo systemctl restart kubelet
```

- **Was:** startet den kubelet neu, damit er `kubelet.conf` neu einliest. Laufende Container werden dabei nicht beendet.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `systemctl is-active kubelet && kubectl get nodes` zeigt `active` und den Knoten mit `Ready`.

Controller Manager und Scheduler laufen als statische Pods: Der kubelet startet sie direkt aus den Dateien in `/etc/kubernetes/manifests/`, nicht über den API-Server. `kubectl delete pod` startet sie deshalb nicht neu. Stattdessen nimmt man die Datei kurz aus dem Ordner, wartet etwa 20 Sekunden, bis der kubelet den Pod beendet hat, und legt sie zurück (Certificate Management with kubeadm).

```bash
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /etc/kubernetes/
```

- **Was:** verschiebt das Manifest des Controller Managers eine Ebene höher. Der kubelet überwacht nur `manifests/` und beendet den Pod. Die Datei bleibt im Ordner `/etc/kubernetes/` erhalten und landet nicht in `/tmp`.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** Nach 20 bis 30 Sekunden gibt `kubectl get pods -n kube-system | grep controller-manager` nichts mehr aus. Erst dann weiter.

```bash
sudo mv /etc/kubernetes/kube-controller-manager.yaml /etc/kubernetes/manifests/
```

- **Was:** legt das Manifest zurück. Der kubelet startet den Pod neu, und der Controller Manager liest beim Start `controller-manager.conf` mit der neuen Adresse.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** Nach etwa einer Minute zeigt `kubectl get pods -n kube-system | grep controller-manager` den Pod mit `1/1` und `Running` und einem `AGE` von unter einer Minute. In den ersten Sekunden ist `0/1` normal, weil die Startprüfung erst nach einigen Sekunden beginnt.

```bash
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /etc/kubernetes/
```

- **Was:** wie oben, für den Scheduler. Bereits laufende Pods sind nicht betroffen, nur neue Pods werden in dieser Zeit nicht verteilt.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** Nach 20 bis 30 Sekunden gibt `kubectl get pods -n kube-system | grep scheduler` nichts mehr aus. Erst dann weiter.

```bash
sudo mv /etc/kubernetes/kube-scheduler.yaml /etc/kubernetes/manifests/
```

- **Was:** legt das Manifest des Schedulers zurück; der Scheduler startet mit `scheduler.conf` neu.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** Nach etwa einer Minute zeigt `kubectl get pods -n kube-system | grep scheduler` den Pod mit `1/1` und `Running`. Ein Neustartzähler von `1` ist möglich und zählt das gewollte Beenden mit; er darf danach nicht weiter steigen.

Beispiel (ruki-ws-02, 03.11.2026): `kube-scheduler-ruki-ws-02   1/1   Running   1 (63s ago)   48s`

**Kontrolle.** Der `grep`-Befehl vom Anfang dieses Abschnitts zeigt jetzt nur noch die fünf Zeilen in `kube-apiserver.yaml` und die Anmerkung in `etcd.yaml`. `kubectl get pods -A` zeigt alle Pods mit `1/1 Running`. Und die Protokolle beider Komponenten enthalten keine Zertifikats- oder Verbindungsfehler:

```bash
kubectl logs -n kube-system kube-controller-manager-<hostname> | grep -iE "x509|certificate|tls|refused" | tail -5
```

- **Was:** durchsucht das Protokoll des Controller Managers nach typischen Wörtern bei Zertifikats- oder Verbindungsproblemen. `-i` ignoriert Groß- und Kleinschreibung, `-E` erlaubt die Oder-Verknüpfung mit `|`. `tail -5` zeigt höchstens die letzten fünf Treffer. Für den Scheduler denselben Befehl mit `kube-scheduler-<hostname>` ausführen.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe, oder nur Zeilen, die mit `I` (Information) beginnen, z. B. `Starting certificate controller` oder `Starting DynamicServingCertificateController`.
- **Prüfen:** Es gibt keine Zeile, die mit `E` (Fehler) beginnt und `x509` oder `refused` enthält.

Beispiel (ruki-ws-02, 03.11.2026): Beim Controller Manager erschienen nur Zeilen `I1103 … "Starting certificate controller"`, beim Scheduler nur `I1103 … "Starting DynamicServingCertificateController"`. Die Zeitstempel in den Protokollen sind in UTC.

## 2.12 Reparatur nach einem Wechsel der IP-Adresse

Symptom: `kubectl` meldet `connection refused`, und `kubectl get nodes` antwortet nicht mehr. Ursache ist meist, dass der DHCP-Server dem Rechner eine neue Adresse gegeben hat. `kube-apiserver.yaml` nennt dann noch die alte Adresse; die Gesundheitsprüfungen schlagen fehl, und der kubelet startet den API-Server immer wieder neu. Genau so ist es auf ruki-ws-07 am 03.11.2026 passiert (eigene Beobachtung).

Die folgenden Schritte setzen voraus, dass [2.11](#211-abhängigkeit-von-der-ip-adresse-weiter-verringern) ausgeführt wurde. Sonst müssen auch die drei Zugangsdateien aus [2.11](#211-abhängigkeit-von-der-ip-adresse-weiter-verringern) angepasst werden.

```bash
ip route show default
```

- **Was:** zeigt die aktuelle Adresse des Rechners (wie in [1.2](01-voraussetzungen.md#12-den-rechner-prüfen)).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile mit `src <neue IP>`.
- **Prüfen:** Die Adresse hinter `src` mit der notierten Adresse vergleichen. Ist sie gleich, liegt die Ursache woanders.

```bash
sudo grep -n advertise-address= /etc/kubernetes/manifests/kube-apiserver.yaml
```

- **Was:** zeigt die Adresse, die der API-Server noch verwendet.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile wie `15:    - --advertise-address=<alte IP>`.
- **Prüfen:** Die alte Adresse unterscheidet sich von der neuen aus dem vorigen Befehl.

> **Achtung:** Vor dem nächsten Befehl `/etc/kubernetes` wie in [2.11](#211-abhängigkeit-von-der-ip-adresse-weiter-verringern) sichern.

```bash
sudo sed -i 's/<alte IP>/<neue IP>/g' /etc/kubernetes/manifests/kube-apiserver.yaml
```

- **Was:** ersetzt im Manifest des API-Servers jedes Vorkommen der alten Adresse durch die neue (`g` = alle Vorkommen in einer Zeile). Betroffen sind die Anmerkung, `--advertise-address` und die drei Gesundheitsprüfungen. Weil sich eine Datei in `manifests/` ändert, startet der kubelet den API-Server von selbst neu (Reconfiguring a kubeadm cluster). Die Punkte in der alten Adresse am besten mit `\.` schreiben.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `sudo grep -c <neue IP> /etc/kubernetes/manifests/kube-apiserver.yaml` gibt `5` aus. Nach ein bis zwei Minuten antwortet `kubectl get nodes` wieder.

Diese Reparatur wurde auf ruki-ws-07 am 03.11.2026 erfolgreich angewendet. Auf ruki-ws-02 ist sie noch nicht mit einem echten Adresswechsel getestet. Offen ist dort auch noch, ob danach `kubectl get nodes -o wide` in der Spalte `INTERNAL-IP` die neue Adresse zeigt oder ob der kubelet dafür neu gestartet werden muss.

<!-- TODO: Reparatur auf ruki-ws-02 bei einem echten Adresswechsel testen; eventuell als Skript -->

Weiter: [3. GPU-Check und NVIDIA Device Plugin](03-gpu-device-plugin.md) | [Übersicht](../README.md)
