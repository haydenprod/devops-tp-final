# DevOps TP Final — Lacets Connectés API

> Mise en place d'une infrastructure DevOps complète pour le déploiement automatisé d'une API Node.js/MySQL sur Kubernetes, avec CI/CD et monitoring.

---

## Arborescence du projet

```
devops-tp-final/
├── Vagrantfile                  # Définition des VMs (k3s + monitoring)
├── ansible/
│   ├── inventory.ini            # Inventaire Ansible (généré automatiquement)
│   ├── playbook.yml             # Playbook principal
│   └── roles/
│       ├── k3s/                 # Installation de k3s
│       ├── docker/              # Installation de Docker
│       ├── node_exporter/       # Installation de node_exporter
│       ├── prometheus/          # Installation et config de Prometheus
│       └── grafana/             # Installation de Grafana
├── api-lacets/
│   ├── Dockerfile               # Image Docker optimisée (multi-stage)
│   └── ...                      # Code source de l'API
├── k8s/
│   ├── mysql.yaml               # Déploiement MySQL + PVC
│   └── api.yaml                 # Déploiement API + HPA + Service
├── prometheus/
│   └── prometheus.yml           # Configuration Prometheus (scrape configs)
├── .github/
│   └── workflows/
│       └── cicd.yml             # Pipeline CI/CD GitHub Actions
└── README.md
```

---

## Prérequis

- [VirtualBox](https://www.virtualbox.org/) ≥ 6.1
- [Vagrant](https://www.vagrantup.com/) ≥ 2.3
- [Ansible](https://www.ansible.com/) ≥ 2.12
- [Docker](https://www.docker.com/) (sur la machine hôte pour le build)

---

## Partie 1 — Infrastructure

### VMs provisionnées

| VM           | IP              | Rôle                     | RAM  |
| ------------ | --------------- | ------------------------ | ---- |
| `k3s`        | `192.168.56.10` | Cluster Kubernetes (k3s) | 2 Go |
| `monitoring` | `192.168.56.20` | Prometheus + Grafana     | 2 Go |

### Démarrage de l'infrastructure

```bash
# Démarrer les deux VMs et provisionner avec Ansible
vagrant up

# Vérifier l'état des VMs
vagrant status

# Vérifier que k3s fonctionne
vagrant ssh k3s -c "sudo systemctl status k3s"
vagrant ssh k3s -c "sudo kubectl get nodes"
```

### Provisionnement Ansible

L'inventaire Ansible est généré automatiquement par le Vagrantfile. Le playbook installe et configure :

- k3s sur la VM `k3s`
- Docker sur la VM `k3s`
- node_exporter sur les deux VMs
- Prometheus et Grafana sur la VM `monitoring`

```bash
# Relancer uniquement le provisionnement Ansible
vagrant provision
```

---

## Partie 2 — Conteneurisation

L'image Docker est basée sur une approche **multi-stage build** pour minimiser la taille finale.

### Image Docker Hub

```
docker pull slydxn/lacets-api:latest
```

### Build local

```bash
cd api-lacets
docker build -t lacets-api:latest .
docker images | grep lacets
```

### Modifications apportées au code source

L'application originale ([almoggutin/Node-Express-REST-API-MySQL-JS-Example](https://github.com/almoggutin/Node-Express-REST-API-MySQL-JS-Example)) a été adaptée :

- Variables d'environnement externalisées pour la connexion MySQL
- Ajout d'un health check endpoint `/health`

---

## Partie 3 — Déploiement Kubernetes

### Manifests

- **`k8s/mysql.yaml`** : Déploiement MySQL avec PersistentVolumeClaim (données persistantes)
- **`k8s/api.yaml`** : Déploiement de l'API avec HorizontalPodAutoscaler (min 1 pod, max 3)

### Déploiement manuel

```bash
vagrant ssh k3s
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

kubectl apply -f k8s/mysql.yaml
kubectl apply -f k8s/api.yaml

# Vérifications
kubectl get pods -A
kubectl get pvc
kubectl get hpa
kubectl get svc
```

### Architecture réseau

- L'API est exposée en **ClusterIP** uniquement (accessible depuis l'intérieur du cluster)
- MySQL est accessible uniquement depuis l'API (pas de port exposé à l'extérieur)
- L'HPA déclenche le scale-up en cas de pic CPU ou mémoire

---

## Partie 4 — CI/CD

### Pipeline GitHub Actions

La pipeline se déclenche automatiquement à chaque push sur la branche `main` et réalise :

1. **Checkout** du dépôt
2. **Build** de l'image Docker
3. **Déploiement** sur le cluster k3s
4. **Vérification** des pods

### Runner self-hosted

Le runner GitHub Actions est installé directement sur la VM `k3s` (`192.168.56.10`), ce qui lui donne un accès direct au cluster sans configuration kubeconfig externe.

```bash
# Vérifier l'état du runner
vagrant ssh k3s -c "sudo /home/vagrant/devops-tp-final/actions-runner/svc.sh status"
```

### Déclencher la pipeline

```bash
# Un simple push sur main déclenche la pipeline
git add .
git commit -m "deploy"
git push origin main
```

---

## Partie 5 — Monitoring et Observabilité

### Architecture

```
k3s (192.168.56.10)          monitoring (192.168.56.20)
┌─────────────────┐          ┌──────────────────────────┐
│  node_exporter  │◄─────────│  Prometheus :9090        │
│  :9100          │  scrape  │  Grafana    :3000        │
└─────────────────┘          │  node_exporter :9100     │
                             └──────────────────────────┘
```

### Accès aux interfaces

| Service    | URL                         | Identifiants      |
| ---------- | --------------------------- | ----------------- |
| Prometheus | `http://192.168.56.20:9090` | —                 |
| Grafana    | `http://192.168.56.20:3000` | `admin` / `admin` |

### Vérification des targets Prometheus

```bash
# Vérifier que les 2 VMs sont bien scrapées
curl http://192.168.56.20:9090/api/v1/targets | python3 -m json.tool | grep -A2 "health"
```

Les deux targets doivent être en état `"up"` :

- `192.168.56.10:9100` — VM k3s
- `192.168.56.20:9100` — VM monitoring

### Dashboard Grafana

Le dashboard **Node Exporter Full** (ID [1860](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)) est importé et affiche les métriques CPU, mémoire, disque et réseau des deux VMs.

```bash
# Vérifier que Grafana est actif
vagrant ssh monitoring -c "sudo systemctl status grafana-server --no-pager"

# Vérifier que Prometheus est actif
vagrant ssh monitoring -c "sudo systemctl status prometheus --no-pager"

# Vérifier que node_exporter est actif sur les deux VMs
vagrant ssh k3s -c "sudo systemctl status node_exporter --no-pager"
vagrant ssh monitoring -c "sudo systemctl status node_exporter --no-pager"
```

---

## Commandes utiles

```bash
# Démarrer tout l'environnement
vagrant up

# Arrêter les VMs
vagrant halt

# Détruire et recréer
vagrant destroy -f && vagrant up

# SSH sur les VMs
vagrant ssh k3s
vagrant ssh monitoring

# Logs k3s
vagrant ssh k3s -c "sudo journalctl -u k3s -f"

# État des pods Kubernetes
vagrant ssh k3s -c "sudo kubectl get pods -A"
```

---

## Idempotence

Tous les playbooks Ansible, manifests Kubernetes et scripts sont conçus pour être **idempotents** : les relancer plusieurs fois produit toujours le même résultat sans effets de bord.
