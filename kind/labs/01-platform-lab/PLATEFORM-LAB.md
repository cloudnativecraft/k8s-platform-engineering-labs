# Kubernetes KIND Lab — Déploiement et dépannage d'une application

Ce dépôt contient un laboratoire Kubernetes pratique réalisé avec **KIND**. Il met en œuvre un cluster local multinœud, un `Deployment` volontairement défectueux et un `Service` permettant d'exposer l'application dans le cluster.

Le but n'est pas seulement de déployer une application, mais d'apprendre à diagnostiquer méthodiquement plusieurs problèmes courants : Pods en état `Pending`, mauvaise `readinessProbe`, configuration injectée par variables d'environnement, mise à jour défectueuse et rollback.

## Compétences travaillées

- créer un cluster Kubernetes local multinœud avec KIND ;
- utiliser les namespaces et les contextes `kubectl` ;
- créer un `ConfigMap` et un `Secret` avec des commandes impératives ;
- diagnostiquer un problème de scheduling ;
- utiliser les labels et `nodeSelector` ;
- comprendre et corriger une `readinessProbe` ;
- exposer un Deployment avec un Service `ClusterIP` ;
- vérifier les EndpointSlices ;
- mettre à jour un Deployment et effectuer un rollback ;
- extraire des informations ciblées avec JSONPath.

## Fichiers du dépôt

```text
.
├── kind-platform-lab.yaml
├── catalog-deployment.yaml
├── catalog-service.yaml
└── README.md
```

### `kind-platform-lab.yaml`

Ce manifeste est lu par KIND, et non directement par Kubernetes. Il décrit le cluster local à créer :

- un nœud `control-plane`, qui héberge les composants de contrôle Kubernetes ;
- deux nœuds `worker`, destinés à exécuter les workloads applicatifs.

Il permet de simuler plus fidèlement un cluster multinœud qu'un cluster composé d'un seul nœud.

### `catalog-deployment.yaml`

Ce manifeste définit le `Deployment` `catalog`. Un Deployment décrit l'état souhaité de l'application et délègue la gestion des Pods à un ReplicaSet.

Il contient notamment :

- trois réplicas de l'application ;
- l'image du conteneur NGINX ;
- le label de Pod `app: catalog` ;
- un `nodeSelector` imposant l'exécution sur les nœuds portant `workload=applications` ;
- les variables `APP_COLOR` et `APP_ENV`, provenant respectivement d'un `ConfigMap` et d'un `Secret` ;
- une `readinessProbe` HTTP servant à décider si un Pod peut recevoir du trafic.

Le manifeste initial contient volontairement deux problèmes : aucun nœud ne possède d'abord le label exigé par le `nodeSelector`, puis la probe interroge un chemin HTTP qui n'existe pas dans l'image NGINX.

### `catalog-service.yaml`

Ce manifeste définit un Service `ClusterIP` nommé `catalog`.

Le Service fournit une adresse réseau stable à l'intérieur du cluster. Son sélecteur `app=catalog` associe le Service aux Pods portant ce label. Le port `80` du Service est transmis au port `80` des conteneurs.

Un Pod simplement `Running` ne devient pas forcément un endpoint utilisable. Il doit aussi être sélectionné par le Service et être considéré comme `Ready`.

## 1. Créer le cluster

Prérequis : Docker, KIND et `kubectl` doivent être installés.

```bash
docker version
kind version
kubectl version --client
```

Création et vérification du cluster :

```bash
kind create cluster --config kind-platform-lab.yaml
kubectl cluster-info --context kind-platform-lab
kubectl get nodes -o wide
```

Le nom du contexte est normalement `kind-platform-lab` lorsque le cluster KIND s'appelle `platform-lab`.

## 2. Préparer le namespace

```bash
kubectl create namespace catalog
kubectl config set-context --current --namespace=catalog
```

La deuxième commande définit `catalog` comme namespace par défaut du contexte courant. Les commandes suivantes n'ont donc plus besoin de l'option `-n catalog`.

Vérification :

```bash
kubectl config view --minify -o jsonpath='{..namespace}'
echo
```

Résultat attendu :

```text
catalog
```

## 3. Créer le ConfigMap et le Secret

### ConfigMap

Prévisualisation sans création :

```bash
kubectl create configmap catalog-config \
  --from-literal=APP_COLOR=blue \
  --dry-run=client -o yaml
```

Création :

```bash
kubectl create configmap catalog-config \
  --from-literal=APP_COLOR=blue
```

Vérification ciblée avec JSONPath :

```bash
kubectl get configmap catalog-config \
  -o jsonpath='{.data.APP_COLOR}'
echo
```

### Secret

Prévisualisation sans création :

```bash
kubectl create secret generic catalog-secret \
  --from-literal=APP_ENV=production \
  --dry-run=client -o yaml
```

Création :

```bash
kubectl create secret generic catalog-secret \
  --from-literal=APP_ENV=production
```

La valeur placée dans `.data.APP_ENV` est encodée en Base64. Cet encodage ne constitue pas un chiffrement.

```bash
kubectl get secret catalog-secret \
  -o jsonpath='{.data.APP_ENV}' | base64 -d
echo
```

Résultat attendu :

```text
production
```

Pour éviter d'afficher la valeur d'un Secret dans un terminal partagé ou dans des logs, on peut seulement vérifier que la clé existe :

```bash
kubectl get secret catalog-secret \
  -o jsonpath='{.data.APP_ENV}' | wc -c
```

## 4. Déployer l'application

```bash
kubectl apply -f catalog-deployment.yaml
kubectl get deployment,replicaset,pods -o wide
```

À ce stade, les Pods doivent rester `Pending` à cause de la contrainte de placement volontairement non satisfaite.

## 5. Diagnostiquer les Pods Pending

La bonne démarche consiste à observer l'incident avant de modifier ou de supprimer les Pods.

```bash
kubectl get pods
kubectl describe pod <nom-du-pod>
kubectl get nodes --show-labels
kubectl get events --sort-by=.metadata.creationTimestamp
```

Le Deployment exige :

```yaml
nodeSelector:
  workload: applications
```

Un `nodeSelector` est une contrainte stricte : le Scheduler ne peut placer le Pod que sur un nœud possédant exactement ce label. Au départ, aucun worker ne le possède. La section `Events` du Pod doit donc signaler que les nœuds ne correspondent pas au selector.

La correction attendue consiste à étiqueter les workers, sans supprimer la contrainte métier du Deployment :

```bash
kubectl label node platform-lab-worker workload=applications
kubectl label node platform-lab-worker2 workload=applications
```

Vérification ciblée :

```bash
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,WORKLOAD:.metadata.labels.workload'

kubectl get pods -o wide
```

Les trois Pods peuvent être répartis sur les deux workers. Il n'est pas garanti que chaque worker reçoive le même nombre de Pods : `nodeSelector` autorise le placement, mais n'impose pas une répartition équilibrée.

### `nodeSelector`, taints et tolerations

Ces notions ne jouent pas le même rôle :

- un label décrit un nœud ;
- `nodeSelector` exige certains labels et attire indirectement le Pod vers les nœuds compatibles ;
- un taint repousse les Pods qui ne possèdent pas la toleration correspondante ;
- une toleration autorise le passage d'un taint, mais ne sélectionne pas un nœud.

Ajouter une toleration générique avec `operator: Exists` n'était pas nécessaire pour résoudre ce lab. Une telle toleration serait trop large et pourrait autoriser le Pod à tolérer des taints non souhaités. La correction principale attendue était d'ajouter `workload=applications` aux workers.

## 6. Diagnostiquer les Pods Running mais NotReady

Une fois les Pods planifiés, les conteneurs démarrent, mais la `readinessProbe` appelle un chemin inexistant, par exemple `/ready` ou `/healthy`.

```bash
kubectl get pods
kubectl describe pod <nom-du-pod>
kubectl logs <nom-du-pod>
kubectl get deployment catalog
kubectl rollout status deployment/catalog --timeout=20s
```

Le résultat important se trouve principalement dans les événements de `describe` : la probe HTTP reçoit une réponse d'échec, typiquement `404`. Le conteneur reste en cours d'exécution, mais le Pod demeure `NotReady`.

Une `readinessProbe` en échec :

- ne redémarre pas automatiquement le conteneur ;
- retire le Pod des endpoints prêts du Service ;
- empêche le Deployment d'atteindre sa disponibilité attendue.

Ce comportement est différent de celui d'une `livenessProbe`, dont les échecs répétés peuvent entraîner un redémarrage du conteneur.

Correction impérative ciblée :

```bash
kubectl patch deployment catalog --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/"}]'
```

La modification du template du Pod crée une nouvelle révision du Deployment et déclenche un rolling update.

```bash
kubectl rollout status deployment/catalog
kubectl get pods
kubectl get deployment catalog
```

Résultat attendu pour le Deployment :

```text
READY   UP-TO-DATE   AVAILABLE
3/3     3            3
```

## 7. Exposer l'application

Si le Service est fourni dans `catalog-service.yaml` :

```bash
kubectl apply -f catalog-service.yaml
```

Équivalent impératif :

```bash
kubectl expose deployment catalog \
  --name=catalog \
  --type=ClusterIP \
  --port=80 \
  --target-port=80
```

Il faut utiliser une seule de ces deux méthodes pour la création initiale afin d'éviter une erreur `AlreadyExists`.

Vérifications :

```bash
kubectl get service catalog
kubectl describe service catalog
kubectl get endpointslices \
  -l kubernetes.io/service-name=catalog
```

Test depuis l'intérieur du cluster :

```bash
kubectl run curl-test \
  --image=curlimages/curl:8.12.1 \
  --restart=Never \
  --rm -it \
  -- curl -sS http://catalog
```

Le nom DNS `catalog` fonctionne ici parce que le Pod de test et le Service se trouvent dans le même namespace. Depuis un autre namespace, il faudrait utiliser au minimum `catalog.catalog`, ou le nom complet `catalog.catalog.svc.cluster.local`.

## 8. Contrôler les variables d'environnement

Récupérer dynamiquement le nom d'un Pod avec JSONPath :

```bash
POD=$(kubectl get pods -l app=catalog \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec "$POD" -- sh -c 'env | grep "^APP_"'
```

Résultat attendu :

```text
APP_COLOR=blue
APP_ENV=production
```

Mise à jour déclarative du ConfigMap à partir d'une commande impérative :

```bash
kubectl create configmap catalog-config \
  --from-literal=APP_COLOR=green \
  --dry-run=client -o yaml | kubectl apply -f -
```

Les variables d'environnement des conteneurs déjà démarrés ne changent pas. Elles sont injectées lors de leur création. Il faut redémarrer progressivement le Deployment :

```bash
kubectl rollout restart deployment/catalog
kubectl rollout status deployment/catalog
```

Puis récupérer un nouveau nom de Pod et contrôler la valeur :

```bash
POD=$(kubectl get pods -l app=catalog \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec "$POD" -- printenv APP_COLOR
```

Résultat attendu :

```text
green
```

## 9. Simuler une mauvaise mise à jour et effectuer un rollback

Déploiement volontaire d'une image inexistante :

```bash
kubectl set image deployment/catalog \
  catalog=nginx:version-inexistante
```

Diagnostic :

```bash
kubectl rollout status deployment/catalog --timeout=30s
kubectl get pods
kubectl describe pod <nouveau-pod>
kubectl rollout history deployment/catalog
```

Le nouveau Pod doit normalement passer en `ImagePullBackOff` après plusieurs tentatives. Pendant un rolling update, les anciens Pods prêts peuvent rester disponibles : l'échec d'une nouvelle version ne signifie donc pas nécessairement une interruption immédiate du Service.

Rollback vers la révision précédente :

```bash
kubectl rollout undo deployment/catalog
kubectl rollout status deployment/catalog
```

Il est préférable d'utiliser la révision précédente automatiquement lorsque c'est bien elle qui est fonctionnelle. Une commande telle que `--to-revision=9` n'est correcte que si l'historique montre explicitement que la révision 9 est la cible voulue.

```bash
kubectl rollout history deployment/catalog
kubectl get deployment,replicaset,pods
kubectl get endpointslices \
  -l kubernetes.io/service-name=catalog
```

## 10. Défi impératif et JSONPath

Passer à cinq réplicas :

```bash
kubectl scale deployment/catalog --replicas=5
kubectl rollout status deployment/catalog
```

Afficher uniquement les noms des Pods et leurs nœuds avec JSONPath :

```bash
kubectl get pods -l app=catalog \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
```

La variante `custom-columns` utilisée pendant le lab est également correcte et souvent plus lisible :

```bash
kubectl get pods -l app=catalog \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName'
```

Afficher uniquement l'image du conteneur :

```bash
kubectl get deployment catalog \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
```

Revenir à trois réplicas :

```bash
kubectl scale deployment/catalog --replicas=3
kubectl rollout status deployment/catalog
```

Compter les endpoints prêts du Service :

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=catalog \
  -o jsonpath='{range .items[*].endpoints[?(@.conditions.ready==true)]}{.addresses[0]}{"\n"}{end}' \
  | wc -l
```

Résultat attendu :

```text
3
```

## Méthode de diagnostic retenue

Pour chaque incident, la démarche suivie est :

1. observer l'état avec `kubectl get` ;
2. consulter les détails et les événements avec `kubectl describe` ;
3. consulter les logs lorsque le conteneur a effectivement démarré ;
4. identifier la contrainte ou la configuration responsable ;
5. effectuer la correction la plus ciblée possible ;
6. contrôler le rollout, les Pods et les endpoints.

## Bilan du lab

### Ce qui a été correctement réalisé

- création d'un cluster KIND à trois nœuds ;
- création et utilisation du namespace `catalog` ;
- création impérative du ConfigMap et du Secret avec prévisualisation `--dry-run=client` ;
- consultation de la valeur encodée du Secret avec JSONPath ;
- utilisation de `describe`, des événements et des labels pour diagnostiquer les Pods `Pending` ;
- ajout du label attendu aux workers au lieu de supprimer immédiatement les Pods ;
- identification de la mauvaise `readinessProbe` et remplacement de son chemin par `/` ;
- contrôle du Service et des EndpointSlices ;
- constat qu'un changement de ConfigMap ne modifie pas les variables des processus déjà lancés ;
- utilisation du mécanisme de rollout et de rollback du Deployment ;
- utilisation correcte de `scale`, `custom-columns` et JSONPath pour obtenir des informations ciblées.

### Points à corriger ou à préciser

- `kubernetes.io/os=linux` ne résout pas le problème initial si tous les nœuds possèdent déjà ce label. La contrainte utile du lab est `workload=applications`.
- Une toleration ne fait pas correspondre un Pod à un nœud. Elle permet seulement au Pod de tolérer un taint.
- Il ne fallait pas supprimer le `nodeSelector` métier. La correction attendue était de rendre les workers compatibles en leur ajoutant le bon label.
- Les trois Pods ne sont pas nécessairement répartis sur trois nœuds. Avec seulement deux workers labellisés, plusieurs Pods peuvent s'exécuter sur le même worker.
- Une readiness probe est exécutée par le kubelet, pas « par le cluster » de manière générale.
- L'erreur de probe n'est pas une erreur de syntaxe ou de parsing dans ce cas : NGINX répond sur `/`, mais pas sur le chemin erroné.
- Un Pod `NotReady` peut être présent dans un EndpointSlice, mais il y apparaît avec `conditions.ready=false` et n'est normalement pas utilisé comme backend prêt.
- Il faut vérifier le résultat après chaque `apply`, `patch`, `scale`, `restart` ou `undo`, notamment avec `kubectl rollout status`.
- Le numéro d'une révision n'est pas fixe. Il faut consulter `kubectl rollout history` avant d'utiliser `--to-revision`.

## Concepts à retenir

- Le Scheduler explique généralement ses refus de placement dans les événements du Pod.
- `nodeSelector` impose une condition stricte sur les labels du nœud.
- Les tolerations et les node selectors répondent à deux problèmes différents.
- `Running` décrit l'état d'exécution du Pod ; `Ready` indique s'il peut recevoir du trafic.
- Une readiness probe gère l'éligibilité au trafic, tandis qu'une liveness probe peut provoquer un redémarrage.
- Un Service sélectionne les Pods par labels et s'appuie sur les EndpointSlices.
- Une variable issue d'un ConfigMap ou d'un Secret est fixée au démarrage du conteneur.
- Un Deployment conserve des ReplicaSets, ce qui rend possible le rollback.
- Une commande n'est pas terminée tant que son effet n'a pas été vérifié.

## Nettoyage

```bash
kind delete cluster --name platform-lab
```

Cette commande supprime entièrement le cluster KIND et tous les objets créés pendant le laboratoire.
