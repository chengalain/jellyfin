# jellyfin

Déploie Jellyfin dans le cluster k3s via le chart Helm officiel. Usage
cible : Live TV / IPTV uniquement, pas de bibliothèque média à stocker.

## Ce que fait le rôle

- Ajoute le repo Helm `jellyfin` (`https://jellyfin.github.io/jellyfin-helm`)
- Déploie le chart via Helm avec les values rendues depuis
  `templates/jellyfin-values.yaml.j2` (namespace créé par Helm lui-même via
  `create_namespace: true`)

## Prérequis / points d'attention

- Nécessite un kubeconfig valide en `~/.kube/config` pointant vers le
  cluster — généré par le rôle [`k3s_server`](https://gitlab.com/homelab1764835/k3s-server)
  lors du provisioning. Doit donc être exécuté après.
- `templates/jellyfin-values.yaml.j2` n'a pas besoin d'être chiffré : pas de
  secret requis pour cette itération (pas d'auth externe, pas de DB). Si une
  valeur sensible s'y ajoute plus tard (ex : token M3U/EPG), passer la
  variable correspondante par ansible-vault côté consommateur.
- Le template ne surcharge que les clés qui diffèrent des defaults du chart
  (vérifiés contre `jellyfin/jellyfin-helm` sur GitHub) — tout le reste est
  laissé à la valeur par défaut du chart, jamais redéfini ici pour rien.
  Si tu changes de version de chart majeure, revérifie
  `helm show values jellyfin/jellyfin` : ses defaults ont pu bouger.
- `jellyfin_service_monitor_enabled` pilote à la fois `metrics.enabled` et
  `metrics.serviceMonitor.enabled` dans le values — le chart n'active le
  ServiceMonitor que si les **deux** sont à `true`
  (`templates/serviceMonitor.yaml` du chart). Il ne scrapera de toute façon
  rien d'utile tant qu'aucun exporter/plugin de métriques Jellyfin n'est
  branché — il n'y a pas de endpoint `/metrics` natif.
- ⚠️ `jellyfin_chart_version` est vide par défaut (= toujours la dernière
  version du chart au moment du run). Une fois Jellyfin déployé, récupère la
  version installée (`helm -n jellyfin list`) et fige-la via cette variable
  pour éviter une mise à jour silencieuse au prochain run.
- Pas de `handlers/` : rien à redémarrer côté Ansible (Helm gère ça).
- La config Live TV (tuner M3U, source EPG) se fait dans l'UI Jellyfin après
  déploiement, hors scope de ce rôle.
- Pas d'Ingress : comme `authentik`/`levelup`, l'exposition publique passe
  par une route cloudflared directe vers le Service ClusterIP (rôle
  [`cloudflared`](https://gitlab.com/homelab1764835/cloudflared)), à ajouter
  séparément via la variable `cloudflared_ingress` du consommateur.

## Variables principales

Voir [`defaults/main.yml`](defaults/main.yml).

| Variable                          | Défaut                                       | Description                                                    |
| ------------------------------------ | ----------------------------------------------- | ------------------------------------------------------------------- |
| `jellyfin_namespace`                | `jellyfin`                                   | Namespace k8s cible                                            |
| `jellyfin_release_name`             | `jellyfin`                                   | Nom de la release Helm                                         |
| `jellyfin_chart_repo_url`           | `https://jellyfin.github.io/jellyfin-helm`   | URL du repo Helm                                               |
| `jellyfin_chart_ref`                | `jellyfin/jellyfin`                          | Référence du chart                                             |
| `jellyfin_chart_version`            | `""` (dernière)                              | Version du chart, à figer une fois connue                      |
| `jellyfin_image_tag`                | `""` (défaut du chart)                       | Version de Jellyfin, à figer une fois connue                   |
| `jellyfin_config_storage_size`      | `1Gi`                                        | Taille du PVC de config (chart : `5Gi`)                        |
| `jellyfin_storage_class`            | `local-path`                                 | StorageClass du PVC de config (chart : celle par défaut du cluster) |
| `jellyfin_media_storage_enabled`    | `false`                                      | PVC média (chart : `true`/25Gi — désactivé ici, Live TV only)  |
| `jellyfin_service_monitor_enabled`  | `true`                                       | Active le ServiceMonitor Prometheus (chart : `false`)          |

## Exemple

```yaml
- hosts: localhost
  connection: local
  roles:
    - jellyfin
```
