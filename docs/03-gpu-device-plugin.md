Zurück: [2. Kubernetes mit kubeadm](02-kubernetes-kubeadm.md) | [Übersicht](../README.md)

# 3. GPU-Check und NVIDIA Device Plugin

Aufbau der Befehle (Was, Wo, Erwartet, Prüfen) und Hinweise zur Sprache der Ausgaben: siehe [Übersicht](../README.md#aufbau-jedes-befehls). Quellen: siehe [Übersicht](../README.md#quellen).

Damit ein Pod die Grafikkarte nutzen kann, müssen vier Teile zusammenspielen: der NVIDIA-Treiber auf dem Rechner, das NVIDIA Container Toolkit, eine containerd-Konfiguration, die das Toolkit verwendet, und das NVIDIA Device Plugin in Kubernetes. Erst das Device Plugin meldet dem Cluster die Ressource `nvidia.com/gpu`, die ein Pod dann anfordern kann (Schedule GPUs, kubernetes.io). Die Abschnitte [3.1](#31-nvidia-treiber-prüfen) bis [3.3](#33-containerd-für-nvidia-konfigurieren) prüfen bzw. richten die Teile auf dem Rechner ein, [3.4](#34-nvidia-device-plugin-installieren) installiert das Device Plugin, [3.5](#35-test-mit-einem-gpu-pod) testet alles mit einem Pod.

Laut README des Device Plugins müssen auf jedem GPU-Knoten vorhanden sein: ein NVIDIA-Treiber, das NVIDIA Container Toolkit ab Version 1.7.0 und `nvidia-container-runtime` als Standard-Runtime auf unterster Ebene (https://github.com/NVIDIA/k8s-device-plugin, Abschnitt *Prerequisites*).

## 3.1 NVIDIA-Treiber prüfen

```bash
nvidia-smi
```

- **Was:** fragt den NVIDIA-Treiber ab und zeigt Grafikkarte, Speicherbelegung und die Prozesse auf der GPU. In der Kopfzeile stehen zwei Versionen: `Driver Version` ist der installierte Treiber, `CUDA Version` ist die höchste CUDA-Version, die dieser Treiber unterstützt. Diese CUDA-Version muss also nicht installiert sein (CUDA Toolkit, Driver, and Architecture Matrix, NVIDIA).
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Tabelle mit `NVIDIA-SMI … Driver Version: … CUDA Version: …` und einer Zeile pro GPU, oder eine der Fehlermeldungen aus der Tabelle unten.
- **Prüfen:** Grafikkarte und Treiberversion werden angezeigt. Je nach Ergebnis geht es mit einem der drei Fälle weiter.

| Fall | Ergebnis von `nvidia-smi` | Vorgehen |
|---|---|---|
| A | Tabelle mit GPU und Treiberversion | Weiter mit 3.2. Getestet auf ruki-ws-02 am 03.11.2026. |
| B | `Befehl nicht gefunden` bzw. `command not found` | Kein Treiber installiert. Hinweise zur Installation: Installing the NVIDIA Container Toolkit, Abschnitt *Prerequisites*. *Noch nicht getestet.* |
| C | `NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver` | Treiber installiert, aber das Kernel-Modul ist nicht geladen, häufig nach einem Kernel-Update. *Noch nicht getestet.* |

<!-- TODO: Fall B und C auf einem Rechner ohne bzw. mit defektem Treiber durchspielen -->

Beispiel (ruki-ws-02, 03.11.2026, Auszug):

```
| NVIDIA-SMI 595.84                 Driver Version: 595.84         CUDA Version: 13.2     |
...
|   0  NVIDIA GeForce RTX 5090        Off |   00000000:01:00.0  On |                  N/A |
|  0%   44C    P8             27W /  600W |     697MiB /  32607MiB |      1%      Default |
...
|    0   N/A  N/A            3241      G   /usr/lib/xorg/Xorg                      280MiB |
|    0   N/A  N/A            3463      G   /usr/bin/gnome-shell                     79MiB |
```

Auf ruki-ws-02 steuert dieselbe GPU auch den Bildschirm (`Disp.A` = `On`). Die grafische Oberfläche belegt dort rund 700 MiB, die für Trainings-Pods nicht zur Verfügung stehen (eigene Messung, 03.11.2026).

GPUs der Blackwell-Generation (z. B. RTX 5090) werden laut NVIDIA erst ab CUDA 12.8 unterstützt (CUDA Toolkit, Driver, and Architecture Matrix). Images für das eigene Training sollten daher auf CUDA 12.8 oder neuer aufbauen.

## 3.2 NVIDIA Container Toolkit

Das Container Toolkit sorgt dafür, dass ein Container den Treiber und die GPU des Rechners sieht. Zuerst wird geprüft, ob es schon installiert ist.

```bash
dpkg -l | grep -E 'nvidia-container|libnvidia-container'
```

- **Was:** `dpkg -l` listet alle Debian-Pakete mit Status und Version. `grep -E` behält nur Zeilen mit `nvidia-container` oder `libnvidia-container` (`|` bedeutet „oder“). Damit werden die vier Pakete des Toolkits erfasst.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** bis zu vier Zeilen (`nvidia-container-toolkit`, `nvidia-container-toolkit-base`, `libnvidia-container-tools`, `libnvidia-container1`), oder keine Ausgabe.
- **Prüfen:** Die ersten beiden Buchstaben jeder Zeile geben den Zustand an: `ii` heißt installiert, `hi` heißt installiert und gegen Updates gesperrt (`hold`, wie kubelet in [2.5](02-kubernetes-kubeadm.md#25-kubeadm-kubelet-und-kubectl-installieren)).

Beispiel (ruki-ws-02, 03.11.2026):

```
hi  libnvidia-container-tools                        1.19.1-1                                         amd64        NVIDIA container runtime library (command-line tools)
hi  libnvidia-container1:amd64                       1.19.1-1                                         amd64        NVIDIA container runtime library
hi  nvidia-container-toolkit                         1.19.1-1                                         amd64        NVIDIA Container toolkit
hi  nvidia-container-toolkit-base                    1.19.1-1                                         amd64        NVIDIA Container Toolkit Base
```

| Fall | Ergebnis | Vorgehen |
|---|---|---|
| A | Keine Ausgabe | Toolkit nach der Anleitung von NVIDIA installieren (Installing the NVIDIA Container Toolkit, Abschnitt *With apt: Ubuntu, Debian*): Voraussetzungen installieren, Paketquelle einrichten, `sudo apt-get update`, dann die vier Pakete in der dort genannten Version installieren (am 03.11.2026: `1.20.1-1`). Danach weiter mit 3.3. *Noch nicht getestet.* |
| B | Vier Zeilen mit `ii` oder `hi` | Nichts installieren, weiter mit 3.3. Getestet auf ruki-ws-02 am 03.11.2026 (Version 1.19.1-1, gesperrt). |

Die installierte Version 1.19.1-1 auf ruki-ws-02 ist älter als die am 03.11.2026 dokumentierte 1.20.1-1. Sie erfüllt die Mindestversion 1.7.0 des Device Plugins und wurde in [3.5](#35-test-mit-einem-gpu-pod) erfolgreich getestet. Ein Update ist daher nicht nötig und bleibt durch `hold` eine bewusste Entscheidung.

## 3.3 containerd für NVIDIA konfigurieren

Die Standardkonfiguration von containerd bindet alle Dateien aus `/etc/containerd/conf.d/` ein (siehe [2.4](02-kubernetes-kubeadm.md#24-container-runtime-containerd)). Dort legt das Toolkit seine Konfiguration ab: `nvidia-ctk runtime configure --runtime=containerd` erzeugt die Datei `/etc/containerd/conf.d/99-nvidia.toml` (Installing the NVIDIA Container Toolkit, Abschnitt *Configuring containerd (for Kubernetes)*). Zuerst wird nachgesehen, ob diese Datei schon existiert.

```bash
ls -l /etc/containerd/conf.d/
```

- **Was:** listet den Inhalt des Ordners. `-l` zeigt zusätzlich Rechte, Besitzer, Größe und Änderungsdatum. Am Datum erkennt man, ob eine Datei von einer früheren Installation stammt.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile mit `99-nvidia.toml`, oder `insgesamt 0` bzw. `total 0` (Ordner leer), oder eine Meldung, dass der Ordner nicht existiert.
- **Prüfen:** Je nach Ergebnis geht es mit einem der beiden Fälle weiter.

Beispiel (ruki-ws-02, 03.11.2026):

```
insgesamt 4
-rw-r--r-- 1 root root 3062 Aug  5 16:17 99-nvidia.toml
```

| Fall | Ergebnis | Vorgehen |
|---|---|---|
| A | `99-nvidia.toml` vorhanden | Nichts erzeugen, nur den Inhalt prüfen (unten). Stammt die Datei von einer früheren Installation, muss sie zum aktuellen containerd passen. Getestet auf ruki-ws-02 am 03.11.2026. |
| B | Datei fehlt | `sudo nvidia-ctk runtime configure --runtime=containerd`, danach `sudo systemctl restart containerd` (Installing the NVIDIA Container Toolkit, Abschnitt *Configuring containerd (for Kubernetes)*). Anschließend wie in Fall A prüfen, ob `nvidia` die Standard-Runtime ist. *Noch nicht getestet.* |

<!-- TODO: Fall B testen; prüfen, ob nvidia-ctk nvidia ohne weitere Option als Standard-Runtime einträgt (nvidia-ctk runtime configure --help) -->

```bash
cat /etc/containerd/conf.d/99-nvidia.toml
```

- **Was:** zeigt den Inhalt der Datei.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine TOML-Konfiguration. Wichtig sind vier Stellen: `version = 3` (Format von containerd 2.x), Abschnitte unter `[plugins."io.containerd.cri.v1.runtime"]`, ein Block `runtimes.nvidia` mit `BinaryName = "/usr/bin/nvidia-container-runtime"` und die Zeile `default_runtime_name = "nvidia"`.
- **Prüfen:** Steht dort `version = 2` oder `[plugins."io.containerd.grpc.v1.cri"]`, stammt die Datei aus der Zeit von containerd 1.x. Dann die Datei sichern und wie in Fall B neu erzeugen.

Beispiel (ruki-ws-02, 03.11.2026, Auszug):

```
version = 3
...
    [plugins."io.containerd.cri.v1.runtime".containerd]
      default_runtime_name = "nvidia"
...
          [plugins."io.containerd.cri.v1.runtime".containerd.runtimes.nvidia.options]
            BinaryName = "/usr/bin/nvidia-container-runtime"
...
            SystemdCgroup = true
```

In `/etc/containerd/config.toml` steht weiterhin `default_runtime_name = 'runc'` (siehe [2.4](02-kubernetes-kubeadm.md#24-container-runtime-containerd)). Welcher Wert gilt, zeigt containerd selbst: Es fügt `config.toml` und die eingebundenen Dateien zu einer Gesamtkonfiguration zusammen.

```bash
sudo containerd config dump | grep -n -E 'default_runtime_name|BinaryName|SystemdCgroup|bin_dirs'
```

- **Was:** `containerd config dump` gibt die zusammengeführte Konfiguration aus, `grep` behält nur die Standard-Runtime, die Programme der Runtimes, den cgroup-Treiber und den Pfad der CNI-Plugins, die Flannel braucht.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `default_runtime_name = 'nvidia'`, eine Zeile `BinaryName = '/usr/bin/nvidia-container-runtime'`, `SystemdCgroup = true` und `bin_dirs = ['/opt/cni/bin']`.
- **Prüfen:** Steht bei `default_runtime_name` noch `runc`, ist die Voraussetzung des Device Plugins nicht erfüllt.

Beispiel (ruki-ws-02, 03.11.2026):

```
80:      default_runtime_name = 'nvidia'
101:            BinaryName = '/usr/bin/nvidia-container-runtime'
109:            SystemdCgroup = true
127:            BinaryName = ''
135:            SystemdCgroup = true
139:      bin_dirs = ['/opt/cni/bin']
```

Auf ruki-ws-02 war kein Neustart von containerd nötig: Die Datei `99-nvidia.toml` existierte schon vor dem Neustart in 2.4.

## 3.4 NVIDIA Device Plugin installieren

Das Device Plugin läuft als DaemonSet, also mit einem Pod pro Knoten. Dieser Pod erkennt die GPUs und meldet sie dem kubelet als Ressource `nvidia.com/gpu`. Vorher zeigt der Knoten diese Ressource nicht:

```bash
kubectl get node <hostname> -o jsonpath='{.status.allocatable}{"\n"}'
```

- **Was:** zeigt nur das Feld `allocatable` des Knotens, also die Ressourcen, die der Scheduler an Pods vergeben kann. `{"\n"}` hängt einen Zeilenumbruch an.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine JSON-Zeile mit `cpu`, `memory`, `pods` und weiteren Einträgen, ohne `nvidia.com/gpu`.
- **Prüfen:** Steht `nvidia.com/gpu` schon dort, läuft bereits ein Device Plugin: `kubectl get pods -A` zeigt es.

Beispiel (ruki-ws-02, 03.11.2026): `{"cpu":"32","ephemeral-storage":"1770063008091","hugepages-1Gi":"0","hugepages-2Mi":"0","memory":"63204992Ki","pods":"110"}`

**Version wählen.** Das README des Device Plugins nennt im Abschnitt *Quick Start* eine feste Version (am 03.11.2026: `v0.17.1`), die nicht immer die neueste ist. Die aktuelle Version steht auf der Releases-Seite unter *Latest* (am 03.11.2026: `v0.20.0`, https://github.com/NVIDIA/k8s-device-plugin/releases). Wir verwenden die Version unter *Latest* und lesen das Manifest vor der Installation:

```bash
curl -fsSL https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.20.0/deployments/static/nvidia-device-plugin.yml
```

- **Was:** lädt das Manifest aus dem Tag `v0.20.0` des offiziellen Repositorys und zeigt es an, ohne etwas zu installieren. `-f` bricht bei einem HTTP-Fehler ab (z. B. 404), `-s` blendet den Fortschritt aus, `-S` zeigt trotzdem Fehlermeldungen, `-L` folgt Weiterleitungen. Für eine andere Version `v0.20.0` ersetzen.
- **Wo:** Terminal des Laborrechners (braucht Internetzugang).
- **Erwartet:** ein YAML-Dokument mit Apache-2.0-Lizenzkopf, `kind: DaemonSet`, `namespace: kube-system` und einer Zeile `image: nvcr.io/nvidia/k8s-device-plugin:v0.20.0`.
- **Prüfen:** Die Version in der Zeile `image:` stimmt mit der gewählten Version überein. `curl: (22) The requested URL returned error: 404` bedeutet, dass der Pfad in diesem Tag nicht existiert.

Beispiel (ruki-ws-02, 03.11.2026, Auszug):

```
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
...
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
...
      - image: nvcr.io/nvidia/k8s-device-plugin:v0.20.0
```

Das Manifest toleriert nur den Taint `nvidia.com/gpu`. Trägt der Knoten noch den Taint des Control Plane, bleibt der Pod auf `Pending`. Deshalb vorher prüfen, ob [2.9](02-kubernetes-kubeadm.md#29-control-plane-für-normale-pods-freigeben) ausgeführt wurde:

```bash
kubectl get node <hostname> -o jsonpath='{.spec.taints}{"\n"}'
```

- **Was:** zeigt nur die Taints des Knotens.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine leere Zeile.
- **Prüfen:** Erscheint `[{"effect":"NoSchedule","key":"node-role.kubernetes.io/control-plane"}]`, zuerst [2.9](02-kubernetes-kubeadm.md#29-control-plane-für-normale-pods-freigeben) ausführen.

Beispiel (ruki-ws-02, 03.11.2026): leere Zeile.

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.20.0/deployments/static/nvidia-device-plugin.yml
```

- **Was:** legt das DaemonSet aus dem eben gelesenen Manifest im Cluster an. `create` bricht mit `AlreadyExists` ab, wenn es das DaemonSet schon gibt. So fällt eine doppelte Installation auf.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `daemonset.apps/nvidia-device-plugin-daemonset created`.
- **Prüfen:** mit den nächsten drei Befehlen.

Beispiel (ruki-ws-02, 03.11.2026): `daemonset.apps/nvidia-device-plugin-daemonset created`

```bash
kubectl -n kube-system get pods -l name=nvidia-device-plugin-ds -o wide
```

- **Was:** listet die Pods des DaemonSets. `-l` filtert nach dem Label aus dem Manifest, `-o wide` zeigt zusätzlich den Knoten.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** ein Pod pro Knoten mit `1/1` und `Running`. Direkt nach dem Anlegen ist für einige Sekunden `ContainerCreating` normal (das Image wird geladen).
- **Prüfen:** Bei `CrashLoopBackOff` oder `Error` die Logs lesen (nächster Befehl).

Beispiel (ruki-ws-02, 03.11.2026):

```
NAME                                   READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
nvidia-device-plugin-daemonset-xp62b   1/1     Running   0          23s   10.244.0.4   ruki-ws-02   <none>           <none>
```

```bash
kubectl -n kube-system logs <pod-name>
```

- **Was:** zeigt die Ausgabe des Device Plugins. `<pod-name>` ist der Name aus dem vorigen Befehl.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** Version, geladene Konfiguration (JSON) und am Ende die Registrierung beim kubelet.
- **Prüfen:** Die Zeilen `Detected platform: nvml` und `Registered device plugin for 'nvidia.com/gpu' with Kubelet` erscheinen. `nvml` heißt, dass das Plugin den Treiber über die NVIDIA-Bibliothek NVML erreicht.

Beispiel (ruki-ws-02, 03.11.2026, Auszug):

```
I1103 12:51:07.289507       1 main.go:269] "Starting NVIDIA Device Plugin" version=<
	1b826acc
...
I1103 12:51:07.301697       1 plugin-manager.go:101] Detected platform: nvml
I1103 12:51:07.301712       1 plugin-manager.go:50] Using device discovery strategy: nvml
I1103 12:51:07.311446       1 server.go:198] Starting GRPC server for 'nvidia.com/gpu'
I1103 12:51:07.311791       1 server.go:142] Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
I1103 12:51:07.312569       1 server.go:149] Registered device plugin for 'nvidia.com/gpu' with Kubelet
```

Die Zeitstempel sind in UTC (12:51 UTC = 13:51 MEZ). `1b826acc` ist der Commit des Releases v0.20.0.

```bash
kubectl get node <hostname> -o jsonpath='{.status.allocatable}{"\n"}'
```

- **Was:** derselbe Befehl wie vor der Installation.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** dieselbe JSON-Zeile, jetzt mit `"nvidia.com/gpu":"1"` (bzw. der Anzahl der GPUs).
- **Prüfen:** Fehlt der Eintrag, einige Sekunden warten und den Befehl wiederholen: Der kubelet aktualisiert den Knotenstatus in Abständen.

Beispiel (ruki-ws-02, 03.11.2026): `{"cpu":"32","ephemeral-storage":"1770063008091","hugepages-1Gi":"0","hugepages-2Mi":"0","memory":"63204992Ki","nvidia.com/gpu":"1","pods":"110"}`

## 3.5 Test mit einem GPU-Pod

Der Test-Pod stammt aus dem README des Device Plugins (https://github.com/NVIDIA/k8s-device-plugin, NVIDIA, Apache-2.0). Er führt einmal das CUDA-Beispiel `vectorAdd` aus, das zwei Vektoren auf der GPU addiert. Eine GPU wird unter `limits` angefordert; Kubernetes übernimmt diesen Wert auch als `requests` (Schedule GPUs, kubernetes.io).

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  restartPolicy: Never
  containers:
    - name: cuda-container
      image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda12.5.0
      resources:
        limits:
          nvidia.com/gpu: 1
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
EOF
```

- **Was:** `<<EOF … EOF` (Here-Document) übergibt den YAML-Text an `cat`, das ihn über `|` an `kubectl apply -f -` weiterreicht. `-` heißt „Manifest von der Standardeingabe lesen“, es entsteht keine Datei. Der Pod landet im Namespace `default`. `restartPolicy: Never` sorgt dafür, dass er nach einem Durchlauf beendet bleibt.
- **Wo:** Terminal des Laborrechners. Den Block vollständig kopieren, von `cat` bis zum letzten `EOF`.
- **Erwartet:** `pod/gpu-pod created`.
- **Prüfen:** mit den nächsten beiden Befehlen, etwa 30 Sekunden später (das Image wird geladen).

Beispiel (ruki-ws-02, 03.11.2026): `pod/gpu-pod created`

```bash
kubectl logs gpu-pod
```

- **Was:** zeigt die Ausgabe des Programms `vectorAdd`.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** am Ende `Test PASSED` und `Done`. Läuft der Pod noch nicht, meldet kubectl, dass der Container noch auf den Start wartet; dann kurz warten und wiederholen.
- **Prüfen:** `Test PASSED` erscheint.

Beispiel (ruki-ws-02, 03.11.2026):

```
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

Das Image ist für CUDA 12.5 gebaut, also älter als die Blackwell-Unterstützung ([3.1](#31-nvidia-treiber-prüfen)). Auf der RTX 5090 lief es trotzdem fehlerfrei (eigene Messung, 03.11.2026).

```bash
kubectl get pod gpu-pod
```

- **Was:** zeigt den Status des Test-Pods.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `0/1`, `Completed`, `0` Neustarts. `0/1` ist hier richtig: Der Container hat seine Arbeit beendet.
- **Prüfen:** `STATUS` ist `Completed`, nicht `Error`.

Beispiel (ruki-ws-02, 03.11.2026):

```
NAME      READY   STATUS      RESTARTS   AGE
gpu-pod   0/1     Completed   0          31s
```

```bash
kubectl delete pod gpu-pod
```

- **Was:** löscht den Test-Pod. Das Image bleibt im Cache von containerd, das DaemonSet des Device Plugins bleibt unberührt.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile, die mit `pod "gpu-pod" deleted` beginnt.
- **Prüfen:** `kubectl get pod gpu-pod` meldet `NotFound`.

Beispiel (ruki-ws-02, 03.11.2026, kubectl 1.36.6): `pod "gpu-pod" deleted from default namespace`

[Übersicht](../README.md)
