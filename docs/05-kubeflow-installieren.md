Zurück: [4. Speicher: Local Path Provisioner als Default StorageClass](04-local-path-provisioner.md) | [Übersicht](../README.md)

# 5. Kubeflow installieren

Aufbau der Befehle (Was, Wo, Erwartet, Prüfen) und Hinweise zur Sprache der Ausgaben: siehe [Übersicht](../README.md#aufbau-jedes-befehls). Quellen: siehe [Übersicht](../README.md#quellen).

Kubeflow wird aus den Manifesten der Kubeflow Community Distribution installiert. Deren README beschreibt zwei Wege: alle Komponenten mit einem einzigen Befehl oder jede Komponente einzeln (README, Abschnitt *Installation*). Diese Anleitung nimmt den einzelnen Befehl. Das README setzt dafür einen Kind-Cluster voraus, den ein Skript aus dem Repository anlegt. Wir haben stattdessen den Cluster aus den Abschnitten 2 bis 4 und überspringen deshalb die Schritte *Create Kind Cluster* und *Save Kubeconfig*.

Abschnitt [5.1](#51-vor-der-installation-prüfen) prüft Speicherplatz und Ziel-Cluster. [5.2](#52-kustomize-installieren) installiert Kustomize, [5.3](#53-manifeste-herunterladen) lädt die Manifeste des gewählten Releases. [5.4](#54-kernel-parameter-für-inotify) erhöht zwei Kernel-Parameter. [5.5](#55-alle-komponenten-installieren) installiert Kubeflow, [5.6](#56-pods-prüfen) prüft, ob alle Pods bereit sind.

## 5.1 Vor der Installation prüfen

**Ressourcen.** Das README des Tags `26.03.1` empfiehlt für die Installation mit einem Befehl 16 GB RAM und 8 CPU-Kerne (Abschnitt *Install with a Single Command*, *Prerequisites*). Die Tabelle im Abschnitt *Kubeflow Components Versions* schätzt den Bedarf aller Komponenten zusammen auf 4380m CPU, 12341Mi Arbeitsspeicher und 65 GB für PVCs. RAM und CPU-Kerne wurden schon in [1.2](01-voraussetzungen.md#12-den-rechner-prüfen) geprüft. Die PVCs landen über den Local Path Provisioner in `/opt/local-path-provisioner` (Abschnitt 4), deshalb hier der freie Platz an dieser Stelle:

```bash
df -h /opt
```

- **Was:** `df` zeigt Größe, belegten und freien Platz eines Dateisystems. `-h` gibt die Werte in K, M, G und T aus. Mit `/opt` zeigt `df` das Dateisystem, auf dem `/opt` liegt, nicht die Größe des Ordners selbst.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Kopfzeile und eine Zeile mit dem Dateisystem. Deutsch: `Dateisystem Größe Benutzt Verf. Verw% Eingehängt auf`, Englisch: `Filesystem Size Used Avail Use% Mounted on`.
- **Prüfen:** In der Spalte `Verf.` (`Avail`) stehen deutlich mehr als 65 GB. Steht in der letzten Spalte `/`, liegen auch die Container-Images (`/var/lib/containerd`) auf diesem Dateisystem und brauchen zusätzlichen Platz.

Beispiel (ruki-ws-02, 03.11.2026):

```
Dateisystem    Größe Benutzt Verf. Verw% Eingehängt auf
/dev/nvme1n1p1  1,8T    385G  1,4T   23% /
```

**Ziel-Cluster.** Die Installation in 5.5 schreibt in den Cluster, auf den die aktuelle kubeconfig zeigt. Vorher sicherstellen, dass das der Cluster dieses Rechners ist und kein anderer:

```bash
kubectl get nodes -o wide
```

- **Was:** listet die Knoten des Clusters aus der aktuellen kubeconfig (`~/.kube/config`). `-o wide` zeigt zusätzlich unter anderem `INTERNAL-IP` und `KERNEL-VERSION`.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** genau ein Knoten mit dem Namen des Rechners, `STATUS Ready`, `ROLES control-plane` und der IP-Adresse des Rechners.
- **Prüfen:** Name und IP gehören zu diesem Rechner. Erscheinen andere Knoten, zeigt die kubeconfig auf einen anderen Cluster: hier abbrechen.

Beispiel (ruki-ws-02, 03.11.2026):

```
NAME         STATUS   ROLES           AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION             CONTAINER-RUNTIME
ruki-ws-02   Ready    control-plane   3h29m   v1.36.6   143.93.153.200   <none>        Ubuntu 24.04.5 LTS   7.0.0-30-generic (amd64)   containerd://2.2.4
```

## 5.2 Kustomize installieren

Die Manifeste von Kubeflow sind in Kustomize-Verzeichnissen organisiert. Das Programm `kustomize` setzt sie zu fertigen YAML-Dateien zusammen, die `kubectl apply` an den Cluster schickt. Die validierte Kustomize-Version steht auf der Release-Seite unter *Dependency Versions (Manifests)*, für 26.03: `5.8.1` (siehe [1.1](01-voraussetzungen.md#11-passende-versionen-bestimmen)).

**Warum nicht `kubectl kustomize`?** kubectl enthält eine eigene Kustomize-Version, bei kubectl 1.36.6 ebenfalls `v5.8.1`. Sie wird mit `kubectl kustomize <Verzeichnis>` aufgerufen (kubectl kustomize, kubernetes.io). Alle Befehle im README der Community Distribution und die Skripte in `tests/` rufen aber das eigenständige Programm `kustomize` auf. Damit sie sich unverändert übernehmen lassen, installieren wir es. Ob `kubectl kustomize example` dasselbe Ergebnis liefert, ist *nicht getestet*.

```bash
kustomize version
```

- **Was:** gibt die Version des eigenständigen Programms `kustomize` aus.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Version wie `v5.8.1` oder die Meldung, dass der Befehl nicht gefunden wurde. Deutsch: `Der Befehl 'kustomize' wurde nicht gefunden, …`, Englisch: `Command 'kustomize' not found, …`.
- **Prüfen:** Je nach Ergebnis geht es mit einem der drei Fälle weiter.

| Fall | Ergebnis | Vorgehen |
|---|---|---|
| A | `v5.8.1` (die Version des Releases) | Weiter mit 5.3. *Noch nicht getestet.* |
| B | Befehl nicht gefunden | Kustomize wie unten installieren. Getestet auf ruki-ws-02 am 03.11.2026. |
| C | Eine andere Version | Kustomize wie unten installieren. Danach mit `which -a kustomize` prüfen, dass `/usr/local/bin/kustomize` an erster Stelle steht. *Noch nicht getestet.* |

<!-- TODO: Fall A und C auf einem Rechner mit vorhandenem kustomize durchspielen -->

Beispiel (ruki-ws-02, 03.11.2026):

```
Der Befehl 'kustomize' wurde nicht gefunden, kann aber installiert werden mit:
sudo snap install kustomize
```

Den Vorschlag `sudo snap install kustomize` übernehmen wir nicht. Stattdessen wird die fertige Datei aus dem offiziellen Release von Kustomize installiert. So stehen Version und Herkunft fest, und die Prüfsumme lässt sich vor der Installation kontrollieren.

**Release prüfen.** Im Browser die Release-Seite öffnen: https://github.com/kubernetes-sigs/kustomize/releases/tag/kustomize%2Fv5.8.1. Der Tag heißt `kustomize/v5.8.1`, `%2F` ist der Schrägstrich in einer URL. Unter *Assets* müssen die Dateien `kustomize_v5.8.1_linux_amd64.tar.gz` und `checksums.txt` stehen. Für eine andere Version `5.8.1` in diesem Abschnitt überall ersetzen.

```bash
curl -fsSL -o /tmp/kustomize_v5.8.1_linux_amd64.tar.gz https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.8.1/kustomize_v5.8.1_linux_amd64.tar.gz
```

- **Was:** lädt das Archiv nach `/tmp` und behält dabei den Originalnamen, den die Prüfung unten braucht. `-f` meldet einen Fehler, wenn der Server z. B. 404 antwortet, statt die Fehlerseite zu speichern. `-s` blendet den Fortschrittsbalken aus, `-S` zeigt Fehler trotzdem an. `-L` folgt den Weiterleitungen von GitHub. `-o` gibt die Zieldatei an.
- **Wo:** Terminal des Laborrechners (braucht Internetzugang).
- **Erwartet:** keine Ausgabe.
- **Prüfen:** Die Prüfsumme weiter unten kontrolliert die Datei.

```bash
curl -fsSL -o /tmp/kustomize_v5.8.1_checksums.txt https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.8.1/checksums.txt
```

- **Was:** lädt `checksums.txt`, die offizielle Liste der SHA-256-Prüfsummen aller Dateien des Releases. Optionen wie oben.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `grep linux_amd64 /tmp/kustomize_v5.8.1_checksums.txt` zeigt eine Zeile mit der Prüfsumme und dem Dateinamen des Archivs. Der Anfang der Prüfsumme stimmt mit dem Wert auf der Release-Seite überein.

```bash
(cd /tmp && sha256sum --check --ignore-missing kustomize_v5.8.1_checksums.txt)
```

- **Was:** berechnet die SHA-256-Prüfsumme des Archivs und vergleicht sie mit der offiziellen Liste. Die Klammern starten eine Unter-Shell: Das `cd /tmp` gilt nur darin, das Terminal bleibt im bisherigen Ordner. `cd /tmp` ist nötig, weil `checksums.txt` nur Dateinamen ohne Pfad enthält. `--check` liest die Liste und prüft jede genannte Datei. `--ignore-missing` überspringt die Dateien, die nicht heruntergeladen wurden (Windows, macOS, andere Architekturen).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `kustomize_v5.8.1_linux_amd64.tar.gz: OK`
- **Prüfen:** Jede andere Ausgabe als `OK` heißt: Datei beschädigt oder falsch. Dann nichts installieren und das Archiv neu laden.

```bash
tar -xzf /tmp/kustomize_v5.8.1_linux_amd64.tar.gz -C /tmp
```

- **Was:** entpackt das Archiv. Es enthält nur die Datei `kustomize`. `-x` entpackt, `-z` entkomprimiert gzip, `-f` gibt das Archiv an, `-C /tmp` den Zielordner.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe.
- **Prüfen:** `ls -l /tmp/kustomize` zeigt eine ausführbare Datei (`x` in den Rechten).

```bash
sudo install -o root -g root -m 0755 /tmp/kustomize /usr/local/bin/kustomize
```

- **Was:** kopiert das Programm nach `/usr/local/bin`, den Ordner für Programme, die nicht über apt oder snap installiert werden. Er steht bei allen Benutzern im `PATH`. `install` kopiert und setzt in einem Schritt Besitzer und Rechte: `-o root -g root` macht root zum Besitzer, `-m 0755` erlaubt allen das Lesen und Ausführen, nur root das Ändern.
- **Wo:** Terminal des Laborrechners, mit `sudo`.
- **Erwartet:** keine Ausgabe, eventuell vorher die Abfrage des Passworts.
- **Prüfen:** mit dem folgenden Befehl.

```bash
kustomize version
```

- **Was:** wie oben.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `v5.8.1`
- **Prüfen:** Die Version stimmt mit der Release-Seite überein. Meldet die Shell weiterhin, dass der Befehl nicht gefunden wurde, leert `hash -r` den Speicher der Shell für Programmpfade. Danach den Befehl wiederholen.

Beispiel (ruki-ws-02, 03.11.2026): Auf der Release-Seite stand für `kustomize_v5.8.1_linux_amd64.tar.gz` die Prüfsumme `sha256:029a7f0f4e1932c52a0476cf02…` (4,93 MB, veröffentlicht am 9. Februar). Die beiden `curl`-Befehle, `tar` und `install` gaben nichts aus. Die Prüfung ergab:

```
kustomize_v5.8.1_linux_amd64.tar.gz: OK
```

und `kustomize version` danach:

```
v5.8.1
```

## 5.3 Manifeste herunterladen

Die Manifeste werden aus dem Tag des gewählten Releases geladen, nicht aus `master`, damit die Installation wiederholbar bleibt. Welcher Tag aktuell ist, steht in den Release Notes auf GitHub unter *Latest* (am 03.11.2026: `26.03.1`, siehe [1.1](01-voraussetzungen.md#11-passende-versionen-bestimmen)).

```bash
git clone --branch 26.03.1 --depth 1 https://github.com/kubeflow/community-distribution.git ~/kubeflow-manifests
```

- **Was:** lädt das Repository der Kubeflow Community Distribution in den Ordner `~/kubeflow-manifests`. `--branch` nimmt einen Branch oder einen Tag, hier den Tag `26.03.1`. `--depth 1` lädt nur diesen einen Commit ohne die ganze Geschichte, das geht schneller und braucht weniger Platz. Für ein anderes Release `26.03.1` ersetzen.
- **Wo:** Terminal des Laborrechners, in einem beliebigen Ordner (der Zielpfad ist absolut).
- **Erwartet:** `Klone nach '…/kubeflow-manifests' …` (Englisch: `Cloning into '…'`), die Fortschrittszeilen von git und danach ein längerer Hinweis auf den Zustand *detached HEAD* (Deutsch: *losgelöster HEAD*). Der Hinweis ist kein Fehler: Ein Tag ist kein Branch, git steht deshalb direkt auf einem Commit.
- **Prüfen:** mit dem folgenden Befehl.

Beispiel (ruki-ws-02, 03.11.2026):

```
Klone nach '/home/ruki-labor/kubeflow-manifests' …
remote: Enumerating objects: 1856, done.
remote: Counting objects: 100% (1856/1856), done.
remote: Compressing objects: 100% (1411/1411), done.
remote: Total 1856 (delta 493), reused 1148 (delta 326), pack-reused 0 (from 0)
Empfange Objekte: 100% (1856/1856), 2.51 MiB | 8.60 MiB/s, fertig.
Löse Unterschiede auf: 100% (493/493), fertig.
Hinweis: Wechsle zu 'f09f3eeaa25cc852665f460497a42b7fc68639ac'.

Sie befinden sich im Zustand eines 'losgelösten HEAD'. Sie können sich
umschauen, experimentelle Änderungen vornehmen und diese committen, und
Sie können alle möglichen Commits, die Sie in diesem Zustand machen,
ohne Auswirkungen auf irgendeinen Branch verwerfen, indem Sie zu einem
anderen Branch wechseln.

Wenn Sie einen neuen Branch erstellen möchten, um Ihre erstellten Commits
zu behalten, können Sie das (jetzt oder später) durch Nutzung von
'switch' mit der Option -c tun. Beispiel:

  git switch -c <neuer-Branchname>

Oder um diese Operation rückgängig zu machen:
  git switch -

Sie können diesen Hinweis ausschalten, indem Sie die Konfigurationsvariable
'advice.detachedHead' auf 'false' setzen.
```

```bash
git -C ~/kubeflow-manifests describe --tags
```

- **Was:** zeigt den Tag, auf dem das Repository steht. `-C <Ordner>` führt git in diesem Ordner aus, ohne vorher mit `cd` hineinzuwechseln.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** der Name des Tags, hier `26.03.1`.
- **Prüfen:** Steht dort ein anderer Name, ist nicht das gewünschte Release geladen.

Beispiel (ruki-ws-02, 03.11.2026): `26.03.1`

**README des Releases lesen.** Maßgeblich für die Installation ist das README im geladenen Tag, nicht das auf der Startseite des Repositorys (dort steht der Stand von `master`). Die wichtigsten Stellen findet:

```bash
grep -n -E "RAM|CPU|StorageClass|12341234|port-forward|kustomize build example|Kubernetes 1\." ~/kubeflow-manifests/README.md
```

- **Was:** sucht im README Zeilen zu Ressourcen, StorageClass, Standardpasswort, Zugriff auf das Dashboard und zum Installationsbefehl. `-n` zeigt die Zeilennummer, `-E` erlaubt erweiterte reguläre Ausdrücke, in denen `|` „oder“ bedeutet. `1\.` sucht einen echten Punkt.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Liste von Zeilen im Format `Nummer:Text`.
- **Prüfen:** Der Installationsbefehl mit `while ! kustomize build example` ist dabei. Weicht er vom Befehl in 5.5 ab, gilt der Befehl aus dem README. Den Abschnitt rund um den Befehl liest man mit `sed -n '<von>,<bis>p' ~/kubeflow-manifests/README.md`.

Beispiel (ruki-ws-02, 03.11.2026):

```
61:This repository periodically synchronizes all official Kubeflow components from the respective upstream repositories. The following matrix shows the git version included for each component along with the rough resource requirements for each Kubeflow component, calculated as the maximum of actual usage and configured requests for CPU/memory as well as storage requirements from PVCs:
63:| Component | Local Manifests Path | Upstream Revision | CPU (millicores) | Memory (Mi) | PVC Storage (GB) |
96::warning: In both options, we use a default email (`user@example.com`) and password (`12341234`). For any production Kubeflow deployment, you should change the default password by following [the relevant section](#change-default-user-password).
118:- 16+ GB of RAM recommended.
119:- 8 CPU cores recommended.
120:- You can exclude components from the `example/kustomization.yaml` to fit Kubeflow into 4-8 GB of memory and 2-4 CPU cores.
149:while ! kustomize build example | kubectl apply --server-side --force-conflicts -f -; do echo "Retrying to apply resources"; sleep 20; done
238:kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
264:Dex is an OpenID Connect (OIDC) identity provider with multiple authentication backends. In this default installation, it includes a static user with the email `user@example.com`. By default, the user's password is `12341234`. For any production Kubeflow deployment, you should change the default password by following [the relevant section](#change-default-user-password).
448:The default way of accessing Kubeflow is via port-forwarding. This enables you to get started quickly without imposing any requirements on your environment. Run the following to port-forward Istio's Ingress-Gateway to local port `8080`:
451:kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
457:2. Log in with the default user's credentials. The default email address is `user@example.com`, and the default password is `12341234`.
688:  **A:** Istio CNI provides better security by eliminating the need for privileged init containers, making it more compatible with Pod Security Standards (PSS). It also enables native sidecars support introduced in Kubernetes 1.28, which helps address issues with init containers and application lifecycle management.
```

Das README des Tags `26.03.1` erwähnt die Default StorageClass nicht (kein Treffer für `StorageClass`). Warum sie trotzdem nötig ist, steht in [4.1](04-local-path-provisioner.md#41-warum-kubeflow-eine-default-storageclass-braucht).

## 5.4 Kernel-Parameter für inotify

Viele Programme in den Pods von Kubeflow überwachen Dateien mit inotify, einer Funktion des Linux-Kernels. Der Kernel begrenzt, wie viele inotify-Instanzen und überwachte Dateien ein Benutzer haben darf. Das README nennt unter den Voraussetzungen höhere Werte, *„to support many pods“*: `fs.inotify.max_user_instances=2280` und `fs.inotify.max_user_watches=1255360`.

```bash
sysctl fs.inotify.max_user_instances fs.inotify.max_user_watches
```

- **Was:** liest die beiden Kernel-Parameter. Ohne `-w` und ohne `=` ändert `sysctl` nichts.
- **Wo:** Terminal des Laborrechners, ohne `sudo`.
- **Erwartet:** zwei Zeilen der Form `fs.inotify.max_user_instances = <Zahl>` und `fs.inotify.max_user_watches = <Zahl>`.
- **Prüfen:** Je nach Ergebnis geht es mit einem der zwei Fälle weiter.

| Fall | Ergebnis | Vorgehen |
|---|---|---|
| A | Beide Werte mindestens so hoch wie im README | Weiter mit 5.5. *Noch nicht getestet.* |
| B | Mindestens ein Wert niedriger | Werte wie unten dauerhaft erhöhen. Getestet auf ruki-ws-02 am 03.11.2026. |

Beispiel (ruki-ws-02, 03.11.2026):

```
fs.inotify.max_user_instances = 128
fs.inotify.max_user_watches = 65536
```

Das README setzt die Werte mit `sudo sysctl …`. So gelten sie aber nur bis zum nächsten Neustart. Damit sie bleiben, schreiben wir sie in eine Datei in `/etc/sysctl.d/`. Diese Dateien wendet das System bei jedem Start an. Sie werden nach ihrem Namen sortiert, und bei gleichen Parametern gilt die Datei, die in dieser Sortierung zuletzt kommt (sysctl.d(5), man7.org). Das Präfix `99-` sorgt dafür, dass unsere Datei spät gelesen wird.

```bash
printf 'fs.inotify.max_user_instances = 2280\nfs.inotify.max_user_watches = 1255360\n' | sudo tee /etc/sysctl.d/99-kubeflow-inotify.conf
```

- **Was:** schreibt die beiden Zeilen in die neue Datei `/etc/sysctl.d/99-kubeflow-inotify.conf`. `printf` erzeugt den Text, `\n` ist ein Zeilenumbruch. `sudo tee <Datei>` schreibt den Text mit root-Rechten in die Datei und gibt ihn zusätzlich aus. Ein einfaches `sudo printf … > Datei` würde nicht funktionieren, weil die Umleitung `>` von der eigenen Shell ohne root-Rechte ausgeführt wird. `tee` überschreibt eine vorhandene Datei gleichen Namens. Die Werte gelten erst nach dem nächsten Befehl.
- **Wo:** Terminal des Laborrechners, mit `sudo`.
- **Erwartet:** die beiden Zeilen, ausgegeben von `tee`.
- **Prüfen:** `cat /etc/sysctl.d/99-kubeflow-inotify.conf` zeigt genau diese zwei Zeilen.

Beispiel (ruki-ws-02, 03.11.2026):

```
fs.inotify.max_user_instances = 2280
fs.inotify.max_user_watches = 1255360
```

```bash
sudo sysctl -p /etc/sysctl.d/99-kubeflow-inotify.conf
```

- **Was:** liest die angegebene Datei und setzt ihre Parameter sofort, ohne Neustart. `-p <Datei>` (lang: `--load`) lädt nur diese Datei. `sudo sysctl --system` würde alle Konfigurationsdateien neu laden und eine lange Liste ausgeben.
- **Wo:** Terminal des Laborrechners, mit `sudo`.
- **Erwartet:** die beiden Zeilen aus der Datei.
- **Prüfen:** Der Lesebefehl vom Anfang dieses Abschnitts zeigt jetzt `2280` und `1255360`.

Beispiel (ruki-ws-02, 03.11.2026):

```
fs.inotify.max_user_instances = 2280
fs.inotify.max_user_watches = 1255360
```

## 5.5 Alle Komponenten installieren

Der Befehl aus dem README verwendet den relativen Pfad `example`. Deshalb zuerst in den Ordner der Manifeste wechseln:

```bash
cd ~/kubeflow-manifests
```

- **Was:** wechselt in das Repository aus 5.3. Dort liegt der Ordner `example/` mit der Datei `kustomization.yaml`, die alle Komponenten aufzählt.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe, die Eingabezeile endet auf `~/kubeflow-manifests$`.
- **Prüfen:** `ls example` zeigt unter anderem `kustomization.yaml`.

```bash
while ! kustomize build example | kubectl apply --server-side --force-conflicts -f -; do echo "Retrying to apply resources"; sleep 20; done
```

- **Was:** Der Befehl ist unverändert aus dem README übernommen (Abschnitt *Install with a Single Command*).
  - `kustomize build example` setzt alle in `example/kustomization.yaml` genannten Manifeste zu einem einzigen YAML-Strom zusammen: Istio, cert-manager, Dex, oauth2-proxy, Pipelines, Katib, KServe, Notebooks, Trainer und weitere.
  - `| kubectl apply … -f -` schickt diesen Strom an den Cluster. `-` nach `-f` steht für die Standardeingabe.
  - `--server-side` lässt den API-Server die Änderungen zusammenführen, nicht kubectl. Einige Objekte, vor allem große CustomResourceDefinitions (CRDs), wären für die Annotation zu groß, die das klassische `kubectl apply` an jedem Objekt speichert.
  - `--force-conflicts` übernimmt Felder, die bereits ein anderer Verwalter gesetzt hat. Das braucht man, wenn der Befehl wiederholt wird.
  - `while ! … ; do …; sleep 20; done` wiederholt alles alle 20 Sekunden, solange `kubectl apply` einen Fehler meldet. Laut README ist das zu erwarten: Manche Objekte lassen sich erst anlegen, wenn ihre CRD bzw. der Pod hinter einem Webhook bereit ist (README, Abschnitt *NOTE*).
- **Wo:** Terminal des Laborrechners, im Ordner `~/kubeflow-manifests`. Kein `sudo`: kubectl verwendet die kubeconfig des Benutzers.
- **Erwartet:**
  - viele Zeilen der Form `<Typ>/<Name> serverside-applied`;
  - in den ersten Durchläufen Fehler und danach `Retrying to apply resources`;
  - Warnungen von Kustomize zur alten Syntax der Manifeste, z. B. `# Warning: 'commonLabels' is deprecated. …`. Sie ändern nichts an der Installation.
  - Nach einigen Durchläufen endet die Schleife von selbst, und die Eingabezeile erscheint wieder.
- **Prüfen:** Der letzte Durchlauf enthält keine Zeile mit `Error`. Die Schleife während der Fehler nicht abbrechen. Wird sie doch mit Strg+C beendet, kann man sie ohne Risiko neu starten: Sie wendet dieselben Objekte noch einmal an.

Diese Fehler traten in den ersten Durchläufen auf und sind normal:

| Fehler | Ursache |
|---|---|
| `failed calling webhook "…": … connect: connection refused` | Das Objekt muss von einem Webhook geprüft werden, dessen Pod noch nicht läuft, weil z. B. sein Image noch geladen wird. Gesehen für KServe, Trainer und Knative. |
| `namespaces "kubeflow-user-example-com" not found` | Den Namespace des Standardbenutzers legt der Profile Controller aus dem Profil `kubeflow-user-example-com` an, sobald sein eigener Pod läuft. |

Beispiel (ruki-ws-02, 03.11.2026): Die Schleife meldete dreimal `Retrying to apply resources` und endete dann von selbst. Je ein Fehler aus den ersten Durchläufen:

```
Error from server (InternalError): Internal error occurred: failed calling webhook "validator.clustertrainingruntime.trainer.kubeflow.org": failed to call webhook: Post "https://kubeflow-trainer-controller-manager.kubeflow-system.svc:443/validate-trainer-kubeflow-org-v1alpha1-clustertrainingruntime?timeout=10s": dial tcp 10.103.251.239:443: connect: connection refused
Error from server (InternalError): Internal error occurred: failed calling webhook "clusterservingruntime.kserve-webhook-server.validator": failed to call webhook: Post "https://kserve-webhook-server-service.kubeflow.svc:443/validate-serving-kserve-io-v1alpha1-clusterservingruntime?timeout=10s": dial tcp 10.97.51.33:443: connect: connection refused
Error from server (InternalError): Internal error occurred: failed calling webhook "webhook.serving.knative.dev": failed to call webhook: Post "https://webhook.knative-serving.svc:443/?timeout=10s": dial tcp 10.103.242.25:443: connect: connection refused
Error from server (NotFound): namespaces "kubeflow-user-example-com" not found
```

Der letzte Durchlauf enthielt 727 Zeilen `serverside-applied` und keinen Fehler (eigene Messung auf ruki-ws-02, 03.11.2026). Seine letzten Zeilen:

```
validatingwebhookconfiguration.admissionregistration.k8s.io/spark-operator-webhook serverside-applied
validatingwebhookconfiguration.admissionregistration.k8s.io/trainedmodel.serving.kserve.io serverside-applied
validatingwebhookconfiguration.admissionregistration.k8s.io/validation.webhook.serving.knative.dev serverside-applied
validatingwebhookconfiguration.admissionregistration.k8s.io/validator.trainer.kubeflow.org serverside-applied
```

**Pull-Limits von Docker Hub.** Einige Images kommen von Docker Hub, das anonyme Downloads begrenzt. Für den Fall, dass Pods deshalb in `ImagePullBackOff` hängen, beschreibt das README im Abschnitt *Install with a Single Command* ein Secret `regcred` mit eigenen Zugangsdaten. Auf ruki-ws-02 war das am 03.11.2026 nicht nötig.

## 5.6 Pods prüfen

Wenn die Schleife fertig ist, sind alle Objekte angelegt. Die Pods brauchen danach aber noch Zeit, um ihre Images zu laden und zu starten. Erst wenn alle bereit sind, sollte man sich am Dashboard anmelden (README, Abschnitt *Connect to Your Kubeflow Cluster*).

```bash
kubectl get pods -A --field-selector=status.phase!=Running
```

- **Was:** listet die Pods aller Namespaces (`-A`), deren Phase nicht `Running` ist. `--field-selector` filtert auf dem API-Server nach dem Feld `status.phase`. So sieht man Pods, die noch warten (`Pending`, z. B. `ContainerCreating`, `Init:0/1`) oder fehlgeschlagen sind, und abgeschlossene Jobs (`Completed`).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `No resources found` oder nur Pods mit `Completed`.
- **Prüfen:** Pods in `ContainerCreating` oder `Init:…`: ein bis zwei Minuten warten und den Befehl wiederholen. Pods, die mehrere Minuten in `Pending`, `ImagePullBackOff`, `ErrImagePull` oder `CrashLoopBackOff` bleiben: `kubectl describe pod <name> -n <namespace> | tail -20` zeigt die Ursache.

Beispiel (ruki-ws-02, 03.11.2026, wenige Minuten nach dem Ende der Schleife): `No resources found`

Ein Pod in `CrashLoopBackOff` hat trotzdem die Phase `Running`, und ein Pod in `Running` kann Container haben, die noch nicht bereit sind. Deshalb zusätzlich die Spalte `READY` prüfen:

```bash
kubectl get pods -A --no-headers | awk '{split($3,a,"/"); if (a[1]!=a[2]) print}'
```

- **Was:** `kubectl get pods -A --no-headers` listet alle Pods ohne Kopfzeile. Die dritte Spalte ist `READY` im Format `bereit/gesamt`, z. B. `2/2`. `awk` bearbeitet jede Zeile: `split($3,a,"/")` teilt die dritte Spalte am `/`, `a[1]` ist die Zahl der bereiten Container, `a[2]` die Gesamtzahl. `if (a[1]!=a[2]) print` gibt die Zeile nur aus, wenn beide Zahlen verschieden sind.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** keine Ausgabe, d. h. bei allen Pods sind alle Container bereit.
- **Prüfen:** Erscheinen Zeilen, auf `STATUS` und `RESTARTS` achten. Kurz nach der Installation können sich Pods mit wenigen Neustarts noch fangen: eine Minute warten und wiederholen. Bleiben Zeilen stehen, wie oben mit `kubectl describe pod` die Ursache suchen.

Beispiel (ruki-ws-02, 03.11.2026): keine Ausgabe.

Alternativ nennt das README die Namespaces, deren Pods man einzeln prüfen kann: `cert-manager`, `istio-system`, `auth`, `oauth2-proxy`, `knative-serving`, `kubeflow` und `kubeflow-user-example-com`, jeweils mit `kubectl get pods -n <namespace>`.

Weiter: [6. Zugriff auf das Dashboard](06-dashboard-zugriff.md) | [Übersicht](../README.md)
