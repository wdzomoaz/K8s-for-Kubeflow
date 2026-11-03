Zurück: [3. GPU-Check und NVIDIA Device Plugin](03-gpu-device-plugin.md) | [Übersicht](../README.md)

# 4. Speicher: Local Path Provisioner als Default StorageClass

Aufbau der Befehle (Was, Wo, Erwartet, Prüfen) und Hinweise zur Sprache der Ausgaben: siehe [Übersicht](../README.md#aufbau-jedes-befehls). Quellen: siehe [Übersicht](../README.md#quellen).

Viele Kubeflow-Komponenten speichern ihre Daten in PersistentVolumeClaims (PVC). Ein PVC beschreibt nur, wie viel Speicher gebraucht wird. Den eigentlichen Speicher, das PersistentVolume (PV), legt ein Provisioner an, und welcher Provisioner zuständig ist, bestimmt die StorageClass. Gibt ein PVC keine StorageClass an, setzt Kubernetes die als Standard markierte StorageClass ein (Storage Classes, kubernetes.io). Gibt es keine solche Klasse, bleibt der PVC im Zustand `Pending`, und der Pod, der ihn braucht, startet nicht.

Ein mit kubeadm aufgesetzter Cluster bringt keine StorageClass mit. Diese Anleitung verwendet den Local Path Provisioner von Rancher. Er legt für jeden PVC ein Verzeichnis auf der lokalen Festplatte des Knotens an, das reicht für einen einzelnen Laborrechner. Abschnitt [4.1](#41-warum-kubeflow-eine-default-storageclass-braucht) zeigt, wofür Kubeflow den Speicher braucht. [4.2](#42-vorhandene-storageclasses-prüfen) prüft, ob schon eine StorageClass existiert. [4.3](#43-local-path-provisioner-installieren) und [4.4](#44-local-path-als-default-markieren) installieren den Provisioner und markieren ihn als Standard. [4.5](#45-test-mit-pvc-und-pod) testet ihn, [4.6](#46-test-aufräumen) räumt den Test wieder ab, [4.7](#47-grenzen-des-local-path-provisioners) nennt die Grenzen dieser Lösung.

## 4.1 Warum Kubeflow eine Default StorageClass braucht

Die Installationsanleitung im Branch `v1.10-branch` der Kubeflow-Manifeste verlangt ausdrücklich einen Cluster *„with a default StorageClass“* (README, Abschnitt *Prerequisites*). Das README im Branch `release-26.03` nennt diese Voraussetzung nicht mehr ausdrücklich. Die PVCs in den Manifesten von 26.03 geben aber weiterhin keinen `storageClassName` an und sind damit auf eine Default StorageClass angewiesen. Einige Beispiele (Branch `release-26.03`, Commit `d68ebc7`, geprüft am 03.11.2026):

| PVC | Komponente | Größe | Datei |
|---|---|---|---|
| `mysql-pv-claim` | Pipelines (MySQL) | 20Gi | `applications/pipeline/upstream/third-party/mysql/base/mysql-pv-claim.yaml` |
| `seaweedfs-pvc` | Pipelines (Objektspeicher SeaweedFS) | 20Gi | `applications/pipeline/upstream/third-party/seaweedfs/base/seaweedfs/seaweedfs-pvc.yaml` |
| `katib-mysql` | Katib (MySQL) | 10Gi | `applications/katib/upstream/components/mysql/pvc.yaml` |
| `metadata-mysql` | ML Metadata | 10Gi | `applications/pipeline/upstream/base/metadata/overlays/db/metadata-db-pvc.yaml` |

Dazu kommen die Workspaces der Notebooks: Laut `spawner_ui_config.yaml` bekommt jedes neue Notebook einen PVC `{notebook-name}-workspace` mit 5Gi. Die Zeile `storageClassName` ist dort auskommentiert, es gilt also ebenfalls die Default StorageClass (`applications/jupyter/jupyter-web-app/upstream/base/configs/spawner_ui_config.yaml`).

## 4.2 Vorhandene StorageClasses prüfen

```bash
kubectl get storageclass
```

- **Was:** listet alle StorageClasses mit Provisioner, `RECLAIMPOLICY` (was mit dem Speicher passiert, wenn der PVC gelöscht wird) und `VOLUMEBINDINGMODE` (wann der Speicher angelegt wird). Die Default StorageClass trägt den Zusatz `(default)` hinter dem Namen. Kurzform: `kubectl get sc`.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `No resources found` oder eine Tabelle mit einer oder mehreren StorageClasses.
- **Prüfen:** Je nach Ergebnis geht es mit einem der drei Fälle weiter.

| Fall | Ergebnis | Vorgehen |
|---|---|---|
| A | `No resources found` | Weiter mit 4.3. Getestet auf ruki-ws-02 am 03.11.2026. |
| B | Eine StorageClass mit `(default)` | Es gibt schon einen Standard. Entweder diesen behalten und 4.3 bis 4.4 überspringen (4.5 trotzdem ausführen), oder ihn nach *Change the default StorageClass* (kubernetes.io) mit `kubectl patch storageclass <name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'` zurücksetzen und mit 4.3 weitermachen. *Noch nicht getestet.* |
| C | StorageClasses ohne `(default)` | Weiter mit 4.3. Gibt es danach mehr als eine Default StorageClass, verwendet Kubernetes für PVCs ohne Klasse die zuletzt angelegte (Change the default StorageClass, kubernetes.io). Es sollte daher nur eine markiert sein. *Noch nicht getestet.* |

<!-- TODO: Fall B und C auf einem Rechner mit vorhandener StorageClass durchspielen -->

Beispiel (ruki-ws-02, 03.11.2026): `No resources found`

## 4.3 Local Path Provisioner installieren

**Version wählen.** Die aktuelle Version steht auf der Releases-Seite unter *Latest* (am 03.11.2026: `v0.0.37`, https://github.com/rancher/local-path-provisioner/releases). Das README nennt im Abschnitt *Installation* dieselbe Version. Wir installieren das Manifest aus diesem Tag, nicht aus `master`, damit die Installation wiederholbar bleibt:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml
```

- **Was:** lädt das Manifest aus dem Tag `v0.0.37` und legt alle darin beschriebenen Objekte an. Existiert ein Objekt schon, wird es aktualisiert. Für eine andere Version `v0.0.37` ersetzen. Das Manifest enthält neun Objekte:
  - den Namespace `local-path-storage`;
  - einen ServiceAccount sowie Role, ClusterRole und die beiden Bindings dazu (die Rechte des Provisioners);
  - das Deployment des Provisioners (Image `docker.io/rancher/local-path-provisioner:v0.0.37`);
  - die StorageClass `local-path` mit `volumeBindingMode: WaitForFirstConsumer` und `reclaimPolicy: Delete`;
  - die ConfigMap `local-path-config`. Sie legt das Speicherverzeichnis auf dem Knoten fest (`/opt/local-path-provisioner`) und enthält die Skripte `setup` und `teardown`, die die Verzeichnisse anlegen bzw. löschen.
- **Wo:** Terminal des Laborrechners (braucht Internetzugang).
- **Erwartet:** neun Zeilen, die alle auf `created` enden.
- **Prüfen:** Die Zeile `storageclass.storage.k8s.io/local-path created` ist dabei. `configured` statt `created` bedeutet, dass das Objekt schon existierte.

Beispiel (ruki-ws-02, 03.11.2026):

```
namespace/local-path-storage created
serviceaccount/local-path-provisioner-service-account created
role.rbac.authorization.k8s.io/local-path-provisioner-role created
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created
rolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
deployment.apps/local-path-provisioner created
storageclass.storage.k8s.io/local-path created
configmap/local-path-config created
```

Danach prüfen, ob der Provisioner läuft:

```bash
kubectl -n local-path-storage get pods -o wide
```

- **Was:** listet die Pods im Namespace `local-path-storage`. `-o wide` zeigt zusätzlich die IP des Pods und den Knoten, auf dem er läuft.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** ein Pod `local-path-provisioner-<hash>` mit `READY 1/1` und `STATUS Running`. Solange das Image geladen wird, steht dort kurz `ContainerCreating`.
- **Prüfen:** Bei `ImagePullBackOff` oder `ErrImagePull` zeigt `kubectl -n local-path-storage describe pod -l app=local-path-provisioner | tail -20` die Ursache.

Beispiel (ruki-ws-02, 03.11.2026):

```
NAME                                      READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
local-path-provisioner-54745b98cf-gnjw5   1/1     Running   0          15s   10.244.0.6   ruki-ws-02   <none>           <none>
```

## 4.4 local-path als Default markieren

Das Manifest legt die StorageClass `local-path` an, markiert sie aber nicht als Standard. Markiert wird eine StorageClass mit der Annotation `storageclass.kubernetes.io/is-default-class` und dem Wert `"true"`. Jeder andere Wert und eine fehlende Annotation gelten als `false` (Change the default StorageClass, kubernetes.io).

```bash
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

- **Was:** `kubectl patch` ändert ein bestehendes Objekt, ohne sein ganzes Manifest neu zu schreiben. `-p` übergibt die Änderung als JSON, das mit dem Objekt zusammengeführt wird. Hier kommt die Annotation `is-default-class: "true"` dazu.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `storageclass.storage.k8s.io/local-path patched`
- **Prüfen:** mit dem folgenden Befehl.

```bash
kubectl get storageclass
```

- **Was:** wie in 4.2.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `local-path (default)` mit Provisioner `rancher.io/local-path`, `Delete` und `WaitForFirstConsumer`.
- **Prüfen:** Fehlt `(default)`, zeigt `kubectl get sc local-path -o jsonpath='{.metadata.annotations}'` die gesetzten Annotationen.

Beispiel (ruki-ws-02, 03.11.2026):

```
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  58s
```

## 4.5 Test mit PVC und Pod

Der Test läuft wie im Abschnitt *Usage* des README: erst ein PVC, dann ein Pod, der ihn einbindet. Das PVC-Beispiel im Repository (`examples/pvc/pvc.yaml`) gibt `storageClassName: local-path` ausdrücklich an und prüft damit nicht, ob die Default StorageClass greift. Der PVC unten ist dasselbe Beispiel ohne diese Zeile, so wie die PVCs von Kubeflow aus 4.1.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-path-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 128Mi
EOF
```

- **Was:** `-f -` liest das Manifest von der Standardeingabe. Das Heredoc `<<EOF … EOF` schickt die Zeilen dazwischen dorthin. Angelegt wird im Namespace `default` ein PVC `local-path-pvc` mit 128 Mi und `ReadWriteOnce` (nur ein Knoten darf ihn schreibend einbinden), ohne StorageClass.
- **Wo:** Terminal des Laborrechners. Den Block als Ganzes einfügen.
- **Erwartet:** `persistentvolumeclaim/local-path-pvc created`
- **Prüfen:** mit dem folgenden Befehl.

```bash
kubectl get pvc local-path-pvc
```

- **Was:** zeigt Zustand, gebundenes Volume, Größe, Zugriffsmodus und StorageClass des PVC.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `STATUS Pending`, leere Spalte `VOLUME`, `STORAGECLASS local-path`.
- **Prüfen:** Entscheidend ist die Spalte `STORAGECLASS`: Dort steht `local-path`, obwohl der PVC keine Klasse angegeben hat. `Pending` ist hier richtig. Mit `WaitForFirstConsumer` wird das Volume erst angelegt, wenn ein Pod den PVC verwendet (Storage Classes, kubernetes.io). Ist die Spalte leer, fehlt die Default StorageClass (zurück zu 4.4).

Beispiel (ruki-ws-02, 03.11.2026):

```
NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
local-path-pvc   Pending                                      local-path     <unset>                 16s
```

Jetzt den Beispiel-Pod aus dem Repository anlegen. Er bindet den PVC `local-path-pvc` unter `/data` ein:

```bash
kubectl create -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/examples/pod/pod.yaml
```

- **Was:** legt den Pod `volume-test` mit dem Image `nginx:stable-alpine` an. Anders als `apply` bricht `create` ab, wenn der Pod schon existiert. Sobald der Pod eingeplant ist, startet der Provisioner einen kurzlebigen Helper-Pod, der mit dem Skript `setup` das Verzeichnis auf dem Knoten anlegt. Danach erstellt er das PV und bindet es an den PVC.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `pod/volume-test created`
- **Prüfen:** mit den folgenden Befehlen.

```bash
kubectl get pod volume-test -o wide
```

- **Was:** zeigt Zustand, IP und Knoten des Test-Pods.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `1/1 Running` nach einigen Sekunden, vorher eventuell `ContainerCreating`, solange das Image geladen wird.
- **Prüfen:** Bleibt der Pod länger als eine Minute in `Pending` oder `ContainerCreating`, zeigt `kubectl describe pod volume-test | tail -20` die Ursache.

Beispiel (ruki-ws-02, 03.11.2026):

```
NAME          READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
volume-test   1/1     Running   0          14s   10.244.0.8   ruki-ws-02   <none>           <none>
```

```bash
kubectl get pvc,pv
```

- **Was:** zeigt die PVCs im Namespace `default` und alle PVs des Clusters in einem Aufruf. Das Komma trennt die Objekttypen.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** Der PVC `local-path-pvc` ist `Bound`, mit einem Volume `pvc-<uid>`. Ein PV mit genau diesem Namen ist ebenfalls `Bound`, mit `RECLAIM POLICY Delete` und `CLAIM default/local-path-pvc`.
- **Prüfen:** Der Name in der Spalte `VOLUME` des PVC und der Name des PV stimmen überein.

Beispiel (ruki-ws-02, 03.11.2026):

```
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/local-path-pvc   Bound    pvc-0501c588-b3e9-48d1-b3bd-efb0ca75db98   128Mi      RWO            local-path     <unset>                 63s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-0501c588-b3e9-48d1-b3bd-efb0ca75db98   128Mi      RWO            Delete           Bound    default/local-path-pvc   local-path     <unset>                          27s
```

Das PV ist jünger als der PVC (27 s gegenüber 63 s): Es wurde erst angelegt, als der Pod startete (eigene Messung auf ruki-ws-02, 03.11.2026).

Zum Schluss wird geprüft, ob die Daten wirklich auf der Festplatte des Knotens landen. Zuerst eine Datei aus dem Pod heraus schreiben:

```bash
kubectl exec volume-test -- sh -c 'echo local-path-test > /data/test.txt && cat /data/test.txt'
```

- **Was:** `kubectl exec` führt einen Befehl im Container des Pods aus. `--` trennt die Optionen von `kubectl` vom Befehl im Container. `sh -c '…'` führt die ganze Zeichenkette in einer Shell aus: Sie schreibt `local-path-test` nach `/data/test.txt` und liest die Datei wieder.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** `local-path-test`
- **Prüfen:** mit dem folgenden Befehl auf dem Rechner selbst.

```bash
sudo ls -l /opt/local-path-provisioner/<volume>_default_local-path-pvc/
```

- **Was:** listet das Verzeichnis des Test-Volumes auf dem Rechner. Der Provisioner benennt die Verzeichnisse nach dem Muster `<PV-Name>_<Namespace>_<PVC-Name>` (README, Abschnitt *Storage classes*, Parameter `pathPattern`). `<volume>` durch den Namen aus der Spalte `VOLUME` ersetzen.
- **Wo:** Terminal des Laborrechners (auf dem Rechner, nicht im Pod).
- **Erwartet:** eine Datei `test.txt` mit 16 Byte (Text und Zeilenumbruch).
- **Prüfen:** `sudo cat` auf diese Datei zeigt `local-path-test`.

Beispiel (ruki-ws-02, 03.11.2026), Befehl mit `pvc-0501c588-b3e9-48d1-b3bd-efb0ca75db98`:

```
insgesamt 4
-rw-r--r-- 1 root root 16 Nov  3 14:26 test.txt
```

## 4.6 Test aufräumen

```bash
kubectl delete pod volume-test
```

- **Was:** löscht den Test-Pod. PVC und PV bleiben bestehen.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile, die mit `pod "volume-test" deleted` beginnt.
- **Prüfen:** `kubectl get pod volume-test` meldet `NotFound`, `kubectl get pvc local-path-pvc` zeigt weiter `Bound`.

Beispiel (ruki-ws-02, 03.11.2026, kubectl 1.36.6): `pod "volume-test" deleted from default namespace`

```bash
kubectl delete pvc local-path-pvc
```

- **Was:** löscht den PVC. Wegen `reclaimPolicy: Delete` löscht der Provisioner danach das PV und startet einen Helper-Pod, der mit dem Skript `teardown` das Verzeichnis auf dem Rechner entfernt. Die Daten darin sind damit weg.
- **Wo:** Terminal des Laborrechners.
- **Erwartet:** eine Zeile, die mit `persistentvolumeclaim "local-path-pvc" deleted` beginnt.
- **Prüfen:** Nach einigen Sekunden meldet `kubectl get pv` `No resources found`, und `sudo ls -l /opt/local-path-provisioner/` zeigt das Verzeichnis des Test-Volumes nicht mehr. Bleibt das PV in `Released`, zeigt `kubectl -n local-path-storage logs -l app=local-path-provisioner --tail=20` die Ursache.

Beispiel (ruki-ws-02, 03.11.2026, kubectl 1.36.6): `persistentvolumeclaim "local-path-pvc" deleted from default namespace`, danach `kubectl get pv`: `No resources found`

**Hinweis: Verzeichnisse aus einer früheren Installation.** Wurde der Rechner schon früher mit dem Local Path Provisioner betrieben, z. B. als Knoten eines anderen Clusters, kann `/opt/local-path-provisioner` bereits Verzeichnisse enthalten. Der neue Provisioner lässt sie unverändert. Er löscht nur Verzeichnisse von PVs, die er selbst angelegt hat. Diese Verzeichnisse nicht löschen, ohne vorher auf dem alten Cluster mit `kubectl get pv` geprüft zu haben, ob dort noch PVs auf sie verweisen. Am Namen erkennt man Namespace und PVC.

Beispiel (ruki-ws-02, 03.11.2026), `sudo ls -l /opt/local-path-provisioner/` nach dem Aufräumen. Die drei Verzeichnisse stammen aus der Zeit, in der ruki-ws-02 Worker im Cluster von ruki-ws-07 war, und haben ihr Datum behalten:

```
insgesamt 12
drwxrwxrwx  3 root root 4096 Apr 10  2026 pvc-1672c7a2-141c-4ba2-8f43-4165d8462529_azemo_dr-1024-pvc
drwxrwxrwx 11 root root 4096 Okt  9 16:15 pvc-5594a0cd-7dfb-4320-877e-1ad30238a919_student1_bomag-gpu-test-workspace
drwxrwxrwx 10 root root 4096 Okt 12 12:26 pvc-c0565c33-09ea-4f1c-a680-e82439414e92_student1_pipeline-cpu-2-workspace
```

<!-- TODO: auf ruki-ws-07 prüfen, ob noch PVs auf diese Verzeichnisse auf ruki-ws-02 verweisen -->

## 4.7 Grenzen des Local Path Provisioners

- **Ein Knoten:** Jedes PV ist an den Knoten gebunden, auf dem es angelegt wurde (Node Affinity über `kubernetes.io/hostname`, README, Abschnitt *Node Affinity Key*). Ein Pod mit diesem PVC kann nur auf diesem Knoten laufen. Für den einzelnen Laborrechner dieser Anleitung ist das kein Problem.
- **Keine Größenbegrenzung:** Die im PVC angeforderte Größe wird nicht durchgesetzt (README, Abschnitt *Compare to built-in Local Persistent Volume feature in Kubernetes*, Unterabschnitt *Cons*). Ein Volume kann wachsen, bis die Festplatte voll ist. `df -h /opt` zeigt den freien Platz.
- **Rechte:** Das Skript `setup` in der ConfigMap `local-path-config` legt die Verzeichnisse mit `mkdir -m 0777` an, also für alle Benutzer des Rechners les- und schreibbar. Im Beispiel oben sieht man das an `drwxrwxrwx`.
- **Löschen heißt Datenverlust:** Mit `reclaimPolicy: Delete` verschwinden die Daten, sobald der PVC gelöscht wird, z. B. mit einem Notebook samt Workspace-Volume. Wichtige Ergebnisse gehören daher zusätzlich in den Objektspeicher von Kubeflow oder in eine Sicherung.

Weiter: 5. Kubeflow installieren (folgt) | [Übersicht](../README.md)
