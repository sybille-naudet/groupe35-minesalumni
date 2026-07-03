# PROJECT.md — CRM Mines Paris Alumni

## Contexte

Les alumni contactent régulièrement l'association pour obtenir diverses informations. Ils sont parfois cotisants, envoient des mails depuis des adresses e-mail différentes de celle enregistrée dans l'annuaire officiel (adresse pro, perso, ancienne adresse école...). Il est difficile de savoir en un coup d'œil, en ouvrant un mail dans Gmail, qui écrit et si sa cotisation est à jour. L'objectif du projet est d'automatiser cette vérification directement dans Gmail grâce à la création d'un Widget, et de proposer une relance pré-rédigée quand c'est pertinent (cotisation à renouveler, ou adresse à mettre à jour dans l'annuaire).

## Démarche
 
Nous avons commencé avec une certaine base déjà construite par Théophile Cantelobre, notamment pour l'authentification. 

D'emblée, et ayant fait le début du code avec l'aide de l'IA, il nous a indiqué que nous pouvions utiliser l'IA, nous montrant même comment lui l'utilisait au mieux. 
 
Le projet a été découpé en **deux briques indépendantes reliées par une API HTTP**, plutôt qu'un seul bloc de code mêlant logique métier et affichage :
 
1. **Backend** (Python, FastAPI, SQLModel, SQLite) : il porte toute la logique « intelligente » — retrouver un membre à partir d'une adresse e-mail parmi plusieurs possibles, décider s'il faut relancer la personne et sur quel sujet (cotisation, adresse à mettre à jour), et rédiger le texte correspondant. Il expose un unique endpoint, `POST /context`, qui prend un e-mail en entrée et renvoie le profil du membre trouvé (ou rien) ainsi qu'un message de relance HTML déjà prêt à être envoyé, ou `null` si aucune relance n'est nécessaire.

2. **Complément Gmail** (Google Apps Script, `CardService`) : c'est la couche d'affichage. À l'ouverture d'un e-mail, il récupère l'adresse de l'expéditeur (`onGmailMessageOpen`), interroge le backend via `UrlFetchApp.fetch`, et se contente d'afficher ce qu'on lui répond — le nom, la promo, le statut, et, si un message de relance existe, un bouton qui crée un brouillon de réponse pré-rempli dans le fil de discussion.
Ce découpage en couches « intelligence côté backend / affichage côté client » n'est pas qu'une préférence esthétique : c'est ce qui permet de faire évoluer une règle métier (par exemple ajouter un troisième cas de relance, ou changer le texte d'un message) **uniquement côté backend**, sans avoir à retoucher, retester et redéployer le complément Gmail — qui est plus contraignant à mettre à jour (redéploiement Apps Script, cache Gmail à vider, etc., comme on l'a vu pendant le développement).
 
Concrètement, la démarche de développement a suivi cet ordre :
1. Modéliser les données (un membre peut avoir jusqu'à 3 adresses e-mail) et charger un premier jeu de données de test (`members.csv` → `seed.py`) pour avoir quelque chose à interroger dès le départ.
2. Construire et valider l'endpoint `/context` indépendamment du complément Gmail (via `/docs` ou `curl`), pour s'assurer que la logique de recherche et de génération des messages était correcte avant de brancher l'interface.
3. Brancher le complément Gmail sur cet endpoint, et itérer sur l'affichage (carte, bouton de réponse, gestion des paramètres et de la liste noire).


## Choix techniques
 
- **FastAPI + SQLModel** : FastAPI permet de définir en une seule classe à la fois le modèle de base de données et le schéma de validation/sérialisation, ce qui évite de dupliquer la définition d'un `Member` entre la couche base de données et la couche API — un gain de temps et une source d'erreurs en moins pour un projet de cette taille.


- **Gestion de plusieurs adresses par membre** (`mail_principal`, `mail_secondaire`, `mail_alternatif`) : c'était une contrainte imposée par la réalité des données — un même alumni écrit parfois avec son adresse pro, parfois avec une adresse perso, parfois avec une ancienne adresse école. Plutôt que de forcer une seule adresse par fiche (et perdre le contact si la personne écrit d'ailleurs), on interroge les trois colonnes en parallèle, en comparant les adresses en minuscules (`func.lower(...)`) pour ne pas rater une correspondance à cause d'une différence de casse. Cela permet aussi de détecter un cas utile : la personne est bien connue, mais elle n'écrit pas depuis son adresse principale — ce qui déclenche un message dédié pour proposer de mettre à jour l'annuaire.

- **Génération du message de relance en HTML, côté backend** : Toute la logique (quel texte, dans quel cas, avec quelles variables comme le nom ou l'adresse principale) est centralisée dans `main.py`. Avantages : un seul endroit à modifier pour changer un texte ou ajouter un cas de relance, sans toucher au code Apps Script ; et une logique testable indépendamment de Gmail. Le format HTML a été choisi pour pouvoir directement peupler le corps d'un brouillon Gmail (`createDraftReply(..., {htmlBody: suggestion})`) avec une mise en forme minimale (paragraphes), plutôt que du texte brut à reformater côté client.

- **Authentification par clé API simple** (header `X-API-Key` ou `Bearer`, comparée à une liste définie dans la variable d'environnement `API_KEYS`) : pour un usage interne à une association, avec un nombre limité d'utilisateurs, une clé partagée par personne/usage est suffisante et beaucoup plus rapide à mettre en place qu'un système d'authentification complet (OAuth, comptes utilisateurs, etc.). Elle reste simple à distribuer et à révoquer : il suffit de retirer la clé de la liste côté serveur pour couper l'accès.
- **Liste noire gérée côté client** (`PropertiesService` dans Apps Script, propre à chaque utilisateur du complément) : ce choix évite d'alourdir le backend avec une notion de préférences par utilisateur du complément (qui n'a pas de compte à proprement parler côté API, seulement une clé). Chaque personne qui utilise le complément peut ainsi exclure certaines adresses ou certains domaines de l'affichage (par exemple des newsletters ou des adresses internes) sans que cela affecte les autres utilisateurs ni nécessite une modification côté serveur.


## Difficultés rencontrées et solutions

1. **Inferface Google.** La difficulté principale réside dans le fait que nous devions coder à la fois sur VS Code et dans App Script (directement connecté à Google). Il fallait donc toujours avoir les bonnes clés API, les bons accès, ce qui nous a été complexe à comprendre au départ.

2. **Conflits Git non résolus.** Le fichier `Code.gs` contenait encore des marqueurs de fusion (`<<<<<<< HEAD` / `=======` / `>>>>>>>`) suite à un merge mal terminé, rendant le fichier syntaxiquement invalide. → Résolu en choisissant, bloc par bloc, la version la plus aboutie (gestion du fil de discussion via `createDraftReply`, textes en français, prise en compte du `messageId`).



## Ce que nous ferions différemment avec plus de temps

- Écrire des tests automatisés (`pytest`) pour l'endpoint `/context` et pour la génération des messages de relance, afin d'éviter les régressions comme celles rencontrées.
- Historiser les relances envoyées, pour éviter de solliciter deux fois la même personne à quelques jours d'intervalle.
- Affiner la gestion des clés API (droits différenciés, expiration, révocation individuelle).
- Ajouter une fonction qui communique sur les actualités MPA se déroulant à proximité
