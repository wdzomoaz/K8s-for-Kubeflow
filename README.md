# K8s-for-Kubeflow

Installationsanleitung für Kubernetes und Kubeflow auf einem einzelnen Linux-Rechner im Labor (kein verteilter Cluster). Am Ende läuft eine Kubeflow-Pipeline mit einem GPU-Schritt.

Diese Anleitung legt keine feste Version fest. Sie zeigt, wie man prüft, welche Kubernetes-Version zu welcher Kubeflow-Version passt. Die Versionsangaben in den Beispielen sind eine Momentaufnahme (Stand 02.11.2026) und müssen vor jeder neuen Installation erneut geprüft werden.

## Inhalt

1. [Voraussetzungen und Kompatibilität prüfen](docs/01-voraussetzungen.md)
2. [Kubernetes mit kubeadm](docs/02-kubernetes-kubeadm.md)
3. [GPU-Check und NVIDIA Device Plugin](docs/03-gpu-device-plugin.md)
4. [Speicher: Local Path Provisioner als Default StorageClass](docs/04-local-path-provisioner.md)
5. [Kubeflow installieren](docs/05-kubeflow-installieren.md)
6. Zugriff auf das Dashboard (folgt)
7. Erster Test: Pipeline mit GPU-Schritt (folgt)

## Aufbau jedes Befehls

Zu jedem Befehl stehen vier Angaben:

- **Was:** was der Befehl tut und was die Optionen bedeuten.
- **Wo:** auf welchem Rechner bzw. in welchem Terminal er ausgeführt wird.
- **Erwartet:** welches Ergebnis man sehen sollte.
- **Prüfen:** woran man erkennt, dass der Schritt geklappt hat.

Wo eine echte Ausgabe vom Laborrechner vorliegt, steht sie als Beispiel unter dem Befehl, mit Rechnername und Datum. Sie zeigt einen konkreten Fall und ersetzt nicht die allgemeine Beschreibung unter **Erwartet**.

**Sprache der Ausgaben:** Ist die Systemsprache des Rechners Deutsch, geben viele Programme ihre Meldungen auf Deutsch aus, z. B. `sysctl`, `apt`, `apt-mark`, `free`, `df` und die Meldung für unbekannte Befehle. Die Anleitung nennt an diesen Stellen beide Varianten. Fachbegriffe in Kubernetes-Ausgaben (`Running`, `Ready`, `created`) bleiben immer Englisch.

## Quellen

- Kubeflow Releases: https://www.kubeflow.org/docs/releases/
- Kubeflow Community Distribution 26.03: https://www.kubeflow.org/docs/releases/kubeflow-26.03/
- Release Notes der Community Distribution: https://github.com/kubeflow/community-distribution/releases
- Kubeflow Community Distribution (Repository): https://github.com/kubeflow/community-distribution
- Kubernetes Version Skew Policy: https://kubernetes.io/releases/version-skew-policy/
- Installing kubeadm (Kubernetes v1.36): https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Install and Set Up kubectl on Linux: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
- Declarative Management of Kubernetes Objects Using Kustomize: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
- Object Names and IDs: https://kubernetes.io/docs/concepts/overview/working-with-objects/names/
- Container Runtimes: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- Kubernetes v1.34: Autoconfiguration for Node Cgroup Driver Goes GA (Blog): https://v1-36.docs.kubernetes.io/blog/2025/09/12/kubernetes-v1-34-cri-cgroup-driver-lookup-now-ga/
- Getting started with containerd: https://containerd.io/docs/getting-started/
- Install Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Docker, Packet filtering and firewalls: https://docs.docker.com/engine/network/packet-filtering-firewalls/
- Docker, Start containers automatically: https://docs.docker.com/engine/containers/start-containers-automatically/
- Creating a cluster with kubeadm (Kubernetes v1.36): https://v1-36.docs.kubernetes.io/docs/setup/independent/create-cluster-kubeadm
- Creating a cluster with kubeadm, Abschnitt *Network setup*: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- kubeadm Configuration (v1beta4): https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/
- kubeadm config validate: https://kubernetes.io/docs/reference/setup-tools/kubeadm/generated/kubeadm_config/kubeadm_config_validate/
- kubeadm init phase control-plane all (Kubernetes v1.35, Feature Gate `ControlPlaneKubeletLocalMode`): https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/generated/kubeadm_init/kubeadm_init_phase_control-plane_all/
- KEP-4471, Make a control-plane's kubelet point to the local API Server on kubeadm join: https://github.com/kubernetes/enhancements/blob/master/keps/sig-cluster-lifecycle/kubeadm/4471-cp-join-kubelet-local-apiserver/README.md
- kubeadm reset (Kubernetes v1.36): https://v1-36.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/
- Safely Drain a Node: https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/
- Reconfiguring a kubeadm cluster: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure/
- Certificate Management with kubeadm: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/
- Flannel: https://github.com/flannel-io/flannel
- Flannel Releases: https://github.com/flannel-io/flannel/releases
- Flannel mit Kubernetes: https://github.com/flannel-io/flannel/blob/master/Documentation/kubernetes.md
- Schedule GPUs: https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/
- CUDA Toolkit, Driver, and Architecture Matrix (NVIDIA): https://docs.nvidia.com/datacenter/tesla/drivers/cuda-toolkit-driver-and-architecture-matrix.html
- Installing the NVIDIA Container Toolkit: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
- NVIDIA device plugin for Kubernetes (Repository und README): https://github.com/NVIDIA/k8s-device-plugin
- NVIDIA device plugin, Releases: https://github.com/NVIDIA/k8s-device-plugin/releases
- NVIDIA device plugin v0.20.0, statisches Manifest: https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.20.0/deployments/static/nvidia-device-plugin.yml
- Storage Classes: https://kubernetes.io/docs/concepts/storage/storage-classes/
- Change the default StorageClass: https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/
- Local Path Provisioner (Repository und README): https://github.com/rancher/local-path-provisioner
- Local Path Provisioner, Releases: https://github.com/rancher/local-path-provisioner/releases
- Local Path Provisioner v0.0.37, Manifest: https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml
- Local Path Provisioner v0.0.37, Beispiel-Pod: https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/examples/pod/pod.yaml
- Kubeflow Manifests v1.10, README (Voraussetzung Default StorageClass): https://github.com/kubeflow/community-distribution/blob/v1.10-branch/README.md
- Kubeflow Community Distribution 26.03, Manifeste (PVCs, `spawner_ui_config.yaml`): https://github.com/kubeflow/community-distribution/tree/release-26.03
- Kubeflow Community Distribution 26.03.1, Release Notes: https://github.com/kubeflow/community-distribution/releases/tag/26.03.1
- Kubeflow Community Distribution 26.03.1, README (Installation, Ressourcen, inotify, Standardbenutzer): https://github.com/kubeflow/community-distribution/blob/26.03.1/README.md
- Kustomize v5.8.1, Release mit Prüfsummen: https://github.com/kubernetes-sigs/kustomize/releases/tag/kustomize%2Fv5.8.1
- kubectl kustomize: https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/
- sysctl.d(5), Linux manual page: https://www.man7.org/linux/man-pages/man5/sysctl.d.5.html

Alle Links geprüft am 02.11. bzw. 03.11.2026.
