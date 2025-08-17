# Introduction à la bibliothèque **Centaure**

Tout ce qui se trouve dans le dossier layers est le code de ce que j'appelle la bibliothéque Centaure (Centaure est le nom du projet). Ce README documente donc ce code.

La bibliothèque **Centaure** contient un ensemble de classes et de fonctions servant à la supervision en masse d'équipements réseau. C'est une bibliothéque de conception d'outil de supervision.
Ses fonctionnalités principales incluent :

- Le **requêtage en masse** d’un ensemble de machines sur un réseau.
- La **le traitement d’informations**.
- L'**écoute de traps**.
- L’**enregistrement** de ces données dans des bases.
- La **gestion d'alerteurs**.

## Petite introduction (Quand et pourquoi utiliser la bibliothèque Centaure ?)

Supposons que vous disposiez d’une base de données listant des machines d’un réseau, avec pour chaque machine :

- `nom_machine`
- `IP`
- Éventuels champs supplémentaires liés à l’accès réseau (clé d’authentification, etc.)

Si vous souhaitez :

1. Récupérer un ensemble de donnée pour chaque machine.
2. Enregistrer ces données en base dans un format structuré.

Alors **Centaure** pourra répondre à vos besoins.
Cependant, pour ce scénario simple, vous trouverez probablement d'autres outils également.

Partons des mêmes hypothèses que précédemment, mais ajoutons que :

- Chaque machine possède un **type** (`type_machine`).
- Vous avez potentiellement une deuxième base listant :

  - Des **OID**
  - Le **nom de la métrique**
  - Le **type de machine** associé à cet OID

**Schéma des tables :**

**Table `machines`**

| nom_machine     | IP         | type_machine | ... |
| --------------- | ---------- | ------------ | --- |
| 134401-AE-HUA-1 | 10.90.0.2  | Huawei       | ... |
| 130320-AE-ELT-1 | 10.90.0.30 | Eltek        | ... |
| ...             | ...        | ...          | ... |

**Table `oid_métriques`**

| oid             | nom_métrique     | type_machine |
| --------------- | ---------------- | ------------ |
| 1.3.6.1.2.1.1.3 | uptime_systeme   | Huawei       |
| 1.3.6.1.2.1.2.2 | trafic_interface | Eltek        |
| ...             | ...              | ...          |

**Objectif :**

- Pour chaque machine, contacter uniquement les OID correspondant à son type.
- Associer chaque valeur à une métrique avec un nom personnalisable.
- Avoir un contrôle total sur la structure de la base où les résultats sont enregistrés, et pouvoir utiliser des bases de données MySQL.

**Centaure** a été pensé pour repondre à ces objectifs, et sera donc parfaitement adaptée pour ce cas.

## Guide de démarage (_Get Started_)

Je reconstruis dans cette partie l'outil de collecte SNMP, étape par étape.

Pour commencer, voyons comment utiliser **Centaure** pour une collecte de donnée dans le scénario du **cas n°2** vu en introduction :

```python
# Import des classes principales
from layers.scrapper.BaseScrapper import BaseScrapper
from layers.request_handler.SNMPRequestHandler import SNMPRequestHandler
from layers.database_manager.MySQLManager import MySQLManager
import mysql.connector

def load_machines_from_db():
    """
    Récupère la liste des machines depuis la base MySQL.
    Retourne une liste de dictionnaires au format attendu par BaseScrapper.
    """

    db_config = {
        "host"="localhost",
        "user"="root",
        "password"="password",
        "database"="machines_db"
    }

    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT NAME, AE_IP, AE_CONSTRUCTEUR FROM ATELIER_ENERGIE")
    rows = cursor.fetchall()
    cursor.close()
    conn.close()

    machines_list = [
        {
            "nom_machine": row[0],
            "ip": row[1],
            "type_machine": row[2]
        }
        for row in rows
    ]
    return machines_list

def load_oids_from_db():
    """
    Récupère la liste des OID à interroger depuis la base MySQL.
    Retourne un dictionnaire :
    { type_machine: [ { 'oid': ..., 'metric': ... }, ... ] }
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT oid, nom_metrique, constructeur FROM METRIQUES")
    rows = cursor.fetchall()
    cursor.close()
    conn.close()

    oids_dict = {}
    for oid, metric, machine_type in rows:
        oids_dict.setdefault(machine_type, []).append({ # on classe les metriques par "machine_type"
            "oid": oid,
            "metric_name": metric
        })
    return oids_dict

class SNMPScrapper(BaseScrapper):
    def __init__(self, scrapping_data):
        super().__init__(scrapping_data) # on appelle la méthode parent

    def _load_default_requests(self):
        """
        Retourne un dictionnaire etiqueté avec le type de machine, contenant pour chaque étiquette une liste de dictionnaire {'oid', 'metric'}.
        """
        return load_oids_from_db()

    def init_requester_with_recorder(self, machine_data):
        """
        classe abstraite qu'on doit obligatoirement réécrire
        """
        address = machine_data['ip'] # dépend de la structure des dictionnaires renvoyés par load_machines_from_db
        name = machine_data['nom_machine']
        requests = self.load_requests(machine_data['type_machine']) # se base sur le dictionnaire retourné par _load_defaut_requests pour charger les bonnes données
        config = {'version': 2, 'community': 'public'} # ou autre chose, peut être chargé depuis la base de donnée en même temps que address ou name si besoin. C'est RequestHandler qui impose les labels à mettre dans le dictionnaire.

        data_manager = MySQLManager(
            SNMPRequestHandler(address, name=name, requests=requests, **config)
        ) # prend une instance de RequestHandler en entrée
        return data_manager # doit renvoyer l'intance, et peut aussi renvoyer alse pour signaler une erreur

if __name__ == "__main__": # on peut ensuite executer le code
    # Chargement des données des AE
    machines = load_machines_from_db()

    # Initialisation du scrapper
    scrapper = SNMPScrapper(machines)

    # Lancement de la collecte (avec sauvegarde des données)
    scrapper.run_all(save=True) # cette ligne peut bien sûr prendre un certain temps à s'executer, en fonction du nombre de machine et de metrique à historiser

    print("Collecte terminée et résultats enregistrés.")
```

Analysons un peu ce code :

1. **Importations**
   Le code importe trois classes clés :

   - `BaseScrapper` : classe mère gérant la logique générale de scrapping.
   - `SNMPRequestHandler` : gère les requêtes SNMP vers les machines.
   - `MySQLManager` : interface pour interagir avec une base MySQL.

2. **Fonction `load_machines_from_db()`**
   Récupère la liste des machines depuis la base de donnée.
   Retourne une liste de dictionnaires formatés pour `BaseScrapper`.
   Chaque machine est représentée par un dictionnaire :

   ```python
   {"nom_machine": ..., "ip": ..., "type_machine": ...}
   ```

3. **Fonction `load_oids_from_db()`**
   Se connecte à la base de donnée et récupère les OID SNMP à interroger, avec leur nom de métrique et le type de machine associé. Regroupe ces OID dans des dictionnaires au format {"oid": ... ,"metric_name": ... }

4. **Classe `SNMPScrapper`**
   Hérite de `BaseScrapper` et personnalise certaines méthodes en utilisant SNMPRequestHandler et MySQLManager.
   On a notamment réécrit la méthode \_load_default_requests, pour qu'elle importe depuis notre base de donnée les informations utiles au bon format (regroupés par `type_machine` dans un dictionnaire).

5. **Méthode `init_requester_with_recorder()`**
   Configure un `SNMPRequestHandler` avec :

   - l’adresse IP,
   - le nom de la machine,
   - la liste d’OID à interroger,
   - la configuration SNMP (version, communauté, etc.).
     L’instance est ensuite encapsulée dans un `MySQLManager` pour enregistrer les données.

6. **Bloc `if __name__ == "__main__":`**
   - Charge les machines depuis la base.
   - Initialise un scrapper SNMP.
   - Lance la collecte avec `run_all(save=True)`, qui interroge toutes les machines et stocke en base les résultats.

> ce code ne fonctionnera pas sur HCAP5 pour trois raisons : la configuration de la base de donnée MySQL n'est pas la bonne, machine_data['type_machine'] ne contiendra pas les bonnes étiquettes même avec la bonne configuration MySQL (elle contient par exemple l'étiquette 'Huawei', mais self.load_requests attend l'étiquette 'huawei') et la configuration fournie n'est valable que pour SNMPv2, donc seules les machines en SNMPv2 seront joignables avec cette configuration de toute façon.

💡 Notez qu'en fait, il existe déjà une classe `SNMPScrapper` héritant de `BaseScrapper`. L'idée dans la suite de ce _Get Started_ est de voir comment reconstruire et utiliser ce SNMPScrapper.

Pour expliquer le fonctionnement exacte de la classe SNMPScrapper tel qu'elle est implémentée, nous allons devoir voir un certains nombre d'éléments supplémentaires.

### personnaliser la configuration SNMP

Supposons maintenant que les éléments de configuration réseau ou SNMP (SNMPv2 ou v3, chiffrement AES ou DES,...) ne sont pas dans votre base de données.

Dans le code précédent, nous définissons et utilisons deux méthodes, \_load_default_requests et load_requests, qui permettent de personnaliser les requests en fonction d'un champ spécifique (ici le constructeur).

Il existe en fait deux méthodes symétriques à \_load_default_requests et load_requests. C'est les méthodes \_load_default_config et load_config. En utilisant ces deux méthodes, on peut tout-à-fait créer une base de donnée avec les spécifications réseau (SNMPv2 ou v3, community, ect...), mais on peut aussi plus simplement créer "à la main" un dictionnaire de configuration avec autant d'entrée que nécessaire, et le faire renvoyer par \_load_default_config.

C'est ce qu'on va faire ici, mais plutôt que de le définir directement dans la méthode, créons un fichier de config à la racine du projet, `config.py`, et définissons dans ce fichier de config une variable globale CONFIG, qui contiendra ce fameux dictionnaire de configuration.

Ajoutons autre chose : actuellement, ce code fait de l'enregistrement en base, mais quel est le format exacte des bases de donnée ? Pour aider le code (et pour d'autres raisons beaucoup plus pertinentes que nous verrons plus loin), vous pouvez fournir une liste des noms des colonnes des tables SQL à MySQLManager, dans l'argument table_struct. Ajoutons ça.

Enfin, ajoutons de la gestion des logs. Quasiment toutes les classes de centaure offrent un moyen de personnaliser les logs qu'elles produisent. Pour ce faire, il faut leur fournir un objet Logger personnalisé de la bibliothéque standard logging.

> logging est une bibliothéque standrard de gestion des logs en python. Elle contient notamment un objet appelé logger, qui est capable de gérer des logs avec ses méthodes debug, warning, error... En cas de doute sur le fonctionnement de cette bibliothéque, vous pouvez consulter la documentation : https://docs.python.org/3.9/library/logging.html. Vous trouverez aussi un exemple de code commenté dans `domain/collection/manage_log.py`.

Complétons le code pour rajouter tout ça :

```python
from config import CONFIG # on suppose qu'on a créé un dictionnaire nommé CONFIG dans le fichier config à la racine du projet
from logging import Logger

class SNMPScrapper(BaseScrapper):
    def __init__(self, scrapping_data, logger: Logger = None): # ici on rajoute un paramètre logger, pour permettre à l'utilisateur d'en fournir un si besoin
        super().__init__(scrapping_data, logger = logger)

    def _load_default_requests(self):
        """
        Retourne un dictionnaire etiqueté avec le type de machine, contenant pour chaque étiquette une liste de dictionnaire {'oid', 'metric'}.
        """
        return load_oids_from_db()

    def _load_default_config(self):
        """
        Retourne un dictionnaire etiqueté avec le type de machine, contenant pour chaque étiquette une liste de dictionnaire {'oid', 'metric'}
        """
        return CONFIG # on a juste besoin de renvoyer ça

    def init_requester_with_recorder(self, machine_data):
        """
        classe abstraite qu'on doit obligatoirement réécrire
        """
        address = machine_data['ip'] # dépend de la structure des dictionnaires renvoyés par load_machines_from_db
        name = machine_data['nom_machine']
        requests = self.load_requests(machine_data['type_machine']) # se base sur le dictionnaire retourné par _load_defaut_requests pour charger les bonnes données
        config = self.load_config(machine_data['type_machine']) # on rajoute le chargement de la config
        # config ici peut par exemple être un dictionnaire du type {'version': 3, 'auth_protocol': "MD5", 'priv_protocol': "AES",... }

        data_manager = MySQLManager(
            SNMPRequestHandler(address, name=name, requests=requests, **config, logger=self.logger), # on transmet notre logger personnalisé, qui est dans self.logger, à ces instances
            table_struct = ['datetime', 'metric_index', 'value'],
            logger = self.logger # on transmet notre logger personnalisé
        ) # prend une instance de RequestHandler en entrée
        return data_manager # doit renvoyer l'intance, et peut aussi renvoyer False pour signaler une erreur
```

> En vérité, il y a encore quelques petites différences entre ce code et le véritable code de SNMPScrapper. En fait, actuellement dans le code sur HCAP5, les requests faites (donc les oids) dépendent uniquement du constructeur, mais la config dépend quant-à-elle du constructeur ET du modèle (du champ modele en base). On cré donc dans le dictionnaire scrapping_data un champ `requests`, qui contient le constructeur en minuscule, et le champ `config`, qui contient un string au format constructeurMODELE. La variable CONFIG dans config.py est de fait bien un dictionnaire etiquetée avec des strings au format constructeurMODELE. Je vous invite d'ailleurs à consulter le fichier config.py si vous voulez mieux comprendre ce point. Pour les requests, on etiquette avec le premier mot du champ 'constructeur' en minuscule.

Comme évoqué précédemment, vous pouvez évidemment également stocker la config dans une base de donnée, ou écrire directement le dictionnaire dans _load_default_config. Notez que load_config renvoie la valeur qu'elle reçoit sans la modifier si c'est une liste, un dictionnaire ou un tuple. load_resquests renvoie la valeur sans la modifier uniquement si c'est une liste, et renvera None si elle reçoit un dictionnaire ou un tuple. Si c'est un string, les deux fonctions vont chercher une clé correspondante dans ce qui est renvoyé par \_load_default_\*, et renvoyer cette valeur, ou None si elles ne trouvent rien.

> Note 1 : les méthodes \_load_default* ne sont appelées qu'une seule fois (par instance de scrapper), lors du premier appel à la fonction load* correspondante avec un string en entrée.

> Note 2 : les fonctions load\* ne renvoient jamais d'erreur. Elles log une erreur et renvoient None en cas de problème.

> Note 3 : Pourquoi seules les listes sont renvoyées tel quel par load_requests ? Parce que requests doit ensuite être fourni à un RequestHandler, et que RequestHandler attend en entrée une liste (de dictionnaire), et rien d'autre. La config est elle gérée par init_requester_with_recorder directement, donc le format est plus libre. Notez par contre que la méthode \_load_default_config (tout comme \_load_default_requests d'ailleurs) doit renvoyer un dictionnaire, c'est ce qu'attend load_config quand elle l'appelle.

### le champ options

Et maintenant si vous voulez mélanger des configurations écrites à la main et des éléments de configuration qui sont dans votre base de données, comment faire ?

En effet vous pourriez avoir une configuration générale de ce type :
SNMPv3, chiffrement AES, authentification SHA, mais avec des mots de passe et/ou nom d'utilisateur (username) spécifiques pour chaque machine.

En fait, on pourrait tout-à-fait gérer ça à la main : il suffirait de réécrire la méthode init_recorder_with_requester, et d'entrer dans requester les champs soit les champs que vous voulez.

Notez cependant qu'il existe une alternative un peu plus "propre" pour faire ça, et qui est utilisée dans le code, en utilisant le champ _options_ de SNMPRequestHandler :
Vous pouvez, dans votre fichier de config, noter entre deux '\_\_' certains champs. Vous pourrez ensuite fournir ces champs à SNMPRequestHandler. Si vous fournissez le champ \_\_username\_\_ pour la valeur username en entrée de requestHandler, celui-ci ira chercher le champ 'username' dans le champ options fourni en entrée (qui doit être un dictionnaire). S'il le trouve, il va remplacer username par la valeur lié à la clé 'username' dans le champ options.

```python
machine_data = {
    'name': '134401-AE-HUA-1',
    'username': '1344011'
}
data_manager = MySQLManager(
    SNMPRequestHandler(address, name='__name__', requests=requests, username='__username__', ... , options=machine_data, logger=self.logger), # on rajoute options=machine_data, qui nous permettra d'utiliser la notation __label__ pour tout les labels contenus dans machine_data. Ici, username sera remplacé par 1344011 dans SNMPRequestHandler.
    logger = self.logger
) # prend une instance de RequestHandler en entrée
```

Notez que le champ `options` peut également être utilisé par toute autre méthode de requestHandler et même de dataManager (pour rappel, dataManager peut prendre en entrée un RequestHandler, et dès lors peut donc accèder à tout ses attributs).
En pratique, le champ options est effectivement utilisé dans MySQLManager. On appelle options['id'] pour générer les noms de table. Vous devez donc fournir à cette classe des instances de RequestHandler avec un champ options contenant au moins la clé 'id'.

> Le `username` n'est pas directement dans la base SQL Historica. En fait, dans le code sur HCAP5, ce champ est généré à partir du nom des AE dans la fonction qui génère le champ scrapping_data (dans domain/collection/generate_dictionnary.py).

### les opérateurs

Mais imaginons maintenant que vous avez le problème suivant : vous ne voulez pas enregistrer en double certaines informations dans votre base de données pour ne pas la surcharger. Ou alors vous voulez appliquer un pré-traitement à vos données, par exemple déchiffrer des données encodées en hexadécimal, avant de les enregistrer en base, et ce sur certaines données spécifiques uniquement. Vous pourriez alors vouloir utiliser les opérateurs.
Les opérateurs (ou operators dans le code) sont des fonctions qui s'appliquent sur les données avant leur enregistrement. C'est une manière de pré-traiter les données enregistrées en base. La gestion des opérateurs se fait dans BaseDataManager, et nécessite deux éléments :

- d'une part, vous devez fournir à BaseDataManager, dans la liste des requests, plus spécifiquement au niveau du dictionnaire de la métrique à traiter, un champ operators, contenant un string représentant une fonction.
- d'autre part vous devez créer un dossier operators à la racine de votre projet et inclure un fichier col_operators.py. Vous devez ensuite écrire une fonction dans ce fichier avec EXACTEMENT le même nom que le string soumis dans le dictionnaire correspondant à la request, et avec le squelette suivant :

```python
def func(metric: str, values: list[dict], connector):
    # contenu de la fonction
    return metric, values
```

où :

- metric est un string, le nom de la metrique, qui va principalement définir le nom de la table dans laquelle la donnée va être enregistrée (le nom sera au format metrique_ID)
- values est une liste de dictionnaire, où chaque dictionnaire a un format du type {'datetime':12121212, 'metric_index': 1, 'value': '12'} (les clés correspondent à des colonnes en base, et contiennent la valeur correspondante qui sera enregistrée).
- connector est un objet contenant un champ connect (connector.connect) et cursor (connector.cursor) de la bibliothéque mysql. Ce champ permet surtout de faire (si besoin) des requêtes en base dans les operateurs sans avoir à se reconnecter à la base SQL (la connexion étant déjà ouverte), donc en limitant les pertes de temps.
- vous renvoyez un champ values contenant les valeurs que vous voulez enregistrer, et qui peut valoir 0 ou None si vous ne voulez finalement pas enregistrer de valeur. Vous devez également renvoyer le champ metric (on renvera en général celui qu'on a reçu en entrée, mais vous pouvez bien sûr le modifier si besoin).

Avant l'enregistrement en base de chaque métrique, BaseDataManager vérifie s'il existe un champ operators dans le dictionnaire de la request avec un contenu valide. Si c'est le cas, il tentera de charger une fonction depuis operators/col_operators avec exactement le même nom, et l'appliquera aux données.

> Note 1 : operators peut également être une liste, ou un long string avec plusieurs noms séparés par des virgules ou des ';', et dans ce cas l'instance se chargera d'en extraire une liste via du regex. Elle appliquera tout les opérateurs les uns à la suite des autres, dans l'ordre dans lequel ils ont étés fournis.

> Note 2 : le nom du fichier col_operators est personnalisable. Il suffit d'indiquer à l'instance de la classe dans quel fichier elle doit aller chercher les opérateurs. Elle cherchera par contre toujours le fichier renseigné dans le dossier operators à la racine du projet. En pratique, on utilise bien le fichier col_operators pour les opérations de collecte en base.

> Note 3 : Comme déjà évoqué, les operateurs sont gérés par MySQLManager, ou plus précisement par sa classe parente BaseDataManager. Le principe est le suivant : tout les operateurs du fichier indiqué sont importés lors de l'initialisation, puis la méthode load_operators est appelée quand des operateurs doivent être appliqués et renvoie une liste d'operateur (d'objet Callable) ou False si elle n'a pas rien trouver ou s'il n'y a aucun operateur à appliquer, les operateurs sont ensuites tous appliqués aux données dans la methode qui a appelé load_operators (en général run_metric). A noter que la méthode load_operators n'est pas optimale en terme de temps d'execution (cherche les opérateurs à partir du nom de la métrique donc doit parcourir toute la liste des metriques), et qu'une amélioration future pourrait impliquer de l'améliorer un peu (même si son temps d'execution est probablement très faible par rapport au temps des requêtages et enregistrements en base).

### SNMPScrapper, version finale

On a maintenant vu tout les éléments essentiels à comprendre pour créer la classe SNMPScrapper. Voyons le code complet :

```python
# Import des classes principales
from layers.scrapper.BaseScrapper import BaseScrapper
from layers.request_handler.SNMPRequestHandler import SNMPRequestHandler
from layers.database_manager.MySQLManager import MySQLManager
import mysql.connector
from config import get_constructor_string, get_config_from, ae_db_config as db_config
import re

def load_machines_from_db():
    """
    Récupère la liste des machines depuis la base MySQL.
    Retourne une liste de dictionnaires au format attendu par BaseScrapper.
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor(dictionary=True)  # Retourne directement des dictionnaires
    cursor.execute("""
        SELECT NAME, AE_IP, AE_CONSTRUCTEUR, AE_TYPE, ID
        FROM ATELIER_ENERGIE
        WHERE AE_SUP = 1
    """)
    rows = cursor.fetchall()
    cursor.close()
    conn.close()

    machines_list = [
        {
            "name": row["NAME"],
            "address": row["AE_IP"],
            "requests": get_constructor_string(row["AE_CONSTRUCTEUR"]), # l'étiquette des requêtes
            "config": generate_config_string(row["AE_CONSTRUCTEUR"], row["AE_TYPE"]), # l'étiquette de la config
            "username": re.sub(r'\D', '', row["NAME"])  # garde uniquement les chiffres
        }
        for row in rows
    ]
    return machines_list


def load_oids_from_db():
    """
    Récupère la liste des OID à interroger depuis la base MySQL.
    Retourne un dictionnaire
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT oid, nom_metrique, constructeur, operateurs FROM METRIQUES")
    rows = cursor.fetchall()
    cursor.close()
    conn.close()
    oids_dict = {}
    for oid, nom_metrique, constructeur, operateurs in rows:
        oids_dict.setdefault(constructeur, []).append({ # on classe les metriques par "constructeur"
            "oid": oid,
            "metric_name": nom_metrique,
            "operator": operateurs
        })
    return oids_dict

class SNMPScrapper(BaseScrapper):
    def __init__(self, scrapping_data, logger=None):
        super().__init__(scrapping_data, logger=logger)

    def _load_default_requests(self):
        return load_oids_from_db()

    def init_requester_with_recorder(self, machine_data: Dict[str, str]):
        if not 'address' in machine_data or not 'name' in machine_data or not 'requests' in machine_data or not 'config' in machine_data:
            return False

        address = machine_data['address']
        name = machine_data['name']
        requests = self.load_requests(machine_data['requests'])
        config = self.load_config(machine_data['config'])

        if isinstance(config, dict):
            recorder = MySQLRecorder(
                SNMPRequestHandler(address, name=name, requests=requests, **config, options=machine_data, logger=self.logger),
                table_struct = collection_table_structure,
                logger = self.logger
            )
            self.logger.info("Dans init_requester_with_recorder (SNMPScrapper) : message avec les informations X ou Y")
            return recorder
        else:
            return False

if __name__ == "__main__": # on peut ensuite executer le code

    # Chargement des données des AE
    machines = load_machines_from_db()

    # Initialisation du scrapper
    scrapper = SNMPScrapper(machines)

    # Lancement de la collecte (avec sauvegarde des données)
    scrapper.run_all(save=True) # cette ligne peut prendre un certain temps à s'executer, en fonction du nombre de machine et de metrique à historiser

    print("Collecte terminée et résultats enregistrés.")
```

> Ce code fonctionne bien, à condition de l'executer dans le dossier /data/sdm_centaure, pour qu'il puisse accéder à config.py. Il réalise une collecte complète sur tout les AE récupérés, ce qui peut prendre facilement 10 à 20 secondes. Ici, je n'ai pas fourni d'objets logger à mon code, l'execution affichera donc tout les logs (comportement par défaut).

L'historique du projet et des bases de donnée fait que les champs servant de clé entre les bases ('Vertiv / Emerson', 'Eltek', ect...) ne sont pas exactement les mêmes entre différentes bases. On a par exemple la correspondance suivante entre les constructeurs :

    {
        'Huawei': 'huawei',
        'Eltek': 'eltek',
        'Vertiv / Emerson': 'vertiv'
    } // Notez qu'on prend en fait le premier mot en minuscule

Vous trouverez à la fin de config.py quelques fonctions qui servent justement à assurer cette correspondance, et qu'on utilise ci-dessus dans le code.

Par ailleurs, les champs table_struct utilisés sont également tous définis dans config.py en pratique, et non directement dans le code.

Bref, la conclusion de toute cette partie : quand vous avez un problème sur la collecte, commencez par aller voir dans config.py !

## Quelques compléments sur MySQLManager

On évoque beaucoup cette classe depuis le début, sans rentrer dans les détails de son fonctionnement. Je reviens donc sur les éléments centreaux de cette classe dans cette partie, d'autant plus qu'elle est utilisée dans quasiment toutes les applications métiers (collecte, gestion des traps, alarmes,...)

La classe MySQLManager gère en fait les intéractions avec la base de donnée. L'intérêt d'avoir une classe dédiée est d'autant plus fort que la structure de la base de donnée n'aide pas forcement à intéragir avec.

La classe MySQLManager est donc capable :

- D'enregistrer des données en base en masse, via sa méthode save
- De modifier des données en base en masse, via sa méthode update
- De récupérer des données en base avec sa méthode fetch
- De supprimer des données en base avec delete

Elle gère les données par machines (AE) et non par table. C'est d'ailleurs tout l'intérêt de ces classes. Elle va être capable d'intéragir avec toutes les tables associées à une même machine sans que l'utilisateur n'ai besoin de se préocuper ou même de connaitre la structure des tables et bases de donnée !

Cette classe est construite sur deux classes parentes :

- BaseDataManager : classe abstraite qui implémente un certains nombre d'élément de logique indépendemment du type de base (MySQL) utilisé
- MySQLRecorder : hérite de BaseDataManager, et implémente la méthode save (plus exactement \_save_entry, j'y reviens plus loin) pour permettre les sauvegardes en base. En pratique, on utilise cette classe dans l'outil de collecte plutôt que MySQLManager, puisqu'on a uniquement besoin des opérations d'enregistrement en base (save).
- MySQLManager : hérite de MySQLRecorder, donc peut utiliser la méthode save. Implémente les méthodes, update, fetch, et delete.

Le principe général de cette classe est le suivant :
La classe possède un attribut data (qui est au format {'metrique_1': [{'datetime': 121212, 'value': '234', ...}, ... ], ... }). A chaque fois qu'elle veut faire une opération en base, elle :

1. gére l'ouverture de la connexion à la base de donnée -> self.run
2. parcourt chaque metrique dans data -> self.run
3. gére les operateurs pour chaque liste (values) associée à chaque metrique -> self.run_metric
4. parcourt la liste après action des operateurs -> self.run_metric
5. prétraite chaque entrée (value) pour en extraire une entry -> self.run_metric
6. appelle une méthode spécifique définie dans une des classes enfants pour faire l'opération en base voulue (les quatres méthodes sont \_fetch_entry, \_save_entry, \_update_entry, \_delete_entry)

Où entry est une liste du type :

    [
        {
        'column_name': 'datetime',
        'column_type': self.SQL_format['datetime'], // INT NOT NULL DEFAULT 0
        'value': value
        },
        {
        'column_name': 'datetime',
        'column_type': self.SQL_format[label], // INT NOT NULL DEFAULT 0
        'value': value
        },
        {
        'column_name': 'datetime',
        'column_type': self.SQL_format[label], // INT NOT NULL DEFAULT 0
        'value': value
        }
    ]

> Ok, mais qu'est ce qu'est le self.SQL_format ci-dessus ?

Voyons ça, et tant qu'à faire, voyons l'ensemble des éléments d'entrée de MySQLRecorder et MySQLManager (ils ont les mêmes).

Voici l'ensemble des attributs d'entrée de MySQLRecorder et MySQLManager :

```python
def __init__(self,
    instance: Union[RequestHandler, None] = None,
    data: Any = None,
    database_config: Dict[str, str] = None,
    tables_id: Union[str, int, None] = None,
    table_struct: Union[List[str], None] = None,
    connector: Any = None,
    SQL_format: Union[Dict[str, str], None] = None,
    auto: bool = False,
    name: str = "",
    logger = None
)
```

Voyons ce que fait chaque élément :

**instance :**
Doit recevoir une instance d'un RequestHandler. On a déjà utilisé cet argument dans des codes plus haut, pour la classe SNMPScrapper. Quand vous fournissez un champ instance, l'instance se retrouvera enregistrée dans l'attribut self.object. Vous devrez collecter des données avec cette instance self.object (par exemple en lançant self.object.run(), voir section dédiée aux RequestHandler plus loin). Vous pourrez ensuite, par exemple, enregistrer directement les données récupérées par self.object simplement en appelant la méthode save. Elle ira directement récupérer les données sauvegardées dans self.object.

**data :**
Vous pourriez aussi vouloir faire des enregistrements en base, modifications, ect... sans passer par une instance de RequestHandler. Vous pouvez alors directement fournir vos données dans le champ data.
Le format attendu par data est le suivant :

    {
        'metrique_1': [
            {'datetime': 121212, 'metric_index': 0, 'value': '432'},
            ...
        ],
        ...
    }

Vous ne voudrez donc à priori jamais fournir un champ data et un champ instance en entrée de MySQLManager. Notez que les valeurs dans data seront potentiellement écrasées quand vous appelerez object.run().

> Remarque : vous pourrez accéder à data en appelant self.data après l'initialisation de votre instance. De même, si vous utilisez une instance, vous pourrez voir les données récupérées dans self.data après lancement des requêtes sur la machine (avec self.object.run() par exemple). Notez que vous ne pouvez pas modifier à la main le champ data (self.data = truc renvera une erreur). Si vous devez absolument modifier data, vous devez appeler self.set_data(nouvelle_valeur_que_vous_voulez_donner_a_data).

**database_config :**
C'est tout simplement les éléments de connexion à la base de donnée. Vous n'aurez pas besoin de l'utiliser la plupart du temps car quand rien n'est fourni, les instances de MySQLRecorder vont chercher la configuration appelée `db_config` dans config.py.

**tables_id :**
Attend un int (un ID) correspondant à un AE spécifique (c'est son ID dans la table ATELIER_ENERGIE). Essentiel sinon le code ne sera pas identifier quelles tables correspondent à la machine sur laquelle on fait des opérations ! Le seul cas ou on peut s'en passer : si vous avez fourni une entrée `instance` contenant un champ options avec {'id': ID}.

**SQL_format :**
Le fameux ! Attend un dictionnaire, avec comme label (key) le nom des colonnes en table, associé au type de ces colonnes. Si rien n'est fourni (c'est quasiment tout le temps le cas), va se set avec le dictionnaire `database_struct_config` de config.py. Voyons un extrait de ce dictionnaire (que vous pouvez retrouver dans le fichier config.py) :

    { # structure des données en base
        'datetime': 'INT NOT NULL DEFAULT 0',
        'metric_index': 'SMALLINT NOT NULL DEFAULT 0',
        'value': 'VARCHAR(63)',
        ...
    }

En fait, MySQLRecorder (donc également MySQLManager) va comme vu précédemment prétraiter les données en leur associant un type (INT, VARCHAR, ect...) grâce à SQL_format, ce qui lui pemet de faire de la gestion d'erreur SQL. Plus précisemment, elle gère les erreurs 'table not exists' et 'columns not exists'. Pour ce faire, elle va tout simplement créer la table (ou les colonnes dans la table) en se basant sur entry, qu'elle reçoit en entrée, et sur les types qu'elle contient. C'est là l'utilité du champ SQL_format !

**table_struct :**
Attend une liste de string. Permet d'imposer (de renseigner) la structure des tables en base. Il suffit de fournir une liste contenant le nom des colonnes, correspondant à des labels dans SQL_format (ou database_struct_config si SQL_format n'est pas fourni). Ce champ force l'instance à respecter une structure de table spécifique quand elle va créer des tables ou des colonnes dans les tables. Si cet argument n'est pas fourni, l'instance va se paser sur le contenu de entry pour construire les tables (ce qui veut dire que si un entry contient par erreur une valeur en plus, une nouvelle colonne sera créée pour stocker cette valeur. Si vous préciser un table_struct, la valeur en trop dans entry sera ignorée).
Il est probabablement toujours bon de renseigner un table_struct quand vous utilisez MySQLRequester ou MySQLManager si vous savez quelle structure doit avoir votre base.

**connector :**
Permet de renseigner un objet connector, (donc contenant connector.connect et connector.cursor de la librairy mysql.connector en python). J'en profite d'ailleurs pour préciser qu'un constructeur de connector est disponible dans `layers/database_manager/connector/MySQLConnector.py`.
Le connector peut aussi être renseigné dans la méthode save, fetch, ect... au moment ou on l'appelle, et c'est plutôt cela qu'on fera la plupart du temps. Vous pouvez cependant du coup le renseigner dans le constructeur, et il sera alors utilisé dans toutes les opérations en base faites.
Notez que même si vous ne renseignez rien, vous pourrez quand même utiliser les méthodes save, fetch... sans problème. Ces méthodes appeleront la méthode interne open_database qui se chargera d'ouvrir la connexion à la base de donnée à partir de `db_config` dans config.py (ou du database_config que vous aurez fourni en entrée le cas échéant). Il s'agit juste de différentes manières possibles de gérer la connexion à la base de donnée.

**auto :**
Permet de forcer la connexion à la base de donnée lors de l'initialisation si passé à True (False par défaut). ça avait de l'intérêt dans une vielle version du code, mais ça n'en a quasiment plus maintenant sauf cas très particulier, et je ne conseille pas de l'utiliser (notez qu'une connexion SQL se referme souvent après quelques secondes d'inactivité, il vaut donc mieux ouvrir la connexion au dernier moment quand c'est possible, donc lors de l'appel à save, fetch, ect...).

**name :**
Permet de renseigner un nom, qui sera accessible dans self.name et qui apparaitra dans certains logs d'erreur (c'est pratique si vous instanciez des disaines d'instance dans un même code pour comprendre d'ou vient un éventuel problème, cela dit c'est le seul intérêt de ce champ). Si un champ `instance` est fourni, prend le nom associé à cette instance.

**logger :**
Déjà évoqué, vous pouvez fournir un objet logger pour personnaliser la gestion des logs.

Pour compléter ce tableau, voyons également les arguments d'entrée des méthodes save, fetch, ect... Ces méthodes appellent toutes la méthode run avec différents arguments. Le fonctionnement de self.run a déjà été évoqué plus haut, mais voyons ici quelques variantes à travers la documentation de ces méthodes.

Voici, pour commencer, le code définissant ces méthodes (que vous retrouverez dans le fichier `layers/database_manager/BaseDataManager.py`) :

```python
def save(self, check_operator: bool = True, connector: Any = None) -> bool:
    return self.run(operation_type='_save_entry', check_operator=check_operator, connector=connector)

def fetch(self, where: str = None, check_operator: bool = False, prepare = True, once = False, connector: Any = None) -> List[Dict[str, Any]]:
    return self.run(operation_type='_fetch_entry', where=where, check_operator=check_operator, prepare=prepare, once=once, connector=connector)

def update(self, where: str = None, check_operator: bool = True, connector: Any = None) -> bool:
    return self.run(operation_type='_update_entry', where=where, check_operator=check_operator, connector=connector)

def delete(self, where: str = None, check_operator: bool = False, prepare = True, once = False, connector: Any = None) -> bool:
    return self.run(operation_type='_delete_entry', where=where, check_operator=check_operator, prepare=prepare, once=once, connector=connector)
```

Voyons l'ensemble des arguments qui apparaissent :

**check_operators :**
booléen, True par défaut. Permet, si on le passe à False, d'empécher l'action des operateurs, ce qui rend au passage l'execution à peine plus rapide. A noter que cet argument n'a pas vraiement d'intérêt, et n'agira probablement pas de la manière dont vous vous y attendez, pour les méthodes fetch et delete. Il est d'ailleurs à False par défaut pour ces deux méthodes.

**connector :**
objet connector. C'est surtout utile quand on a plusieurs MySQLManager et qu'on veut optimiser le temps d'execution : on peut alors ouvrir la connexion en amont et passer la connexion déjà ouverte à chaque appel des méthodes fetch, save, update,...

**where :**
Attend un string. Permet de préciser une condition where (le string s'ajoutera à la suite de la requête SQL avec "WHERE " + where).
Petite particularitée : le string where est prétraité avant d'être utilisé dans la requête, par la méthode `_treat_where` (de BaseDataManager), de manière à remplacer les éléments au format `__label__` par la valeur associée au label dans chaque value. C'est particulièrement util dans certains cas pour update (vous pouvez par exemple faire des modifications en base en masse et de manière précise en indiquant un `metric_index` spécifique dans vos values et en fournissant un where = "metric_index = \_\_metric_index\_\_).

**operation_type :**
Argument d'entrée de run, qui vous peut-être intrigué. Comme évoqué précdemment, run est globalement juste une méthode qui itere sur une liste d'entrée, leur applique divers traitements, puis les envoient à d'autres méthodes. Et bien operation_type est un string correspondant au nom de "l'autre méthode" à laquel run (ou plus exactement run_metric la plupart du temps) va envoyer ses entrées traitées. La méthode cherche dans sa propre instance une méthode avec le même nom et la charge pour lui fournir les données. Vous n'avez cependant en général pas à vous préocuper de ce champ.

**prepare :**
booléen. Empéche l'étape de préparation des données quand il est à False (renvoie une liste avec un dictionnaire vide à la place des données attendues). Son utilitée est vue juste après.

**once :**
Modifie le fonctionnement de run quand il est True, son unique utilitée est vue juste après également.

### quelques spécificitées de fetch

MySQLManager a d'abord été pensée pour faire des enregistrements en base. Les autres opérationsont été ajoutés progressivement au fil de l'eau. De fait la méthode fetch fonctionne sur le même schéma que la méthode save, Ce qui rend son fonctionnement particulièrement contre-intuitif, et qui la rend donc compliquée à utiliser.

> D'ailleurs, repenser le fonctionnement des recherches en base pour cette classe est à mon avis un axe d'amélioration du code très pertinent. Il serait notamment intéressant de l'étendre pour pouvoir faire des requêtes avec des ORDER BY, LIMIT, ect...

Mon conseil :
La plupart du temps, quand on fait un SELECT, on voudra en fait juste récupérer toutes les données vérifiant une certaine condition WHERE, qu'on précisera en clair (sans utiliser les notations type \_\_label\_\_). Dans ce cas, les codes suivants fonctionneront très bien.

Pour récupérer les données liées à plusieurs metriques sur un même AE (c'est ce que vous voudrez faire la plupart du temps), vous pouvez utiliser le code ci-dessous :

```python
data_manager = MySQLManager(data = {'metrique_1': [ ... ], 'metrique_2': [ ... ], ... }, tables_id=ID) # ID : l'id correspondant à l'AE dont vous voulez récupérer les données.
# le champ data servira de référence pour savoir quelles metriques récupérer
data_manager.fetch(where = "la condition sql que vous voulez", prepare = False, once = True) # on precise bien les arguments prepare = False et once = True. Ce code ira récupérer les données liées à toutes les metriques contenues dans data en respectant la condition where fournie. Vous pouvez aussi passer le where directement au constructeur : data_manager = MySQLManager(..., where = "La condition sql que vous voulez", ...)
print(data_manager.fetched_data) # contiendra un dictionnaire du type {'metrique_1': [{'datetime':121212, 'metric_index': 2, 'value': '54'}, ...], ... }
```

> J'en profite pour faire remarquer que les données de fetch s'enregistrent dans un attribut spécifique, l'attribut fetched_data.

A noter que vous pouvez faire exactement la même chose sans préciser once=True, mais qu'il faut dans ce cas fournir à MySQLManager le champ `data={'metrique_1': [], 'metrique_2': [], ... }` (avec des listes vides).

Si vous voulez les données d'une seule métrique (donc d'une seule table) :

```python
data_manager = MySQLManager(data = { ... }, tables_id=ID) # ID : l'id correspondant à l'AE dont vous voulez récupérer les données.
# Ici, ce que vous indiquez dans data n'a strictement aucunes importances
data_manager.run_once('_fetch_entry', metric_name = nom_de_la_metrique_a_recuperer, where = "ce que vous voulez", prepare = False) # là encore, prepare = False permet d'éviter certains comportements inatendus
```

Dans tout les cas, les données récupérées seront accessibles dans data_manager.fetched_data au format :

    {
        'metrique_1': [
                {'datetime':121212, 'metric_index': 2, 'value': '54'},
                ...
            ],
        'metrique_2': [
            ...
        ],
        ...
    }

> Vous pouvez également bien sûr créer vos propres scripts pour faire des SELECT, et ce sera peut-être la solution la plus simple dans certains cas, en particulier si vous voulez récupérer les données liées à une metrique pour chaque machine. Le code de Centaure fait l'inverse : il récupère les données des metriques pour une machine spécifique. Par exemple, si vous voulez récupérer toutes les alarmes actives, le plus simple sera de faire un SHOW TABLES, puis de récupérer toutes les tables au format alarme_ID, et de faire un SELECT avec alarme_active = 1 dans chaqu'unes de ces tables.

### Bug exploit pour faire une requête ORDER BY ..

Il n'y a rien de prévu nativement dans MySQLManager actuellement pour rajouter d'autres conditions à la requête (ORDER BY, LIMIT, ect... )

Notez cependant qu'il y a un 'bug' dans le fonctionnement de MySQLManager que vous pouvez exploiter pour rajouter ces éléments à votre requête :
En fait, il suffit de fournir une valeur de where du type where = "alarme_active = 1 ORDER BY start_time". Le string sera directement concaténé au reste de la requête : requete_sql + "WHERE" + where, et la requête prendra donc en compte les commandes supplémentaires. Attention par contre à ne pas mettre un 'WHERE' au début de votre string sans quoi il apparaitra deux fois et la requête plantera !

> Bien sûr, comme évoqué précédemment, ajouter une solution plus perenne pour faire ce type de requête dans le futur pourrait être pertinent...

### Un point sur la méthode delete

delete est de loin la méthode la moins testée, car n'avait pas d'utilitée dans les applications métiers développées jusqu'à maintenant. Elle soufre cependant visiblement des mêmes problèmes que fetch, et il faut donc l'appeler avec les mêmes arguments pour obtenir le résultat souhaité. Au delà de ce point, je peux juste vous conseiller de tester de manière appronfondie toute opération faite avec cette méthode avant de la lancer sur la base (par exemple, et à minima, en faisant un fetch avant avec les mêmes arguments et en affichant le résultat pour voir quelles metriques sont réellement impactées).

## Quelques compléments sur SNMPRequestHandler

Dans la continuité du point précédent, voyons quelques éléments généraux sur SNMPRequestHandler.

Comme pour MySQLManager, SNMPRequestHandler gére une machine spécifique, dans son cas pour les requêtes SNMP.

SNMPRequestHandler possède un ensemble de méthode capable de faire des requêtes SNMP diverses, ainsi qu'une méthode de traitement des oid (capable de faire le lien entre des oids reçus en réponse des requêtes, et des oids envoyées). Cette méthode est primordiale car elle permet de requêter plusieurs oids en une seule fois, correspondant éventuellement à des metriques différentes, et à refaire le lien avec les metriques en question à partir des oids dans la réponse des requêtes. Elle fait aussi une sorte de "premier traitement" de la donnée, qui vise à standardiser toutes les données récupérées en les structurant au format attendu par les autres scripts ({'metrique': [{dictionnaire de donnée 1, dictionnaire de donnée 2, ... }, ... ]}).
La méthode run combine ces différentes méthodes pour récupérer toutes les valeurs liées à une série d'oid (plus exactement à une serie de metrique passées en entrée).

> La construction de la méthode run est très dépendante du type de machine à contacter et de leur structure, et dépend globalement de considérations liées au protocole SNMP. Par exemple, si on avait que des machines en SNMPv3, on pourrait faire une (unique) requête get sur tout les oids, puis une (unique) requête bulk (type de requête SNMP) sur tout les oids sur lesquels le get a échoué.

Il y a moins de points techniques à voir sur le code, nous aurons simplement à passer en revue les arguments d'entrée du constructeur.

### Structure du constructeur

Le constructeur de SNMPRequestHandler a la structure suivante :

```python
def __init__(
    self,
    address: str,
    name: str = '',
    requests: Optional[List[Dict[str, Any]]] = None,
    version: int = 2,
    community: str = 'public',
    username: str = '**username**',
    auth_protocol: str = 'MD5',
    auth_password: str = 'Col4Emd5',
    priv_protocol: str = 'AES',
    priv_password: str = 'Col4Eaes',
    port: int = 161,
    timeout: int = 1,
    retries: int = 1,
    options: Optional[Dict[str, Any]] = None,
    logger=None
) -> None:
```

Comme vous pouvez le voir, la plupart des champs sont liées à la configuration du protocole SNMP ou des couches inférieures.

**address :**
String. Adresse IP de la cible au format standard "10.90..".

**name :**
String. Nom descriptif de l’instance. On pourra par exemple mettre le NAME de la machine. Sert en fait uniquement à personnaliser les logs. Un nom par défaut sera généré si vous n'en fournissez pas.

**version :**
Int. Version du protocole SNMP à utiliser (2, 3).

**community :**
String. Chaîne de communauté SNMP. Ignoré si version=3.

**username :**
Nom d’utilisateur SNMPv3.

**auth_protocol :**
Protocole d’authentification SNMPv3 (`MD5`, `SHA`, etc.). L'ensemble des protocoles disponibles est visible dans un dictionnaire en haut du fichier `layers/request_handler/SNMPRequestHandler.py`.

**auth_password :**
Mot de passe pour l’authentification SNMPv3.

**priv_protocol :**
Protocole de chiffrement SNMPv3 (`AES`, `DES`, etc.). L'ensemble des protocoles disponibles est là encore visible dans un autre dictionnaire en haut du fichier `layers/request_handler/SNMPRequestHandler.py`.

**priv_password :**
Mot de passe pour le chiffrement SNMPv3.

**port :**
Int. Port SNMP de la cible (par défaut 161).

**timeout :**
Int. Durée maximale d’attente d’une réponse (secondes).

**retries :**
Int. Nombre de tentatives en cas d’échec.

**options :**
Dictionnaire d’options supplémentaires. Ce dictionnaire est complétement libre, mais dans ce code, on y renseigne en général un champ username et un champ id.
Le principal intérêt de ce champ est d'intéragir avec une méthode de l'instance, `load`. Cette méthode est appelée pour chaque élément passé en entrée de SNMPRequestHandler. Si l'élément est un string, et qu'il a le format \_\_label\_\_, `load` va vérifier si le champ `options` contient l'étiquette `username`. Si c'est le cas, elle remplace la valeur du champ par la valeur dans le champ `options`.

**requests :**
Le champ requests est le seul un peu spécifique. Il doit avoir un format très précis, à savoir le suivant :

    [
        {
            'oid': '1.3.6.4.2.12.134',
            'metric_name': 'metrique_1',
            'operators': 'operateur_1'
        }, // pour chaque oid à contacter, la liste doit contenir un dictionnaire de ce type
        ...
    ]

Vous pouvez ne pas fournir de champ operators. Ce champ n'est d'ailleurs jamais utilisé directement par SNMPRequestHandler, il ne sert que lors de l'enregistrement en base pour MySQLManager. Il est présent ici car, d'un point de vu métier, il semble plus logique d'associer l'operateur à l'oid ou à la metrique.
Vous pouvez également ne pas fournir de champ 'metric_name'. La requête se fera quand-même, mais la donnée sera enregistrée dans un champ spécifique, et ne sera pas prise en compte lors de l'enregistrement en base si vous passez votre SNMPRequestHandler à un MySQLManager.
Le programme ignorera les metriques qui n'ont pas d'oid.

**logger :**
Instance de logger pour les messages et avertissements, comme pour les autres constructeurs de classe.

Les autres méthodes importantes de SNMPRequestHandler ne prennent pas d'argument. Ainsi, vous pouvez simplement appeler .run() pour lancer la collecte une fois l'instance initialisée.

Il y a cependant un dernier point à évoquer sur cette classe.

### point sur la méthode update_results

Cette méthode est celle qui gère la jointure entre oid et metrique quand c'est nécessaire, et qui fait un premier formatage des données.
Ce qu'il y a d'important à dire sur cette méthode, c'est que c'est celle qu'on va vouloir réécrire si on veut changer les fonctionnalitées de SNMPRequestHandler. Ainsi, pour la vérification des tables d'alarme, j'ai réutilisé le même code mais réécris cette méthode pour lui permettre de gérer la reconnaissance d'oids spécifiques dans des arbres plus complexes, donc en changeant le pré-traitement effectué par cette méthode. En particulier, dans ce cas précis, j'ai créé deux autres méthodes pour huawei et vertiv qui gèrent la gestion des arbres d'oid propres à leur constructeur. Vous trouverez les détails sur cette méthode dans `domain/alarm_monitor/SNMPAlarmRequestHandler.py`.

## Gestion des traps et des alarmes

En plus de l'ensemble des dossiers et fichiers de code évoqués ci-dessus, vous trouverez dans layers les dossiers alarms_manager et event_listener. Ces deux fichiers sont en fait liés à la gestion des alarmes et des traps.

### Gestion des traps

La gestion des traps n'est pas aussi "standardisée" que les opérations de requêtage et d'enregistrement en base, et beaucoup de choses se font au niveau de domain. En fait, les traps dépendent surtout de event_listener, dont nous parlons juste en-dessous.

Pour le reste, le fichier principal de gestion des traps est `domain/traps_and_alarms/prcess_trap.py`. Je reviens dessus rapidement juste après.

### point sur event_listener

event_listener est une classe d'écoute de trap, basé sur pysnmp (qui, à partir d'une tram UDP, et capable de construire une trap SNMP). Elle a le constructeur suivant :

```python
__init__(self, device_configs: List[Dict[str, Any]], listen_address="0.0.0.0", port=162, trap_callback: Optional[Callable] = None)
```

listen_address et port servent à définir les trams IP et UDP sur lesquelles on écoutera. Il n'y a à priori aucunes raisons de les modifier. Précisons si besoin que des services d'écoute de ce type construisent toujours leur tram IP avec l'adresse 0.0.0.0 pour recevoir toutes les trams IP (sauf si on veut écouter une machine très spécifique, au quel cas on pourra éventuellement mettre l'IP de la machine).

device_configs est une liste de dictionnaire, où chaque dictionnaire contient une configuration SNMP d'une machine que l'on veut superviser. Le format correspond (volontairement) au format d'entrée de SNMPRequestHandler, à savoir :

    {
        'version': ..
        'community': ..
        'username': ..
        'auth_protocol': ..
        'auth__password': ..
        'priv_protocol': ..
        'priv_password': ..
        'engine_id': .. // on doit fournir en plus un engine_id
    }

Enfin, trap_callback attend une fonction. C'est en pratique dans cette fonction que le reste va opérer. SNMPTrapListener réceptionne et déchiffre les traps, puis appelle trap_callback en lui passant en entrée un dictionnaire au format :

    {
        '1.3.6.1..': 'valeur associe a cet oid',
        ...
    }

### Complément sur la gestion des traps

La suite de la logique liée à la gestion des traps se fait dans `domain/traps_and_alarms/process_trap.py`, qui contient la fonction process_trap, fonction qui est passée à event_listener.

Les traps renvoient un arbre d'OID avec plusieurs feuilles, donc plusieurs valeurs. L'idée est d'enregistrer ces valeurs comme une seule entrée (ligne) dans une table. C'est ce que fait process_trap.

Cette fonction se base sur les données dans la table TRAPS pour effectuer un certain nombre de traitement. Précisement, elle appelle la fonction generate_results dans `domain/traps_and_alarms/generate_results.py`, qui s'occupe de récupérer les données dans la table de TRAPS, et d'appliquer des opérateurs aux données, puis de les reformater au format standard attendu MySQLManager. process_trap récupère alors le data généré, et le passe ensuite à une instance de MySQLManager pour l'enregistrer en base.

process_trap (comme son nom, certe, ne l'indique pas) gère aussi les alarmes. Elle appelle pour ça la fonction process_alarm (dans `domain/traps_and_alarms/process_alarm.py`) qui prend en entrée la donnée des traps, la retraite un peu (le format des traps et des alarmes différe très légèrement), et utilise AlarmManager pour mettre à jour les alarmes à partir de ces données.

### Parentèse sur les opérateurs des traps

Comme évoqué juste au-dessus (et comme vous pouvez le voir dans l'interface historica dédiée aux traps) il existe un système d'opérateur pour les traps.
Ce système d'opérateur est indépendant de celui de MySQLManager, et prend effet en amont. Il est géré dans la fonction generate_results.
Les opérateurs sont importés depuis le fichier `operators/trap_operators.py`. Ainsi, si vous voulez rajouter un opérateur, il suffit de rajouter une fonction dans ce fichier, nommé avec le nom que vous voulez donner à votre opérateur.

## Gestion des alarmes (round 2)

Nous avons évoqué la classe AlarmManager dans la section précédente. En fait, cette classe est définie dans le dossier `layers/alarm_manager`. C'est une classe spécialisée dans l'enregistrement des alarmes en base.

Dans la suite de cette partie, nous rentrons plus en détail dans le fonctionnement de la gestion des alarmes.

Commençons par quelques éléments sur la logique de gestion des alarmes.

Du point de vue de Centaure, la gestion des alarmes consiste principalement en une gestion fine (ajout ou modification) des données en base.

Il n'y a pas de moyens formel précis pour récupérer les données des alarmes prévu par Centaure (tout est géré dans domain, comme c'est le cas de la gestion des traps décrite dans la partie précédente), et ceci principalement parce qu'il existe plusieurs approches pertinentes pour récupérer des données d'alarme, à savoir notamment :

- On peut se baser sur des listes d'OID donnant le statut des alarmes qu'on souhaite surveiller et récupérer ces données à intervalle régulier (on pourra alors utiliser la classe Scrapper).
- On peut récupérer les données des tables d'alarme actives, que beaucoup de constructeurs proposent (on pourra également utiliser la classe Scrapper dans ce cas, à condition de faire quelques modifications dans une version personnalisée de la classe SNMPRequestHandler). C'est l'une des solutions adoptées dans le code.
- on peut utiliser des OID de surveillance fournies dans les traps. Certains constructeurs renvoient en effet dans la trap un OID qui permet de connaître l'état de l'alarme qui a levé la trap ou de vérifier qu'elle est toujours active.
- enfin on peut tout simplement se baser sur les traps elles même, en activant l'alarme quand on reçoit une trap d'activation, ou en désactivant l'alarme si c'est une trap off. Cette approche est également utilisée dans la version SFR, et a déjà été décrite plus haut.

  Il est probablement en pratique pertinent de cumuler plusieurs approches (et ce que ne font pas, d'ailleurs, la plupart des outils de supervision !) car elles ont chacunes leurs avantages et leurs inconvénients, et plus simplement parce qu'il est compliqué d'obtenir toutes les alarmes avec une seule approche. On pourra alors faire plusieurs services gérant chacun les alarmes de manière asynchrone et décorélé, mais en passant systèmatiquement par la même classe (AlarmManager).

  Comme évoqué précédemment, Centaure ne gère pas directement de manière standardisée la récupération et la standardisation des données pour les alarmes. Il impose cependant que les données en base contiennent un élément ref, et un élément alarme_active. Les alarmes sont gérées par les alarm_manager, qui sont des classes héritant à la fois de BaseAlarmManager, et d'un DatabaseManager capable de gérer des opérations d'écriture, lecture, et modifications en base. On obtiens ainsi une nouvelle classe capable de réaliser l'opération "treat". Cette opération ajoute et/ou insert en base des données de manière conditionnelle : elle modifie la donnée si celle-ci est présente en base mais que le champ alarme_active n'a pas la même valeur qu'en entrée, et insert la donnée en base si celle-ci n'est pas déjà présente avec le champ alarme_active à 1.

Mais comment le code arrive à lier les données en entrée avec les données en base ? Comment sait-on qu'une alarme est active en base ?
C'est en utilisant l'argument ref : ref est un bigint permettant d'identifier de manière unique un type d'alarme. Cet argument peut être fourni par la trap. Certains constructeurs associent en effet une ref à chaque type d'alarme existante pour leur équipement. Cette ref sert suite d'identifiant de l'alarme : il ne peut pas y avoir 2 alarmes avec la même ref active en base (donc avec le champ alarme_active à 1). Logiquement, le constructeur du matériel supervisé ne devrait pas permettre à deux alarme avec la même ref d'être active en même temps sur son équipement.
En pratique les constructeurs ne transmettent pas toujours la ref des alarmes, que ce soit via les traps au via de la collecte directe (requête snmpget sur l'équipement). Il peut donc être pertinent d'utiliser un autre élément envoyé par les équipements pour générer soi-même les ref. Les équipements renvoient par exemple presque toujours un string unique à l'alarme, contenant des informations en clair sur la nature de l'alarme (et qui est parfois personnalisable). Il peut être pertinent utiliser ce string (ou un autre) pour générer une ref unique à un type d'alarme. La bibliothèque contient d'ailleurs une fonction generate_ref qui permet de générer un bigint unique à partir d'un string. C'est cette approche qui est utilisée dans ce code. Elle a l'énorme avantage d'être généralisable à plusieurs constructeurs dont les formats de traps et d'alarme varient.
Il existe probablement beaucoup d'autres manières pertinentes de générer et gérer les refs. Vous pouvez toujours personnaliser librement les refs, tant que vous fournissez un élément 'ref' pour chaque alarmes en entrée des AlarmManager.

Un exemple de code pour finir ?
En fait, il n'y a pas grand chose à dire de plus car MySQLAlarmManager s'utilise exactement comme MySQLManager. Voyez l'exemple ci-dessous :

```python
recorder = MySQLAlarmManager(
    SNMPAlarmRequestHandler(address, name=name, requests=requests, **config, options=machine_data, logger=self.logger), # une classe héritant de SNMPRequestHandler, et qu'on aura modifier pour qu'elle nous renvoie toutes les données voulues. On aurait aussi très bien pu fournir un champ data.
    table_struct=['ref', 'equipement', 'info', 'message_erreur', 'alarme_active', 'start_time', 'end_time'], # une structure possible des alarmes en table
    logger=self.logger # le logger
)

recorder.object.run() # on récupère les données des alarmes sur l'équipement via SNMPRequestHandler
recorder.treat() # la seule particularité de MySQLAlarmManager : on peut utiliser la méthode treat, qui va donc enregistrer les données en base si leur ref ne correspond à aucune alarme active, et qui va modifier les alarmes actives sinon, en fonction de la valeur du champ alarme_active.
```

Dans notre cas, pour la gestion des alarmes, on a envi d'également faire une chose en plus :
On veut générer le dictionnaire exacte des alarmes à mettre à off ou à activer, à partir des alarmes qu'on a récupérés comme active sur l'équipement, et en les comparant aux alarmes réellement actives dans notre base. On rajoute pour cela une méthode `merge_fetched_with_results` dans la classe MySQLAlarmManager, qui va comparer les valeurs dans data (donc les valeurs récupérées par l'instance de RequestHandler), et les valeurs de fetched_data (donc des valeurs qu'on récupère en base) pour définir les modifications à faire en base. Avec cette nouvelle méthode, on peut ainsi utiliser le code suivant :

```python
recorder = MySQLAlarmManager(
    SNMPAlarmRequestHandler(address, name=name, requests=requests, **config, options=machine_data, logger=self.logger), # une classe héritant de SNMPRequestHandler, et qu'on aura modifié pour qu'elle nous renvoie toutes les données voulues. On aurait aussi très bien pu fournir un champ data.
    table_struct=['ref', 'equipement', 'info', 'message_erreur', 'alarme_active', 'start_time', 'end_time'], # une structure possible des alarmes en table
    logger=self.logger # le logger
)
recorder.object.run() # on récupère les données des alarmes sur l'équipement via SNMPRequestHandler

recorder.fetch(where = "alarme_active = 1") # on récupère les alarmes actives en base

recorder.merge_fetched_with_results() # si une alarme renvoyée par l'équipement n'est pas dans la base, on la laisse pour qu'elle soit activée, et si une alarme active en base n'est pas renvoyée par l'équipement, c'est qu'elle n'est probablement plus active, donc on la rajoute avec alarme_active = 0 (et un end_time). On remplace ensuite data par cette nouvelle valeur avec la méthode set_data()

recorder.treat() # on effectue les treat, qui va cette fois opérer sur les données "mergés", donc éventuellement passer à off certaines alarmes
```

En pratique, c'est ce code qui est utilisé, mais il est exploité dans la classe AlarmSNMPScrapper (qui hérite de SNMPScrapper). Vous pouvez consulter le code dans `domain/alarm_monitor` pour plus de détails.

## alerter

Les alerteurs (ou alerters dans le code) sont un ensemble de classe capable de récupérer des données en base, vérifier un certain nombre de condition sur ces données, et, si les conditions sont vérifiées, envoyer des emails.

La classe principale est `layers/alerter/MySQLAlerter.py`. Cette classe hérite de BaseScrapper, et récupère donc certaines des méthodes des scrappers.

MySQLAlerter gère l'ensemble des vérifications liées à une alerte. Une alerte peut porter sur plusieurs machines, et la classe vérifiera alors les conditions de l'alerte sur toutes les machines demandées. Voyons ses arguments d'entrée :

```python
    def __init__(self, conditions: Union[List[Dict], str], name: str = 'alerter', machines: Union[List[str], str] = [], construct: str = "", limit_date: int = 0, emails: Union[List[str], str] = [], logger: logging.Logger = None)
```

**name :** C'est un nom, qui aparaitra dans certains logs d'erreur et dans l'email envoyé le cas échéant. Il n'a aucunes autres influence sur l'execution du code.

**machines :** C'est une liste de string (les noms des machines tel qu'indiqué dans historica). La classe va récupérer en interne les données liées à ces machines depuis la base historica, principalement l'ID, et récupérer les données correspondant à ces machines en base.

**construct :** String. Vous pouvez renseigner un constructeur plutôt que des noms de machine en base. Toutes les machines liées à ce constructeur seront alors récupérées. Vous devez renseigner exactement le champ construct indiqué en base (il n'y a aucun traitement particulier fait sur ce champ). Notez que si vous fournissez un constructeur et un champ machines, la requête récupérera tout ce qui correspond à l'un des deux paramètres au moins.

**limit_date :** Attend un timestamp UNIX (donc un int). C'est une des conditions sur les données en base. Si vous indiquez un timestamp correspondant à l'heure courant moins 6 heures, l'alerteur vérifiera vos conditions sur toutes les données en base sur les 6 dernières heures.

**emails :** Liste de str, ou str si un seul email ou si une liste d'email séparé par des ','. Si l'alerte est déclanchée, l'email sera envoyé à tout les emails renseignés dans ce champ.

**conditions :** List ou JSON (string au format JSON). Contient une liste du type :

    [
        {
            "metrique":"phase_1_courant ; phase_2_courant ; phase_3_courant",
            "type_condition":"seuil",
            "valeur_max":"100",
            "valeur_min":"",
            "valeur_egale":"",
            "valeur_differente":"2147483647",
            "nombre_valeurs_seuil":1,
            "logic_operator":"ET"
        },
        ...
    ]

Ce JSON contient en fait la liste des conditions de déclanchement de l'alerte, avec pour chaque condition toutes les informations nécessaires précisant la nature de l'alerte. Si vous voulez des précisions sur chaque champ, consultez la page historica des alertes.

Cette classe est la plus récente, et je n'ai donc pas eu de raison pour l'instant de "généraliser" son fonctionnement. Ainsi, elle est prévue pour fonctionner dans le cadre d'un code spécifique uniquement :

```python
def get_past_timestamp(heures=0, jours=0, semaines=0):
    """
    Retourne le timestamp (int) correspondant à la date/heure actuelle
    moins le nombre d'heures, jours et semaines fourni.

    :param heures: Nombre d'heures à soustraire
    :param jours: Nombre de jours à soustraire
    :param semaines: Nombre de semaines à soustraire
    :return: Timestamp UNIX (int)
    """
    maintenant = datetime.now()
    delta = timedelta(hours=heures, days=jours, weeks=semaines)
    date_calculee = maintenant - delta
    return int(date_calculee.timestamp())

def get_alerter(name):
    """
    Récupère les informations de l'alerteur 'name' dans la table ALERTEURS
    et retourne un dictionnaire avec les infos clés.
    """

    conn = mysql.connector.connect(**ae_db_config)
    cursor = conn.cursor(dictionary=True)

    query = """
        SELECT nom, constructeur, ae_specifiques, conditions, emails, semaines, jours, heures
        FROM ALERTEURS
        WHERE active = 1 AND nom = %s
    """
    cursor.execute(query, (name,))
    row = cursor.fetchone()

    cursor.close()
    conn.close()

    if not row:
        return None # Aucun résultat trouvé

    timestamp = get_past_timestamp( # Calcul du timestamp limite
        heures=row['heures'] or 0,
        jours=row['jours'] or 0,
        semaines=row['semaines'] or 0
    )

    return {
        'name': row['nom'],
        'construct': row['constructeur'],
        'limit_date': timestamp,
        'emails': row['emails'],
        'conditions': row['conditions']
    }

if __name__ == "__main__": # le code principal est ici
    alerter_info = get_alerter("phase haute") # on récupère un alerteur appelé "phase haute" ici
    alerter = MySQLAlerter(**alerter_info, logger = logger)
    alerter.check() # on lance la vérification avec check, qui enverra également un email le l'alerte est déclanchée
```

Si vous voulez maintenant vérifier et déclancher toutes les alertes en base, il suffit de modifier get_alerter pour qu'il renvoit toutes les alertes en base plutôt qu'une seule spécifique. 

Dans la pratique, la fonction get_alerter est dans `domain/alerter/get_alerter.py`.

## process_manager

Le dernier dossier présent dans layers et qui n'a pas encore été évoqué est process_manager.
Il y a dans ce dossier plusieurs outils pour gérer des process. En fait, comme les opérations de collecte sont longues, on voudra parfois faire en sorte qu'une nouvelle collecte ne puisse pas se lancer tant que l'encienne n'est pas terminée. Le but étant de ne pas surcharger la RAM du serveur faisant tourner le script, en faisant tourner 5, 10, 30 fois le même script en parallèle. Ce besoin est d'autant plus important que de nombreuses classes de centaure sont assez gourmandes en RAM.
Voici donc quelques exemples permettant d'utiliser ce code :

```python
from layers.process_manager.manage_processes import has_active_processes_in_list, add_pid_to_list, remove_pid_from_list
import os

if has_active_processes_in_list(): # permet de vérifier si un autre processus est déjà actif
        print("Arret de l'execution : un autre processus est actif.")
        sys.exit()

pid = os.getpid()
add_pid_to_list(pid)

# executez ici le code voulu

remove_pid_from_list(pid)
```

Vous pouvez également imposer une limite au nombre de process actif en même temps avec `howmany_pid_in_list`:

```python
from layers.process_manager.manage_processes import has_active_processes_in_list, add_pid_to_list, howmany_pid_in_list, remove_pid_from_list
import os

K = 5 # le nombre de process actif en même temps, personnalisable
if has_active_processes_in_list() and howmany_pid_in_list() > K: # on rajoute une condition ici
    print(f"Arret de l'execution : {K} autres processus actifs.")
    sys.exit()

pid = os.getpid()
add_pid_to_list(pid)

# executez ici le code voulu

remove_pid_from_list(pid)
```

> Notez que tout les process que vous passez à cette fonction et qui sont encore actifs sont pris en compte.

> Il n'est pas dramatique d'oublier d'appeler remove_pid_from_list, c'est même obstionnelle. En fait, has_active_processes_in_list vérifie si les process renseignés sont encore actifs. C'est également pour ça que je vous conseille de toujours appeler has_active_processes_in_list, même si vous ne voulez utiliser que howmany_pid_in_list, car ce dernier ne fait pas de vérification.

> Vous trouverez dans le même fichier la fonction kill_process_in_list, qui permet tout simplement de forcer l'arrêt de tout les process actifs renseignés.

Et.. c'est tout pour process_manager !
