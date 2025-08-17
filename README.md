# Quelques éléments de documentation généraux

Ce court README vous donne les éléments de base à connaitre sur ce projet, puis vous redirige vers un readme plus spécialisé.

Ce github étant public, vous ne trouverez pas le code ici, seulement deux fichiers README.md correspondant à la documentation. Le code est sur le serveur HCAP5, dans le dossier `/data/sdm_centaure` (centaure étant le nouveau nom du projet).

Ce code est structuré autour de deux dossiers. Un dossier domain, qui contient des fonctionnalitées métier, et un dossier layers, qui contient des éléments de code qui sont utilisés dans différentes applications métiers présentes dans domain. Chaque dossier dans domain correspond donc à une application métier spécifique (gestion des traps, gestion de la collecte SNMP,...).

Plus précisement, la structure du projet est la suivante :

```
/sdm_centaure
│
├── /domain/
│   ├── /collection/
|   |    ├── main.py
|   |    ├── test.py
|   |    └── utils // contient tout les utilitaires propres au domain
|   |
│   ├── /trap/
│
├── /layers/ // ce qu'il y a dans ce dossier doit être modifié avec précaution, car peut impacter plusieurs fonctionnalitées métiers. Il ne s'agit pas de fonctionnalitées métier, mais de couches qui peuvent être assemblées de différentes manières pour former des fonctionnalitées métier. Les fichiers de test pytest portent principalement sur le contenu de ce dossier.
|    ├── /request_handler/ // brique de collecte SNMP, structure en classe
|    ├── /database_manager/ // brique d'interaction avec la base de donnée, structure en classe
|    ├── /event_listener/ // brique d'écoute sur des ports
|    ├── /alarm_manager/ // brique de gestion des alarmes, dépend de request_handler, database_manager et event_listener
|    └── /alerter/ // brique de gestion des alerteurs (envoi d'emails)
|
├── /obsolete/ // contient du code obscolete ou non fonctionnel, va être supprimé
|
├── /operators/
|   ├── col_operators.py // contient l'ensemble des 'operateurs' (fonctions de traitement des données avant enregistrement en base) de l'outil de collecte
|   ├── trap_operators.py // contient l'ensemble des 'operateurs' des traps
|
├── config.py // contient toutes les variables statiques et les éléments de configuration
├── pytest.ini // fichier de configration de pytest (vous pouvez lancer les tests en vous positionnant à la racine du projet et en lançant la commande pytest -v)
└── requirements.txt // les bibliothéques requises avec leur version
```

**requirements et version :**  
Un fichier de requirements indique l'ensemble des bibliothéques nécessaires pour la bonne execution du code. Les numéros de version doivent également être respectés, notamment pour pysmi et pysnmp (code incompatible avec les versions les plus récentes de ces deux bibliothéques).
L'ensemble de ce code est compatible avec python3.9, est normalement compatible avec python3.8 (mais pas de tests faits pour vérifier), et est peut-être compatible avec python3.7. Il n'est pas compatible avec des versions inférieures à python3.7. Le code devrait être compatible avec toutes les versions futures de python3, les mainteneurs de python s'étant engagés à maintenir le retrocompatibilitée à partir de python3.7.

Vous retrouverez ainsi les fonctionnalitées métier suivantes :

#### alarm_monitor

Module de surveillance des alarmes SNMP. Gère la collecte, le traitement et l'enregistrement des alarmes actives sur les équipements réseau. Inclut des fonctionnalités pour scraper les données SNMP d'alarmes, traiter les résultats et maintenir l'état des alarmes en base de données.

#### traps_and_alarms

Module de traitement des traps SNMP et des alarmes. Gère la réception, le parsing et l'enregistrement des traps SNMP en tant qu'alarmes dans la base de données. Inclut également des utilitaires pour la gestion des logs et l'auto-complétion des données historiques.

#### collection

Module de collecte de données SNMP. Génère des dictionnaires de configuration pour la collecte de données depuis les équipements réseau, en se basant sur les informations stockées en base de données. Gère la configuration et les logs de collecte.

#### document_mib

Projet en court non terminé..

#### process_alerters

Module de traitement des alertes. Gère l'exécution automatique du système d'alertes, permettant de traiter et d'envoyer des notifications basées sur les données collectées et les règles d'alerte configurées.

## les fichiers executables

Pour chaque fonctionnalité métier, vous trouverez à la racine du projet un (et parfois plusieurs) fichiers python correspondant au fichier principal de la fonctionnalitée. C'est toujours les fichiers à la racine du projet qui sont lancés lors des executions.

- collection_main.py : gére la collecte régulière des données. Est lancée toutes les 15 minutes par le crontab.
- listen_traps.py : gére l'écoute sur le port 162 et la gestion des traps reçues sur ce port (et des alarmes qui liées à ces traps). Est gérée par un service systemctl, traplistener-sdm, qui peut être lancé, arrêté, ect... avec les commandes systemctl start/stop/status/ect... traplistener-sdm comme n'importe quel autre service daemon.
- main_alarm_monitor : gère la vérification des alarmes en base à partir des données récupérées sur les équipements. Est lancé toutes les 5 minutes par le crontab.
- main_alarm_monitor_for_one_machine : peut être lancé à partir d'une requête HTTP sur l'endpoint api_ae/check_alarm.php (ou en commande shell si besoin, en précisant le nom de l'AE à vérifier dans l'argument name).
- manage_bdd_main : ne peut être lancé qu'en ligne de commande. Prend plusieurs arguments et est capable de faire des suppressions / du netoyage dans la base de donnée. Ne pas lancer sans savoir précisement ce que vous faites !!!
- snmp : petit script de test de requête snmp qui supporte DES. Ce lance en ligne de commande, et est documenté (taper python snmp.py -h pour la documentation). C'est juste un utilitaire de test, calqué sur snmpwalk, je le laisse si besoin.

## Layers

Layers se comporte comme une bibliothèque indépendante. Layers n'importe jamais rien depuis domain (et il serait plus que pertinent de garder cette logique !). Le seul fichier externe au dossier que le code peut importer est config.py. 

Vous pouvez aussi décider de limiter au maximum les nouvelles modifications dans cette partie du code et de ne l'utiliser que pour créer de nouvelles applications métiers ou modifier celles existantes en fonction des besoins. J'appelle "applications métier" tout les fichiers présentés juste au-dessus, chacun dépendant d'un dossier spécifique dans `domain/`.

## Et ensuite..

Pour la suite de cette documentation, je vous invite à lire le fichier README.md présent dans le dossier layers, qui contient une documentation générale de l'ensemble du code dans ce même dossier.
