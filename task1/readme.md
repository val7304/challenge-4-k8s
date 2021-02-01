
# Challenge 4 - Managing Workloads on Kubernetes

Date limite de remise du challenge: le 06 février 2021 

## Task 1:
L'objectif ici est de déployer l'application WordPress sur le cluster mono-noeud Docker Desktop Kubernetes. 

Pour obtenir l'application WordPress, se rendre sur ce lien ci https://docs.docker.com/compose/wordpress/ . Examiner le `docker-compose.yaml` proposé et démarrer le stack directement sur Docker afin de vérifier que l'application est opérationnelle.

**Travail demandé:** 
 - Rédiger un manifeste pour le déploiement et un autre pour un service du type loadbalancer. Utiliser une solution basique du type `hostpath` pour le volume. Procéder au déploiement et tester que l'application est opérationnelle. Commiter avec Git cet état.
----
**Exécution de la task1-A:**
Création des fichiers, 1 pour mySql, 1pour Wordpress, en vue d'une installation basique.

-- Installation du cluster mono-nœud: `kubectl create -f ./`
ps: J'ai nommé les fichiers de sorte que le mysql.yaml démarre en premier, puisqu'il faut d'abord installer la base de données.

Pour revenir sur le commit du premier état:
`git log`  ensuite `git checkout IDduCommit`
ou via VSCod, Git Graph
 
**Connexion au site:** 
Page View: http://localhost
Page Admin: http://localhost/wp-admin/
login & mdp sont ceux que vous avez spécifiez lors de l'installation.

**Configuration originelle:** 
- wordpress: 4.8-apache
- mysql:5.6

---
 - Améliorer votre solution en utilisant un `volume` Kubernetes pour la base de données, des `ConfigMap`s et des `Secrets` pour les paramètres et mots de passes de l'application. Commiter avec Git ce nouvel état.
----
**Exécution de la task1-B:**
-- Changer le commit si besoin pour voir l'état changé

**Changement des configurations:**
- mySql:  
-- **Ajout d'un secret** pour la base de données root, le mot de passe est `password`

- Wordpress: 
-- màj version : `wordpress:5.6.0-php7.3-apache`
-- **Ajout de configMaps** pour la définition de/du: 
-1. nom de la base de données: `wordpress_web`
-2. préfixe des tables wordpress_web `K8S_`

**Vérification des nouveaux paramètres:** 
Comme nous ne savons pas le vérifier depuis le wiziwig de wordpress il nous faut nous connecter à un pod après avoir runner les fichiers avec: 
`kubectl create -f ./` 

**loadbalancing** : se situe sur l'installation de Wordpress

`kubectl get po`
`kubectl delete po IdDuPods`
`kubectl get po` pour le voir se régénérer avec un autre ID
Il se régénérera tant que nous n'aurons  pas supprimer les déploiements
Vous pouvez aussi le voir en faisant: 

    PS D:\challenge-4\task1> kubectl get svc 
    NAME              TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
    kubernetes        ClusterIP      10.96.0.1        <none>        443/TCP        37h
    wordpress         LoadBalancer   10.102.202.169   localhost     80:32044/TCP   17m
    wordpress-mysql   ClusterIP      None             <none>        3306/TCP       17m

**Mysql**:  
voir le secret: `kubectl describe secret mysql-pass`
Vous verrez  qu'il est bien là mais vous n'y aurez pas accès.
Pour le décoder, exécutez cette commande depuis un terminal bash: 

    $ echo "cGFzc3dvcmQ=" | base64 --decode
    > password
Vous aurez l'occasion de l'encoder plus tard, en clair cette fois ;) 

1. **nom de la base de données et préfixe des tables wordpress_web**  table: `wordpress_web` et préfixe des tables: `K8S_`: 
Depuis un terminal depuis votre racine du dossier: 
`kubectl get po`
on saute dedans:
`kubectl exec -it wordpress-mysql-idDuPod -- /bin/bash`
on se connecte à la base de données: 
`mysql -u root mysql -p`
`> password:` password			
	
`#the secret is applied!
       
	mysql> show databases;	#the configmap var wordpress.db.name wordpress_web is applied 
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| performance_schema |
	| wordpress_web      |
	+--------------------+
	4rows in set (0.00 sec)
	
	mysql> use wordpress_web; 
	mysql> show tables; 
	#the configmap var: wordpress.db.prefix | $table_prefix "K8S_" is applied
    +-------------------------+
    | Tables_in_wordpress_web |
    +-------------------------+
    | k8s_commentmeta         |
    | k8s_comments            |
    | k8s_links               |
    | k8s_options             |
    | k8s_postmeta            |
    | k8s_posts               |
    | k8s_term_relationships  |
    | k8s_term_taxonomy       |
    | k8s_termmeta            |
    | k8s_terms               |
    | k8s_usermeta            |
    | k8s_users               |
    +-------------------------+
    12 rows in set (0.00 sec)
    mysql> exit
    Bye
    root@wordpress-mysql-6c479567b-x59rw:/# exit
    exit

**Vérification des commits:**
- via un terminal:
```console
git log --pretty=oneline
```
ou git graph dans VScode

**delete app:**


    #suppression des déployements
    kubectl delete deploy --all
    
    #suppression des secrets et des configurations
    kubectl delete secret mysql-pass
    kubectl delete configmap wordpress-configmap
    
    #suppression des services
    kubectl delete svc wordpress
    kubectl delete svc wordpress-mysql
    
    #suppression des volumes
    kubectl delete pvc --all
    #et si nécessaire:
    kubectl delete pv --all

*Glad I made this challenge ;)* 
