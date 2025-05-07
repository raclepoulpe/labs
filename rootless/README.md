# Sécurisation des Conteneurs : Focus sur l'approche Rootless

**Public cible :** Étudiants en informatique

**Durée totale estimée :** 2 heures 45 minutes - 3 heures 15 minutes

## Objectifs du cours

- Comprendre les enjeux de la sécurité des conteneurs.
- Identifier les risques liés à l'exécution de conteneurs en tant que root.
- Maîtriser le concept de conteneurs rootless et ses avantages en matière de sécurité.
- Savoir configurer et exécuter des conteneurs rootless dans un environnement OpenShift.
- Comprendre les limitations et les considérations spécifiques lors de l'utilisation de conteneurs rootless.
- Être informé des étapes pour construire des images rootless.

## I. Introduction à la Sécurité des Conteneurs (environ 25 minutes)

Bienvenue à la première partie de notre cours dédié à la sécurisation des conteneurs. Avant de plonger dans le vif du sujet, il est essentiel de bien comprendre les fondations sur lesquelles repose cette technologie et les défis de sécurité qu'elle soulève.

- **Rappel des concepts de base des conteneurs (environ 7 minutes) :** Commençons par un petit rappel. Qu'est-ce qu'un conteneur, fondamentalement ? C'est un environnement isolé qui regroupe une application et toutes ses dépendances. Pensez-y comme à une bulle logicielle autonome. Pour réaliser cette isolation, le noyau Linux utilise des mécanismes appelés **namespaces**. Vous devez connaître les principaux : le **PID namespace** pour l'isolation des identifiants de processus, le **Network namespace** pour l'isolation du réseau, le **Mount namespace** pour l'isolation du système de fichiers, l'**UTS namespace** pour l'isolation du nom d'hôte, l'**IPC namespace** pour l'isolation des communications inter-processus, et enfin, celui qui nous intéressera particulièrement plus tard, l'**User namespace**.  
    En parallèle de l'isolation, nous avons les **cgroups**, ou Control Groups. Ils permettent de limiter et de contrôler l'utilisation des ressources par les conteneurs : CPU, mémoire, entrées/sorties disque, etc. C'est crucial pour éviter qu'un conteneur ne monopolise les ressources de l'hôte.  
    Et n'oublions pas que tout conteneur est créé à partir d'une **image**. Cette image est un modèle statique contenant tout le nécessaire pour exécuter l'application. La sécurité commence dès la source de ces images.  
    Exemple de Dockerfile montrant la création d'une image avec des couches (layers) :

Fichier Dockerfile-ubunutu-nginx :

```dockerfile
FROM ubuntu:latest  
RUN apt-get update && apt-get install -y nginx  
COPY www/index.html /var/www/html/  
EXPOSE 80  
CMD ["nginx", "-g", "daemon off;"]
```

Exemple de commande pour exécuter un conteneur en spécifiant les ressources (cgroups) :

```shell
docker build -t mynginx:ubuntu-1 --file Dockerfile-ubunutu-nginx .
docker run -d --name mynginx --cpu-shares 512 --memory 512m -p 8080:80 mynginx:ubuntu-1
```

Comparaison avec l'image officielle proposée par Nginx: 

```shell
docker run -d --name mynginx-official --cpu-shares 512 --memory 512m -p 8081:80 -v ./www:/usr/share/nginx/html nginx
```

- **Les enjeux de la sécurité des conteneurs (environ 7 minutes) :** Pourquoi devons-nous nous soucier de la sécurité de ces conteneurs ? Plusieurs aspects sont critiques :  
  - Les **vulnérabilités des images** : Si une image contient des failles de sécurité dans ses composants (système d'exploitation, bibliothèques, applications), tous les conteneurs basés sur cette image en hériteront.
  - L'**isolation des processus** : Bien que les conteneurs isolent les processus, cette isolation n'est pas une barrière absolue. Des failles dans le noyau ou la configuration peuvent permettre une "évasion de conteneur".
  - L'**accès au système hôte** : Lorsque des conteneurs partagent des volumes ou doivent accéder à des ressources de l'hôte, des configurations incorrectes peuvent créer des points d'entrée pour des attaques.
  - Le **cycle de vie des conteneurs** : La sécurité doit être intégrée à chaque étape, de la construction de l'image à la suppression du conteneur.
- **Les risques liés à l'exécution de conteneurs en tant que root (environ 6 minutes) :** Un point particulièrement sensible est l'exécution de processus en tant qu'utilisateur root à l'intérieur du conteneur. Bien que ce root soit isolé par les namespaces, il présente des risques :  
  - L'**escalade de privilèges** : Si un attaquant compromet un processus root dans le conteneur, il a déjà un niveau d'accès élevé dans cet environnement.
  - L'**impact en cas de compromission** : Si une vulnérabilité permet à cet attaquant de s'échapper du conteneur, ses privilèges root internes peuvent faciliter l'obtention d'un accès root sur le système hôte, avec des conséquences potentiellement désastreuses.
- **Présentation des différentes stratégies de sécurité (environ 5 minutes) :** Pour atténuer ces risques, plusieurs stratégies existent :  
  - Le **principe du moindre privilège** : N'accorder que les droits strictement nécessaires à chaque processus.
  - Le **hardening des images** : Minimiser la taille des images et supprimer les composants inutiles.
  - Les **Network Policies** : Contrôler le trafic réseau entre les conteneurs.
  - Les **Security Context Constraints (SCC) dans OpenShift** : Un mécanisme puissant pour définir les politiques de sécurité des pods, et que nous allons explorer plus en détail.
  - Exemple de Network Policy Kubernetes pour isoler les conteneurs :  

```yaml
apiVersion: networking.k8s.io/v1  
kind: NetworkPolicy  
metadata:  
  name: isolate-nginx  
spec:  
  podSelector:  
    matchLabels:  
      app: nginx  
  policyTypes:  
  - Ingress  
  ingress:  
  - from:  
    - podSelector:  
      matchLabels:  
        app: my-app
```

## II. Le Concept de Conteneurs Rootless (environ 40 minutes)

Maintenant que nous avons posé les bases, entrons dans le vif du sujet : les conteneurs rootless. C'est une approche élégante pour renforcer la sécurité en s'attaquant directement au problème des privilèges root.

- **Qu'est-ce qu'un conteneur rootless ? (environ 10 minutes) :** Un conteneur rootless exécute les processus à l'intérieur du conteneur avec un utilisateur **non-root**. Comment est-ce possible alors que certaines opérations semblent nécessiter des droits élevés ? La clé réside dans l'utilisation astucieuse des **User Namespaces**. Rappelez-vous, ils permettent de mapper des **UID** (User Identifier) et des **GID** (Group Identifier) internes au conteneur vers des UID/GID externes sur l'hôte. Ainsi, un processus qui s'exécute avec l'UID 0 (traditionnellement root) à l'intérieur du conteneur peut en réalité correspondre à un UID non privilégié sur la machine hôte. C'est une véritable translation d'identité qui se fait au niveau du noyau.  

- **Les avantages de l'approche rootless (environ 10 minutes) :** Pourquoi adopter cette complexité ? Les bénéfices en termes de sécurité sont substantiels :  
  - **Réduction de la surface d'attaque** : Si un attaquant compromet un processus dans un conteneur rootless, même s'il obtient les privilèges de l'utilisateur interne (qui pourrait être l'UID 0 _dans le namespace_), cet utilisateur n'aura pas les super-pouvoirs du véritable root sur l'hôte.
  - **Isolation renforcée** : Les User Namespaces créent une barrière de sécurité supplémentaire. Une évasion de conteneur se traduira par un accès avec les privilèges d'un utilisateur lambda sur l'hôte, rendant l'escalade de privilèges beaucoup plus ardue.
  - **Limitation des privilèges internes** : L'approche rootless encourage à concevoir des applications qui fonctionnent sans nécessiter de privilèges root, même à l'intérieur du conteneur, ce qui est une excellente pratique en soi.
- **Les composants clés pour le rootless (environ 10 minutes) :** Plusieurs éléments rendent possible le fonctionnement des conteneurs rootless :  
  - **userd (ou slirp4netd)** : Pour gérer le réseau. Les conteneurs rootless ne peuvent pas lier directement les ports inférieurs à 1024. Ces outils permettent de faire du forwarding de ports non privilégiés vers les ports souhaités à l'intérieur du conteneur, sans droits root sur l'hôte.
  - **Stockage rootless** : La gestion des systèmes de fichiers nécessite des solutions adaptées comme overlayfs qui peut fonctionner en mode non privilégié pour gérer les couches des images et les volumes.
  - **Gestion des processus par l'utilisateur** : C'est l'utilisateur non privilégié qui lance et gère les processus à l'intérieur du conteneur, sous son identité traduite dans le User Namespace.
  - Exemple de configuration de Podman pour le mode rootless : (Note : la configuration exacte peut varier selon les versions de Podman) :

```shell
# /etc/subuid et /etc/subgid  
# Exemple :  
# user1:10000:65536  
# user1:10000:65536  
$ podman run --rm -it -p 8080:80 nginx:alpine
```

- **Rootless vs. Privileged Containers (environ 5 minutes) :** Pour bien saisir l'intérêt, comparons avec leur opposé : les **privileged containers**. Ces derniers sont lancés avec des capacités Linux étendues et un accès important aux ressources de l'hôte, contournant de nombreuses protections. Ils sont risqués en cas de compromission. Les conteneurs rootless, au contraire, minimisent les privilèges et renforcent l'isolation.  
    Exemple de définition de pod Kubernetes avec un privileged container :

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: privileged-container  
spec:  
  containers:  
  - name: my-container  
    image: my-image  
    securityContext:  
      privileged: true
```

- **Construction d'Images Rootless (Information) (environ 5 minutes) :** Même si nous ne pourrons pas le pratiquer directement dans Gitpod, il est crucial que vous compreniez comment construire des images optimisées pour le rootless. Cela implique généralement :  
  - D'ajouter un utilisateur non-root spécifique dans votre Dockerfile à l'aide de la commande adduser.
  - De s'assurer que l'application s'exécute sous cet utilisateur en utilisant la directive USER.
  - De gérer correctement les permissions des fichiers et répertoires à l'intérieur de l'image pour que cet utilisateur non-root puisse y accéder et y écrire si nécessaire (souvent à l'aide de chown pendant la construction de l'image).
  - **\[Inclure ici un lien vers un tutoriel externe de qualité sur la construction d'images rootless avec Podman, par exemple\]**
  - Exemple de Dockerfile pour construire une image rootless :

```dockerfile
FROM alpine:latest  
RUN adduser -u 1001 myuser  
WORKDIR /app  
COPY myapp /app  
RUN chown -R myuser:myuser /app  
USER myuser  
CMD \["./myapp"\]
```

## III. Mise en Pratique sur OpenShift 4.18 (environ 75 minutes)

Passons maintenant à la partie concrète. Nous allons explorer comment OpenShift 4.18 nous permet de travailler avec des concepts proches du rootless grâce à ses Security Context Constraints (SCC).

- **Prérequis (annoncé précédemment)**  

- **Exploration des Security Context Constraints (SCC) (environ 10 minutes) :** Les SCC sont des objets Kubernetes spécifiques à OpenShift qui définissent les permissions et les capacités qu'un pod peut demander et utiliser. Elles agissent comme des politiques de sécurité au niveau du cluster. Nous allons nous concentrer sur deux SCC importantes pour notre sujet :  
    1. **restricted** : C'est la SCC par défaut et la plus sécurisée. Elle interdit l'exécution en tant qu'UID 0 sur l'hôte, bloque l'escalade de privilèges et limite les capacités Linux disponibles.
    2. **nonroot** : Cette SCC est spécifiquement conçue pour les pods qui doivent s'exécuter en tant qu'utilisateur non-root. Elle exige que runAsNonRoot: true soit spécifié dans le securityContext du pod et que l'UID du conteneur soit différent de zéro.

Exemple de définition de SCC (simplifié) :

```yaml
apiVersion: security.openshift.io/v1  
kind: SecurityContextConstraints  
metadata:  
  name: restricted  
# ...  
  runAsUser:  
  type: MustRunAsNonRoot  
# ...
```

- **Exercice 1 (Obligatoire) : Déploiement d'un Pod simple avec le SCC restricted (environ 20 minutes)** :  
    1. Créez un fichier nginx-restricted.yaml avec la définition d'un déploiement Nginx simple (comme montré précédemment).
    2. Appliquez ce déploiement sur OpenShift avec oc apply -f nginx-restricted.yaml.
    3. Observez le statut du pod avec oc get pods et examinez les logs avec oc logs &lt;nom-du-pod&gt;.
    4. **Discussion :** Bien que l'image Nginx puisse potentiellement lancer des processus en tant que root _à l'intérieur du conteneur_, la SCC restricted en vigueur empêche l'exécution en tant qu'UID 0 sur le nœud hôte. OpenShift gère cela en arrière-plan. De plus, toute tentative d'action nécessitant des privilèges root sur l'hôte (comme la liaison à un port < 1024 sans configuration spécifique) serait bloquée par cette SCC.

Exemple de fichier nginx-restricted.yaml :

```yaml
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: nginx-deployment  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app: nginx  
  template:  
    metadata:  
      labels:  
        app: nginx  
    spec:  
      containers:  
      - name: nginx  
        image: nginx:latest
```

- **Exercice 2 (Obligatoire) : Déploiement d'un Pod configuré pour un utilisateur spécifique avec le SCC restricted (environ 25 minutes)** :  
    1. Créez un fichier nginx-nonroot-simu.yaml (comme montré précédemment) en ajoutant un securityContext au niveau du pod spécifiant runAsUser: 1001.
    2. Appliquez ce déploiement avec oc apply -f nginx-nonroot-simu.yaml.
    3. Vérifiez que le pod démarre correctement.
    4. **Discussion :** Ici, nous forçons l'exécution du conteneur avec l'UID 1001 sur l'hôte, renforçant le principe du moindre privilège. La SCC restricted autorise cela tant que l'UID n'est pas 0 et que d'autres contraintes sont respectées.

Exemple de fichier nginx-nonroot-simu.yaml :

```yaml
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: nginx-deployment  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app: nginx  
  template:  
    metadata:  
      labels:  
        app: nginx  
    spec:  
      securityContext:  
        runAsUser: 1001  
      containers:  
      - name: nginx  
        image: nginx:latest
```

- **Exercice 3 (Obligatoire) : Déploiement avec le SCC nonroot et création d'une image non-root (simulation et discussion) (environ 20 minutes)** :
    1. **Partie A : Tentative de déploiement d'une image standard avec la SCC nonroot :** Modifiez le fichier nginx-restricted.yaml pour ajouter l'annotation security.openshift.io/scc: nonroot dans la section metadata du template. Appliquez la modification et observez l'échec du pod. Analysez les événements avec oc describe pod &lt;nom-du-pod&gt;. Vous devriez voir des erreurs liées au non-respect des exigences de la SCC nonroot (absence de runAsNonRoot: true ou tentative d'exécution en tant que root dans l'image).
    2. **Partie B : Discussion sur la création d'une image non-root :** Présentez l'exemple de Dockerfile pour une image non-root (comme montré précédemment). Expliquez l'importance de l'instruction USER et de la gestion des permissions. Soulignez que pour que la SCC nonroot soit pleinement respectée, l'image elle-même doit être conçue pour fonctionner avec un utilisateur non-root.

Exemple de modification du fichier nginx-restricted.yaml pour utiliser l'annotation nonroot :

```yaml
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: nginx-deployment  
spec:  
# ...  
  template:  
    metadata:  
      annotations:  
        security.openshift.io/scc: nonroot # Ajout de l'annotation  
# ...
```

## IV. Limitations et Considérations (environ 20 minutes)

L'adoption des conteneurs rootless est une avancée significative en matière de sécurité, mais elle vient avec son lot de considérations pratiques.

- **Compatibilité des applications (environ 5 minutes) :** Certaines applications héritées ou mal conçues peuvent nécessiter des privilèges root pour certaines opérations (écriture dans des répertoires spécifiques, installation de paquets, etc.). Il peut être nécessaire de les adapter, ce qui peut demander du temps et des efforts de développement.

Exemple de code d'une application nécessitant des privilèges root (à éviter) :

```python
import os  
# Ceci nécessitera root  
os.makedirs("/etc/config", exist_ok=True)
```

- **Gestion des ports (environ 5 minutes) :** Rappelez-vous que les conteneurs rootless ne peuvent pas lier directement les ports inférieurs à 1024. Des solutions de redirection de ports (via userd, slirp4netd ou des services comme les LoadBalancers et Ingress controllers) sont nécessaires, ce qui peut complexifier la configuration réseau.

Exemple d'utilisation d'un service Kubernetes pour exposer un conteneur rootless sur le port 80 :

```yaml
apiVersion: v1  
kind: Service  
metadata:  
  name: nginx-service  
spec:  
  selector:  
    app: nginx  
  ports:  
  - protocol: TCP  
    port: 80 # Port du service  
    targetPort: 8080 # Port du conteneur
```

- **Accès aux ressources hôtes (environ 5 minutes) :** L'accès à certaines ressources de l'hôte (périphériques, certaines fonctionnalités du noyau) peut être plus complexe en mode rootless et nécessiter des configurations spécifiques au niveau du moteur de conteneur et des SCC dans OpenShift pour garantir la sécurité.
- **Performances (environ 5 minutes) :** Dans certains cas très spécifiques impliquant des opérations d'E/S intensives, une légère surcharge de performance pourrait être observée avec les conteneurs rootless en raison de l'abstraction des User Namespaces. Cependant, pour la majorité des applications, l'impact est minime. Il est toujours bon de tester vos applications dans un environnement rootless pour valider les performances.

## V. Conclusion et Ressources (environ 15 minutes)

Pour conclure notre session, récapitulons les points essentiels et indiquons quelques pistes pour continuer votre apprentissage.

- **Récapitulatif des points clés (environ 5 minutes) :** Nous avons exploré les risques liés à l'exécution de conteneurs en tant que root et l'intérêt de l'approche rootless pour renforcer la sécurité. OpenShift, avec ses SCC, nous offre des outils puissants pour appliquer ces principes. Nous avons même mis la main à la pâte avec quelques exercices.
- **Importance de l'approche rootless dans une stratégie de sécurité globale (environ 5 minutes) :** L'adoption du rootless est une étape importante, mais elle doit s'inscrire dans une stratégie de sécurité plus large, incluant la sécurisation des images, la gestion des politiques réseau et le monitoring continu.
- **Liens vers la documentation et les ressources (environ 5 minutes) :** Pour aller plus loin, je vous encourage vivement à consulter :
  - La documentation officielle d'OpenShift sur les SCC et l'exécution de pods en tant qu'utilisateur non-root.
  - Des tutoriels et la documentation de Podman et Docker concernant la construction d'images rootless.
  - Les ressources de sécurité du CNCF et des blogs spécialisés dans la sécurité des conteneurs.

N'oubliez pas que la sécurité est un domaine en constante évolution. Restez curieux et continuez à vous informer. Si vous avez d'autres questions, n'hésitez pas. Bon courage dans votre parcours !
