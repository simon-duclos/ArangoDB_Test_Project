
Projet NFE204 
ArangoDB  
Juin 2020 

Table des matières
1.	Introduction :  
2.	Présentation d’ArangoDB :	  
a.	Historique  
b.	Une base de données “Multiparadigmes” : vraiment ?  
c.	Moteur RocksDB  
d.	Installation  
e.	Architecture du mode cluster  
f.	Communiquer et interagir avec ArangoDB  
g.	Objets ArangoDB et aspects techniques NoSQL 
h.	Langage de requête AQL  
3.	Données de migration interrégionales dans ArangoDB :  
a.	Présentation des données	  
b.	Création de la base de données  
c.	Insertion des données dans la base  
d.	Visualisation de graphe	  
e.	Requêtes sur graphe	  
f.	Recherche d’information	  
4.	Comparaison avec Neo4J, la référence des bases de données graphes	  
5.	Conclusion	  
6.	Annexes	  

 
1.	Introduction : 

Souhaitant a priori étudier une base de données orientée graphe, j’ai choisi ArangoDB qui n’est pas très connue et qui a visiblement été peu étudiée. Cette base a la particularité d’être multi-paradigmes, c’est à dire qu’elle n'est pas uniquement dédiée aux graphes (même si cela semble être son utilisation la plus courante).
Dans ce rapport, je commencerai par présenter ArangoDB, ses différentes fonctionnalités. Je présenterai ensuite les différents moyens d’interagir avec l’outil dont le langage de requêtage AQL, les différentes configurations possibles en matière de partitionnement, réplication.
Dans une seconde partie, je présenterai les données de mobilité régionale que je souhaite étudier, et je montrerai les différentes étapes d’intégration des données. Ensuite je testerai différentes requêtes et leur visualisation sous forme de graphe, et pour finir je présenterai plusieurs fonctionnalités de recherche d’information.


 
2.	Présentation d’ArangoDB :

a.	Historique

Crée en 2011 en Allemagne sous le nom d’Avocado DB (dont s’inspire toujours le logo), la base de données a été renommée ArangoDB en 2012. En 2014, l’entreprise s’implante aussi dans la Silicon Valley.
Dès son début, l’outil a été construit autour d’un business model assez répandu dans les entreprises technologiques américaines : produire une version gratuite sous licence open-source (Apache 2.0), disposant de nombreuses fonctions - nommée ArangoDB Community – et un ensemble de services et fonctions avancées disponibles moyennant des licences mensuelles : ArangoDB Enterprise. Une version hébergée est aussi commercialisée sous le nom d’ArangoDB Oasis.

b.	Une base de données “Multiparadigmes” : vraiment ?

ArangoDB est présentée par ses concepteurs (et par le discours commercial de l’éditeur) comme une base “multi-paradigme” : clé-valeur, document, graphe et recherche.
En pratique, ArangoDB est avant tout une base de données qui stocke des documents JSON. Elle est sans-schéma imposé ni vérification de colonnes ni typage (“schéma-less”). 
Il est possible de spécifier une clé (_key) ou pas lors de la création du document. En l’absence de clé, la base génère elle-même la clé. Du coup, il est possible d’insérer des lignes et d’obtenir des clés en retour, ce qui rappelle le fonctionnement d’une base purement clé-valeur. Cela dit, ArangoDB déconseille ce fonctionnement en clé/valeurs sur des fortes volumétries, ainsi que le stockage de valeurs sous forme d’objet binaires (BLOB), ce qui limite le champ des usages.
Il est possible, et c’est même le cas d’usage le plus mis en avant, d’utiliser ArangoDB comme base de données graphes, où le modèle de données est généralement décrit comme “sommet-nœud". Dans ce cas, les nœuds sont stockés comme des documents (JSON donc), dans une ou plusieurs collections (l’équivalent des tables relationnelles), les sommets comme d’autres documents (JSON aussi du coup) identifiés comme un type de document particulier (edge document) et pour lesquels les champs _from et _to sont obligatoirement non vides. Cette contrainte est d’ailleurs la seule qui constitue une ébauche de schéma dans l’outil.
En présence de documents nœuds et sommets, il est possible de créer un graphe associant les deux. Il faut passer par des instructions spécifiques ou par l’interface web et bien sûr ici les graphes - comme aussi tous les objets systèmes - sont stockés en base sous forme de document JSON (collection système _graph visible avec les droits d’admin).




c.	Moteur RocksDB

Le moteur de stockage original d’ArangoDB, nommé mmfiles – a été déprécié puis abandonné dans les versions récentes. Le moteur actuel est RocksDB, un projet géré et “open-sourcé” par Facebook, lui-même une fourche de LevelDB de google.
RocksDB est une base de données purement clé-valeurs de “bas-niveau”, inutilisable en tant que tel sans de grosses compétences techniques. 
Cela permet de comprendre que le travail de développement d’ArangoDB se concentre donc sur la partie fonctionnalités et usages de la base de données plutôt que sur le bas niveau. 

d.	Installation

Il est possible d’installer ArangoDB de multiples façons : 
•	En local, sur serveur physique ou hébergé en téléchargeant le programme d’installation sur le site https://www.arangodb.com/download il faut installer séparément le serveur et les utilitaires. Sur mon PC équipé de Windows 10 Home, l’installation s’est passée sans difficulté.
•	Sur Docker, deux machines prêtes à l’emploi sont disponibles [arangodb et arangodb-starter] et faciles à faire tourner car bien documentées. Tout est préinstallé, serveur et utilitaires. Le port par défaut est le 8529. Comme il n’y a pas d’outils en client lourd, l’utilisation de docker apporte les mêmes fonctionnalités que l’installation locale. 
•	L’éditeur commercialise un service de cloud managé, nommé ArangoDB Oasis.

Il est tout à fait possible de faire tourner ArangoDB en mode stand-alone, avec une seule machine. Il faut alors lancer simplement l’exécutable arangod ou lancer l’image docker comme suit :
docker pull arangodb
docker run -e ARANGO_ROOT_PASSWORD=monmdp -p 8529:8529 -d --name arangoNFE204 arangodb

Avec plusieurs serveurs, plusieurs types de configuration sont possibles :
•	Serveurs indépendants qui ne communiquent pas (mode nommé « single »)
•	Mode « active failover », où on trouve alors un serveur maître et plusieurs serveurs esclaves qui peuvent remplacer celui-ci en cas de panne.
•	Mode “cluster”, c’est à dire que la distribution sur les nœuds -réplication, partitionnement par exemple- est géré par une “agence” (agency) constituée d’un nombre impair d’agents (au moins 3) sous un principe de consensus (Consensus de Raft).
•	Mode maître-esclave
•	Mode réplication entre datacentres (non testé dans le cadre de ce projet).

Regardons de plus près le mode cluster. Celui-ci fonctionne à partir de 3 serveurs ensemble ; il suffit de créer 3 sous-répertoires dans le répertoire d’installation du serveur puis de lancer 3 serveurs en indiquant une adresse IP de contact unique. 
Pour l’installation en mode cluster, il y a deux façons de procéder :
•	ArangoDB propose un mode nommé “starter”, relativement facile à configurer qui gère tout seul les différents composants. C’est ce mode-dont je décris ci-bas l’installation-que j’ai utilisé pour ce projet, qui m’a permis de tester les différents modes de réplication.
•	Par opposition au mode starter, ArangoDB désigne par mode “manuel” l’installation où il faut configurer plus en détail machine par machine. On peut alors choisir les adresses des serveurs, le mode d’authentification, les serveurs qui jouent le rôle d’agence ou non.
Voici le code que j’ai exécuté sous Windows pour l’installation en mode starter : ayant créé trois répertoires S1, S2, S3, il faut lancer dans 3 terminaux (cmd = invites de commande)  le programme arangod en mode administrateur avec les commandes suivantes :
arangodb  --starter.data-dir S1
arangodb --starter.join 127.0.0.1 --starter.data-dir S2
arangodb --starter.join 127.0.0.1 --starter.data-dir S3 

Les serveurs se lancent, par exemple le serveur 3 :
 
Dans l’interface web, onglet « node », les 3 serveurs sont visibles :
 
Dans chaque répertoire S1, S2, S3 sont créés des sous-répertoires contenant les données et la configuration. L’arborescence est claire, ça vaut le coup d’être signalé :
 

Pour l’installation dans docker, il faut spécifier des ports différents (au lieu des répertoires) sur 3 machines dockers différentes et utiliser l’image docker-starter dédiée [nom de l’image : arangodb-starter] [Pour les instructions docker, voir en annexe 1].
La suite du projet, y compris l’analyse des données de mobilité régionales, repose sur une installation en mode cluster sur 3 serveurs tournant en parallèle sur la même machine -bien sûr, il s’agit d’une configuration de test. L’installation sur plusieurs machines au sein du même réseau se déroule néanmoins de la même façon que sur une seule machine.
e.	Architecture du mode cluster

En mode cluster, différents composants d’ArangoDB interagissent :
•	Un nombre impairs d’agents forment l’agence (Agency). L’agence contient la configuration du cluster et s’occupe de gérer la cohérence des données. Un mécanisme de consensus est mis en place basé sur le protocole RAFT. L’agence n’a pas besoin de forte ressource mémoire pour fonctionner et peut fonctionner sur des serveurs moins performants que les autres composants.
Par défaut implémenté sur le port 8531 [+10*n, où n = nombre d’agents], on peut connaître l’état des agents en tapant la commande : curl GET localhost:8531/_api/agency/config 

Exemple avec trois serveurs en mode starter (la configuration de notre projet, voir chapitre suivant), les agents sont sur les ports 8531,8541 et 8551.
 

•	Les coordinateurs assurent l’interface avec les clients dont les applications qui ont besoin d’accéder aux données. Par défaut implémenté sur le port 8530 [+10*n, où n = nombre de coordinateurs], on peut connaître l’état des coordinateurs en tapant la commande : 
curl GET http://localhost:8530/_db/DB_MOBREGI/_admin/server/role 

•	Les serveurs de bases de données stockent les données. Par défaut implémenté sur le port 8529 [+10*n, où n = nombre de coordinateurs], on peut connaître l’état des serveurs en tapant la commande : 
curl GET http://localhost:8529/_db/DB_MOBREGI/_admin/server/role 

Remarque : il y a aussi bien sûr la notion de « shard », c’est-à-dire une unité de stockage des éléments des collections. Un shard contient tout ou partie des collections et des répliques. Si le facteur de réplication des bases ou collections est différent du nombre de serveurs, le nombre de shards et nombre de serveurs de bases de données diffèrent. Dans ce projet, je fais tourner 3 instances de serveurs et un facteur de réplication de 3. Chaque shard a un serveur principal et 2 serveurs de réplication.
Dans le mode starter, les serveurs de base de données et coordinateurs sont les mêmes, et les trois premiers serveurs lancés accueillent les agents.
En mode manuel, de nombreuses configurations sont possibles : d’avoir plus de coordinateurs que de serveurs de base de données ou l’inverse, de faire tourner les coordinateurs sur un serveur applicatif pour être au plus près des clients.
f.	Communiquer et interagir avec ArangoDB

i.	Interface web
L’installation d’ArangoDB s’accompagne d’un serveur web et donc d’une interface accessible via le port 8529 (http://localhost:8529 en local par défaut) et permettant un certain nombre d’opérations d’administration (bien sûr, il faut avoir les droits), de créer et modifier des documents, collections, de gérer les utilisateurs. Étonnamment l’interface est assez incomplète : certaines fonctionnalités ne sont pas possibles alors que l’interface le suggère pourtant : par exemple impossible d’ajouter ou de modifier directement les graphes (troublant : il existe pourtant un bouton créer, mais quand l’utilisateur clique dessus, le bouton ne lance rien même avec des droits d’admin...). Il manque aussi la possibilité de supprimer certains objets (collections, bases) alors qu’il est possible de les créer depuis cette même interface...
L’interface web permet de taper des requêtes dans le langage propriétaire AQL, lesquelles permettent d’interroger les collections, documents, graphes, vues, mais aussi de lancer la plupart des opérations d’administration (et aussi la création de graphes). Nous reviendrons plus tard sur le langage AQL.
ii.	Arangosh : le shell arango
L’installation d’ArangoDB vient avec un certain nombre d’utilitaires (citons par exemple arangodump pour les reprises, arangobackup pour les sauvegardes, arangoimport pour les imports de données en masse) dont une interface en ligne de commande nommée arangosh permettant de communiquer avec la base de données via des instructions javascript, et aussi d’y encapsuler des requêtes AQL. Contrairement à l’interface web, toutes les opérations sont réalisables depuis arangosh (administration, ajout/modification/suppression d’objets...). Arangosh fonctionne en mode synchrone. Exemple de requêtes :
 
iii.	API REST
Il existe aussi une API REST permettant de communiquer avec la base de données en HTTP. Il faut alors d’abord récupérer un jeton (token) d’authentification à utiliser dans chaque requête. Une bonne documentation de l’API est disponible dans l’interface web avec possibilité de tester. Comme Arangosh, toutes les opérations sont réalisables via l’API. 
Exemple d’utilisation :
 

g.	Objets ArangoDB et aspects techniques NoSQL

i.	Bases de données 
Le concept de base dans ArangoDB correspond peu ou prou à son équivalent relationnel, c’est à dire un ensemble d’objets de stockage. L’interface web ne permet de se connecter qu’à une seule base à la fois (sur chaque fenêtre). La création/suppression peut se faire par arangosh ou API.
ii.	Collection
Correspond à une table en relationnel, mais en l’absence de schéma il est possible d’insérer toutes sortes de champs et de types différents à chaque ligne.
Il y a deux types de collections, à spécifier à la création et non modifiables ensuite : les collections document et les “sommets de graphes”, edge collections. En requêtage on trouve aussi des collections systèmes.
En mode cluster c’est au niveau de la collection et uniquement au moment de la création que sont définis [Voir annexe 2] :
•	Le partitionnement : le nombre de partitions (number of shards) juqu’à 1000, la clé de partitionnement (par défaut la clé du document _key qui peut être choisie au moment de l’insertion du document).
•	La réplication : le nombre de répliques choisies, jusqu’à 10.
•	La réplication acceptable : le nombre de répliques minimum pour que l’insertion de données soit jugée effectuée. Ce chiffre peut être inférieur au nombre de réplications défini précédemment, qui sera considéré comme un objectif “à terme”.
Pour modifier les paramètres après la création, le seul moyen est de créer une nouvelle collection avec des caractéristiques différentes et de copier les données dedans.
Remarque : il est possible de spécifier des valeurs de réplication/partitionnement lors de la création d’une base de données, mais c’est trompeur car il ne s’agit en fait que de valeurs par défaut lors de la création des collections ; c’est seulement au niveau de ces dernières que s’effectuent le sharding, donc la réplication et le partitionnement.
Il est possible et très vite nécessaire de créer des index au niveau des collections afin d’améliorer les performances. Par défaut, des index sont créés sur la clé _key ainsi que sur les champs _from et _to s’il s’agit d’une collection de sommets. Il y a plusieurs types d’index (pour le requêtage, pour la recherche d’information, géospatial).
iii.	Document
Equivalent à une ligne relationnelle, contient du JSON. Contient obligatoirement les champs _key (spécifié ou déterminé automatiquement par le système au moment de la création du document), un champ technique _rev (rev signifie révision, en fait c’est le hash de l’estampille de la dernière modification) et un _id technique aussi (concaténation du nom de la base et de la _key).
Les documents des collections “sommets” edge doivent contenir les champs _from et _to, destinés à référencer les id des nœuds des graphes. Les valeurs de ces champs doivent donc être de la forme “NOM_DE_COLLECTION/KEY_DU_NŒUD". Pour insérer des documents dans une collection edge, il faut au préalable avoir créé un graphe contenant la définition des nœuds et des sommets.
Lors de l’insertion de documents vers une collection “edge” se produit alors le seul contrôle de cohérence effectué par ArangoDB  : les documents sans les champs _from ou _to correspondant à des sommets sont automatiquement rejetés.
Exemple d’insertion de données via l’API REST avec des erreurs : :
 
Dans le premier cas, l’essai d’insérer une clé avec le caractère “é” provoque une erreur.
Dans le deuxième cas, l’erreur provient de l’absence de champ “_from” et “_to”.

iv.	Graphe
Un graphe associe une collection document et une collection “sommet”. Ensemble, ces deux objets forment un graphe, qu’il est alors possible de requêter directement et sur lequel un certain nombre de fonctions s’appliquent.

v.	Vues – Arangosearch
Il est possible dans ArangoDB de créer des vues permettant d’améliorer la recherche d’information, en particulier la recherche en plein texte. Comme dans ElasticSearch, les vues ne sont pas uniquement des requêtes mais sont stockées. La cohérence avec les documents se fait “à terme”, c’est à dire qu’il peut théoriquement y avoir une différence entre les données renvoyées par les vues et les collections peuvent différer temporairement.
Comme leur homonyme relationnel, les vues sont requêtables directement. 

h.	Langage de requête AQL
S’il s’agit bien d’un langage de requêtage (d’où le “query language”), il n’y a que très peu de ressemblance orthographique avec les langages déclaratifs SQL des bases de données relationnelles ! Les fonctionnalités sont similaires (requêtage, modification, ajout de données, transactions (enfin une ébauche plutôt)), il y a aussi des fonctions spécifiques à la recherche d’informations et d’autres spécifiques aux graphes.
Sa syntaxe le rapproche plutôt de la programmation impérative. 
Point capital : il est possible d’effectuer des jointures nativement et sans difficulté au moyen de boucles for imbriquées.
Une requête commence par un FOR et termine par un RETURN, on peut ajouter un FILTER :
FOR my IN COL_MOBREGI_FULL
FILTER my._id == "COL_MOBREGI_FULL/109644325"
RETURN my 

Voici une jointure (simple et rapide) :
FOR s IN MOB_REGI_SMALL
    FOR v1 IN MOB_REGI_VILLES
        For v2 IN MOB_REGI_VILLES
    FILTER s._from == 'MOB_REGI_VILLES/C93048' AND s._from == v1._id AND s._to == v2._id
    RETURN { Origine : v1.nom, Destination : v2.nom, Nombre : floor(s.nombre) } 
 
Une requête avec jointure et insert :

FOR s IN MOB_REGI_SMALL
    FOR v1 IN MOB_REGI_VILLES
        For v2 IN MOB_REGI_VILLES
    FILTER s._from == v1._id AND s._to == v2._id
    INSERT {_from : s._from, _to : s._to,nombre: s.nombre, demenagement : concat(floor(s.nombre)," personnes ont déménagé de ",v1.nom," vers ",v2.nom," en 2016 } INTO MOB_REGI_SMALL_DEM 
Les requêtes spécifiques aux graphes et à la recherche d’information seront présentées plus bas.




 
3.	Données de migration interrégionales dans ArangoDB :
a.	Présentation des données

L’INSEE publie à l’occasion de chaque recensement les données dites de mobilité résidentielles, présentant les déménagements vers les communes françaises. Depuis 2013, ces données sont collectées à l’aide des recensements partiels effectués chaque année. Il y a plusieurs jeux de données disponibles avec une granularité plus ou moins faible (il n’y a pas de fichier unique car pour des raisons d’anonymisation, certaines données ne sont disponibles qu’à l’échelle du département et pas au niveau des communes de moins de 5000 habitants).
J’ai choisi les déménagements au niveau de chaque commune.
La description des champs est disponible à l’adresse suivante https://www.insee.fr/fr/statistiques/fichier/4171543/contenu_RP2016_migcom.pdf .
Dans le cadre de ce projet, j’ai voulu me concentrer sur la partie “mobilité” au sens de déménagement, je n’ai donc retenu que les champs suivants :
Dans le fichier Varmod_MIGCOM (Référence INSEE des découpages géographiques et en particulier des communes) :
Nom du champ	Libellé
COD_MOD	Code INSEE de la commune française ou modalité "99999".
LIB_MOD	Nom de la commune. La modalité "99999" correspond à "Etranger".

Dans le fichier FD_MIGCOM_2016 (Fichier contenant les “faits” : les migrations d’une commune à une autre) :
Nom du champ	Libellé
COMMUNE	Code INSEE de la commune française sans arrondissement ou modalité "99999" du lieu de résidence au moment du recensement (après déménagement s’il y a lieu)
Attention, ne correspond pas au code postal !
ARM	Code INSEE de l’arrondissement pour Paris, Lyon, Marseille au moment du recensement (après déménagement s’il y a lieu). 
DCRAN	Code INSEE de la commune française avec arrondissement ou modalité "99999" du lieu de résidence au 1er janvier de l’année précédent le recensement (avant déménagement s’il y a lieu)
IPONDI	Nombre de personnes. Ce chiffre contient 10 décimales car il s’agit d’une pondération statistique et non une valeur exacte depuis que le recensement s’effectue de manière partielle (2013) 





b.	Création de la base de données

Voici la syntaxe utilisée dans Arangosh pour créer la base « DB_MOBREGI » :
arangosh> require("@arangodb").db._createDatabase("DB_MOBREGI", {replicationFactor:3, writeConcern:3} ,[{ username: "SDU",passwd: "mon-mdp",active: true }]);
true
Comme précisé au chapitre d’avant, un facteur de réplication de 3 est choisi, pour avoir autant de shards que de serveurs. On utilisera un writeConcern de 3 pour avoir une réplication immédiate et non « à terme ».
Plaçons-nous dans la base de données nouvellement créée et ajoutons ensuite la collection qui contiendra les villes de départ et d’arrivée. Comme le but est de s’en servir pour stocker les arêtes du graphe, on créera une collection de type « sommets ». On aurait aussi pu la créer via l’interface web.
arangosh> require("@arangodb").db._useDatabase("DB_MOBREGI");
arangosh> require("@arangodb").db._createEdgeCollection("COL_MOBREGI_DEM")   ;
true 

Ajoutons ensuite la collection de référence des villes. On utilise un facteur de partionnement de 3 (nous regarderons plus tard ce point) :
arangosh> require("@arangodb").db._create("COL_MOBREGI_VILLES",{ numberOfShards : 3})   ;
true

c.	Insertion des données dans la base

Effectuons ensuite l’insertion des données des fichiers csv vers la base de données créée.
Avec l’utilitaire arangoimport l’insertion des données de déménagement s’effectue sans problème et prend un peu plus de 1h50 sur mon ordinateur (C’est long ! Mais il ne s’agit pas de 3 serveurs en parallèles mais 3 instances sur la même machine, on est donc limité par le flux en écriture. Avec une seule instance de serveur et non 3, l’opération prend 25 minutes environ). 
Une fois le fichier chargé en totalité la collection compte alors 19 181 873 documents. La collection est créée lors de l’import (--create-collection true).
Windows cmd (admin)> arangoimport --file D:\FD_MIGCOM_2016.csv --type csv --separator ";" --collection "COL_MOBREGI_RAWIMPORT" --server.database DB_MOBREGI --create-collection true 
 
 
Ensuite créons un index dans arangosh pour permettre d’accélérer la requête d’après. Ici ça n’est pas franchement nécessaire – l’index met 5 minutes à se créer et on en gagne 2 sur la requête – mais illustratif !
arangosh> require("@arangodb").db.COL_MOBREGI_RAWIMPORT.ensureIndex({ type: "persistent", fields: [ "DCRAN","ARM","COMMUNE" ], unique: false }); 
 

Une fois les données chargées dans COL_MOBREGI_RAWIMPORT, on les transforme pour ne plus conserver que les champs correspondants à la ville de départ et à la ville d’arrivée grâce à la requête AQL suivante (effectuée dans l’interface web) :
FOR mob IN COL_MOBREGI_RAWIMPORT
    COLLECT from = LENGTH(mob.DCRAN)==4 ? CONCAT("COL_MOBREGI_VILLES/C0",mob.DCRAN) : CONCAT("COL_MOBREGI_VILLES/C",mob.DCRAN),
            to = mob.ARM == 'ZZZZZ' ? (LENGTH(mob.COMMUNE)==4 ? CONCAT("COL_MOBREGI_VILLES/C0",mob.COMMUNE) : CONCAT("COL_MOBREGI_VILLES/C",mob.COMMUNE)):CONCAT("COL_MOBREGI_VILLES/C",mob.ARM) 
    AGGREGATE Nombre = SUM(mob.IPONDI)
    SORT from DESC
    INSERT {
        _from :from,
        _to:to,
        Nombre
    } INTO COL_MOBREGI_DEM 

Quelques explications :
Sur le retravail des colonnes : Le champ « identifiant de la ville de départ » correspond à la colonne DCRAN ; l’« identifiant de la ville d’arrivée » à la colonne COMMUNE. Pour Paris, Lyon, Marseille il est préférable d’avoir l’identifiant de l’arrondissement indiqué dans la colonne ARM. 
Et pour les identifiants des communes des départements 01 à 09, le champ est détecté comme entier et non texte par ArangoDB, qui supprime donc de fait le « 0 » en première position. On rajoute donc un 0 dans ce cas-là dans les champs _from et _to.
Au final, le champ « identifiant de la ville de départ » s’appelle _from et « identifiant de la ville de départ » correspond à _to. Ces deux champs _from et _to sont requis pour les collections de sommets. Ils ont pour nomenclature « MOB_REGI_VILLES/C00000 » pour pouvoir correspondre (on dirait matcher en bon franglais) avec l’identifiant _id de la ville dans la collection MOB_REGI_VILLES-lequel est une concaténation du nom de la table et de la clé _key, laquelle est au format C00000.
Sur le formalisme AQL :
On reconnait bien sûr INSERT {liste des champs} INTO collection (la collection COL_MOBREGI créée précédemment). 
COLLECT va de pair avec AGGREGATE pour créer des regroupements et agrégations.
Il n’y a pas de « if then else » ou « case when » dans AQL (trop simple sûrement), mais à la place un opérateur « ternaire » de syntaxe (condition ? retour_si_oui : retour_si_non).

Voilà. Une fois réduit aux simples champs mentionnés plus tôt, la table “de fait” fait 481 979 documents [voir annexe 10 pour la requête], soit 39 fois moins que le fichier initial. Quant à la table de référence des communes, elle contient 35 368 documents, correspondant au nombre de communes françaises en date de la publication des chiffres du recensement.
On insère ensuite les données « villes » correspondant au csv ; voici les instructions arangoimport
Windows cmd (admin)> arangoimport --file D:\Varmod_MIGCOM_2016.csv --type csv --separator ";" --collection "COL_MOBREGI_VILLES_RAWIMPORT" --server.database DB_MOBREGI --create-collection true

Remarque : attention au format d’encodage des fichiers source avec arangoimport. Celui-ci doit être UTF8 sans BOM (Byte Order Mark). Parfois, l’insertion fonctionne bien avec BOM mais cela génère des erreurs quand on essaie d’insérer des champs systèmes (_key, _id, _from, _to). Le message d’erreur ne précise alors pas l’origine du problème (un octet de rajouté en début de fichier)…
 
Autre problème : pour le fichier précédent (Varmod_MIGCOM_2016.csv) inséré dans COL_MOBREGI_VILLES_RAWIMPORT, quand le fichier est enregistré en UTF8 avec BOM, un champ n’est pas requêtable dans ArangoDB une fois inséré, alors que le champ existe pourtant :
 
Autant le dire tout de suite, j’ai mis du temps à comprendre que le problème venait du format du fichier source…
Ensuite on insère les données de villes dans la table grâce au script AQL suivant :
FOR vil IN COL_MOBREGI_VILLES_RAWIMPORT
    FILTER vil.COD_VAR IN ["COMMUNE", "ARM"]
    INSERT {_key : (LENGTH(vil.COD_MOD)==4 ? CONCAT("C0",vil.COD_MOD) : CONCAT("C",vil.COD_MOD)), nom : vil.LIB_MOD} 
    INTO COL_MOBREGI_VILLES 
Remarquons un FILTER, l’équivalent AQL d’une clause WHERE. 

Profitons pour regarder l’architecture de réplication/partitionnement de la collection :
 
Ce tableau est disponible dans l’interface web, rubrique « Node ». 
On voit que la collection est partitionnée en 3 shards, chacun étant répliqué avec un facteur de 3 (un principal et deux répliques).
On peut aussi chercher à savoir sur quel shard se trouve un document de la collection. L’interface web ne permet pas de le faire, on peut utiliser l’API REST pour cela :
curl -X PUT --header 'accept: application/json' --data-binary @- --dump - http://localhost:8529/_db/DB_MOBREGI/_api/collection/COL_MOBREGI_VILLES/responsibleShard <<EOF
{"_key":"45040154"}
EOF
 
Nous observons que deux documents de clé contigüe ne sont pas sur le même shard. Précisons que j’ai fait une requête sur deux clés (_key) car par défaut le partitionnement s’effectue sur ce champ (c’est modifiable avec prudence).


d.	Visualisation de graphe

Avec les données de déménagement insérées dans une collection de type “sommets” et les données de référence (les villes) dans une collection document, nous pouvons créer un graphe pour analyser les déménagements.
Voici les instructions arangosh. L’opération retourne {[graph]}, ce qui confirme que le graphe est créé. 
arangosh> require("@arangodb").db._useDatabase("DB_MOBREGI");
arangosh> var graph_module = require("@arangodb/general-graph");
arangosh> var graph = graph_module._create("GRA_MOBREGI");
arangosh> graph._addVertexCollection("COL_MOBREGI_VILLES");
arangosh> var rel = graph_module._relation("COL_MOBREGI_DEM", ["COL_MOBREGI_VILLES"], ["COL_MOBREGI_VILLES"]);
arangosh> graph._extendEdgeDefinitions(rel);
arangosh> graph; 

Remarque : on aurait pu aussi créer le graphe dans l’interface web.
Explication : on se place dans la base de données DB_MOBREGI et on crée un graphe nommé GRA_MOBREGI. On lui associe une collection pour les nœuds COL_MOBREGI_VILLES, que l’on associe 2 fois (pour la commune de départ et d’arrivée) à la collection de sommets COL_MOBREGI_DEM.
Voici le résultat dans arangosh :
 

Nous pouvons maintenant visualiser le graphe dans l’interface web. Sans plus attendre, affichons le graphe que tout le monde meurt d’envie de voir : le graphe des déménagements depuis le 3e arrondissement de Paris où se situe le CNAM ! 
 

L’outil de visualisation de graphe permet de choisir un sommet puis de l’étendre. Il consomme beaucoup de mémoire vive, ce qui amène lenteurs et plantages sur mon ordinateur. Il faudrait avoir au moins un serveur applicatif séparé.
Ajoutons un attribut « département » dans la table COL_MOBREGI_VILLES grâce à une table AQL :
FOR vil IN COL_MOBREGI_VILLES
    UPDATE {_key : vil._key, departement : SUBSTRING(vil._key,1, 2) }
    INTO COL_MOBREGI_VILLES 

Et un attribut « classe_nombre » pour rassembler les déménagements de moins de 20 personnes, entre 20 et 100 et au-delà de 100 personnes.
FOR dem IN COL_MOBREGI_DEM
    UPDATE {_key : dem._key, classe_nombre : (dem.Nombre >= 20 ? (dem.Nombre>100 ? ">100" : "20-100") : "<20") }
    INTO COL_MOBREGI_DEM 

Il est possible de se servir de ces attributs dans le graphe, par exemple pour séparer les communes de destination par couleur en fonction du département ou encore ajouter des libellés sur les arêtes :
 

Pour un autre exemple de visualisation avec l’éditeur, voir en annexe.

L’éditeur AQL permet aussi de faire des requêtes sur le graphe et d’obtenir des résultats en JSON et leur visualisation :
 
e.	Requêtes spécifiques sur les graphes

AQL propose un certain nombre de fonctions permettant de suivre les parcours de graphes, les sommets et arcs visités.
Requête simple pour suivre tous les parcours de déménagements possibles entre deux villes via une autre, à l’aide des notions de “vertices”, “edge” et “path” dont les concepts existent en AQL :
 

Les fonctions les plus impressionnantes en termes de rapidité pour le parcours de graphe sont les plus courts chemins “shortest paths” et “K shortest paths”.
La requête suivante entre 2 villes prend 23ms alors qu’il y a 3 étapes, c’est vraiment très rapide.
 





La requête sur les k plus proches voisins prend plus de temps :
 

f.	Recherche d’information

ArangoDB propose des fonctionnalités de recherche d’information comme dans ElasticSearch par exemple. Il est possible de référencer du texte, d’appliquer tokenisation, racinisation, classement avec ou sans pondération par exemple.
Il faut pour cela créer une vue de recherche (view), et y annexer un document JSON qui spécifie les paramètres à appliquer. C’est possible via l’interface web, mais il existe dans Arangosh un certain nombre de fonctions préexistantes.
Les opérations sur texte se font via des “analyseurs”, des chaines de traitement qui permettent d’effectuer :
•	Rien, aucune modification : analyseur “identity”
•	Tokenisation : analyseur “delimiter”
•	Racinisation : analyseur “stem”
•	Tokenisation, racinisation par langue : “text_fr” ou “text_en” suivant la langue (il y en a d’autres)
Les étapes de recherche d’information dans ArangoDB sont les suivantes :
1.	Création d’une vue
2.	Création d’un lien liant les champs de la vue et l’analyseur à appliquer
3.	Appliquer le lien à la vue
4.	C'est fait ! La vue peut être requêtée

Afin de tester la recherche d’information sur notre jeu de données de mobilité résidentielle, nous allons créer une nouvelle collection de sommets.
arangosh> require("@arangodb").db._createEdgeCollection("COL_MOBREGI_RI")   ;

Puis un petit champ texte en AQL comme suit :
FOR s IN COL_MOBREGI_DEM
    FOR v1 IN COL_MOBREGI_VILLES
        FOR v2 IN COL_MOBREGI_VILLES
    FILTER s._from == v1._id AND s._to == v2._id
    INSERT {_from : s._from, _to : s._to,nombre: s.nombre, demenagement : concat(floor(s.nombre)," personnes ont déménagé de ",v1.nom," vers ",v2.nom," en 2016") } INTO COL_MOBREGI_RI 


 
Créons ensuite la vue dans arangosh:
arangosh > var v = require("@arangodb").db._createView("VUE_MOBREGI","arangosearch"); 


Puis un lien de recherche, qui référence tous les champs sans modification (par défaut, l’analyseur identity est appliqué, qui n’effectue aucune opération) sauf le champ “demenagement” auquel est appliqué l’analyseur “text_fr” qui effectue une tokenisation et une racinisation :
arangosh > var link = {
                       includeAllFields: true,
                       fields : { demenagement : { analyzers : [ "text_fr" ] } }
                    }; 

Appliquons le lien à la vue :
v.properties({ links: {COL_MOBREGI_RI: link }}) 

Arangosh renvoie ensuite les propriétés de la vue :
 
Vérifions que la vue fonctionne grâce à la requête AQL :
arangosh > db._query("FOR d IN VUE_MOBREGI COLLECT WITH COUNT INTO count RETURN count") 

 Il y a bien des documents dans la vue.
Nous pouvons maintenant tester notre vue, en effectuant une racinisation puis tokenisation :
 


Recherchons par exemple les déménagements de Montreuil vers/depuis Paris dans le champ déménagement (on aurait bien sûr pu faire une requête AQL avec LIKE sans vue) :
 
Surprise : le graphe n’est pas en étoile car il y a plusieurs Montreuil en France (et 20 arrondissements à Paris aussi !) et visiblement des déménagements de Montreuil-sur-Mer et Montreuil-Juigné vers/depuis Paris.
On peut aussi calculer le coefficient TF-IDF des documents de la vue, effectuer un classement :
 




 
4.	Comparaison avec Neo4J, la référence des bases de données graphes

Neo4J est incontestablement le leader des bases de données graphes dans le monde , et il va sans dire que son équipe marketing a réussi un travail impressionnant de référencement : toute requête dans un moteur de recherche parlant de base de données graphe vous parle rapidement de Neo4J, idem pour les vidéos. Les références clients sont bien plus nombreuses (d’ailleurs, je n’ai pas trouvé de références clients ArangoDB en France…). Cela se ressent sur le nombre de forums consacrés à Neo4J, beaucoup plus nombreux. Pour ArangoDB, j’ai pu trouver des réponses à mes questions (sur Stackoverflow principalement), mais les réponses datent d’il y a plus de 3 ans en général.
J’ai donc voulu comparer Neo4J avec ArangoDB en insérant les données de mobilité régionales. Après avoir essayé l’image docker, j’ai préféré utiliser Neo4J Desktop dont la prise en main est facile et bien documentée.
Il faut créer une base de données puis insérer les données du csv. Pour plus de simplicité, j’ai travaillé avec un csv identique à la collection COL_MOBREGI_VILLES : les villes. 
Le langage de requête s’appelle Cypher, voici la commande d’insert :
LOAD CSV with headers FROM 'file:/// COL_MOBREGI_VILLES.csv' AS row MERGE (v:ville {villeId: row.id}) 

On crée de même la table correspondant à la collection COL_MOBREGI_DEM : les déménagements.
LOAD CSV with headers FROM 'file:///COL_MOBREGI_DEM.csv' AS row 
MERGE (vf:v_from {ville_from: row.from}) 

LOAD CSV with headers FROM 'file:///COL_MOBREGI_DEM.csv' AS row 
MERGE (vt:v_to {ville_to: row.to}) 

LOAD CSV WITH HEADERS FROM 'file:///COL_MOBREGI_DEM.csv' AS row
MATCH (vf:v_from {ville_from: row.from})
MATCH (vt:v_to {ville_to: row.to})
MERGE (vf)-[d:DEMENAGE_VERS]->(vt)
RETURN count(*); 


On peut ensuite afficher le graphe des déménagements depuis le 3e arrondissement de Paris :
MATCH p=(n:v_from { ville_from: 'C75103' })-[r:DEMENAGE_VERS]->() RETURN p LIMIT 250 

 
La grosse différence entre Neo4J et ArangoDB repose sur le fait que Neo4J est une base de données orientée graphe, alors que Neo4J comporte une couche graphe sur une base de données de stockage de documents JSON. 
Cela se traduit concrètement par le fait qu’AQL est un langage de programmation assez généraliste, facile de prise en main car intermédiaire entre SQL et un langage impératif. Pour créer un graphe dans ArangoDB, il faut créer une collection de documents JSON sous un format spécifique.
 A l’opposé dans Neo4J, Cypher est un language dans lequel il est possible de définir directement des relations. Cypher est un peu plus difficile (je trouve) à prendre en main car sa syntaxe est vraiment particulière ; en revanche elle est plus adaptée aux graphes.
Dans le cadre de ce projet, la comparaison s’arrête là. Il faudrait de nombreuses pages pour comparer stockage, réplication, partitionnement, recherche d’information entre les deux outils. 
5.	Conclusion

Souhaitant au démarrage me concentrer sur une base de données NoSQL orienté graphes, l’étude d’ArangoDB dans le cadre de ce projet pour l’UE NFE204 m’a emmené bien au-delà de mes attentes. Ce n’est pas une base de données très utilisée (3e des bases graphes selon le classement de DB-engines], et je croyais implicitement que ses fonctionnalités seraient limitées. Mais bien au contraire, la richesse de cet outil qui propose un stockage orienté document, avec possibilité de construire des graphes au-dessus, un langage de requêtage qui évoque le SQL mais qui est en réalité bien différent mais qui permet quand même de faire des jointures, une API REST, des fonctionnalités de recherche d’information…Tout le cours de NFE204 se trouve condensé dans un même outil !
La contrepartie de cette richesse, c’est que j’ai mis beaucoup plus de temps que j’avais prévu initialement à me former (y compris, quoique sommairement, à Neo4J pour faire un bref comparatif comme je m’y étais engagé dans le document de présentation du projet), je n’ai pas pu creuser, aller en profondeur de tous les sujets comme je l’aurais souhaité. Si je devais reprendre au début, je choisirais sûrement de mon concentrer sur un seul aspect (probablement l’architecture).
Je dois dire que j’ai eu durant ce projet très peu de bugs, ce qui est plutôt rare dans mon expérience. La documentation en ligne n’est pas monumentale, mais bien calibrée suffisamment précisément car elle couvre tous les aspects. 
 
6.	Annexes 
i.	Lancement de trois instances sur trois images docker en mode starter 
> Machine docker 1 
export IP=192.168.99.100
docker volume create arangodb1
docker run -it --name=adb1 --rm -p 8528:8528 -v arangodb1:/data -v /var/run/docker.sock:/var/run/docker.sock arangodb
/arangodb-starter --starter.address=$IP

> Machine docker 2 
export IP=192.168.99.100
docker volume create arangodb2
docker run -it --name=adb2 --rm -p 8548:8528 -v arangodb2:/data -v /var/run/docker.sock:/var/run/docker.sock arango
db/arangodb-starter --starter.address=$IP --starter.join=$IP

> Machine docker 3 
export IP=192.168.99.100
docker volume create arangodb3
docker run -it --name=adb3 --rm -p 8558:8528 -v arangodb3:/data -v /var/run/docker.sock:/var/run/docker.sock arango
db/arangodb-starter --starter.address=$IP --starter.join=$IP 

ii.	Créer une collection 

 
iii.	Requête retournant le nombre de lignes d’une collection 

 

iv.	Essai de visualisation de graphe : 
Interprétation : j’ai choisi de commencer le parcours du graphe par la commune où j’ai grandi (Beautiran, code INSEE 33037). Cette commune est proche (physiquement certes, mais ici dans le graphe c’est en termes de nombre de déménagements) d’un gros nœud : Bordeaux, depuis lequel déménagent beaucoup de personnes vers de nombreuses communes.
 


