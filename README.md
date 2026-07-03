# groupe35-minesalumni

# CRM Mines Paris Alumni

Cet outil est outil qui est un complément Gmail qui affiche, à l'ouverture d'un e-mail, la fiche annuaire de l'expéditeur (nom, promo, ville, statut de cotisation) et propose un message de relance pré-rédigé si besoin (cotisation à jour, ou adresse e-mail différente de l'adresse principale).

Le projet a deux parties :
- **Backend** (Python / FastAPI / SQLModel) : expose une API qui recherche un membre par e-mail.
- **Complément Gmail** (Google Apps Script) : interroge le backend et affiche le résultat dans Gmail.

## 1. Installation du backend

**Prérequis** : Python 3.11+, [uv](https://docs.astral.sh/uv/) (ou pip).

```bash
git clone <url-du-repo>
cd <dossier-backend>
uv sync                        # installe les dépendances (ou : pip install -e .)
cp .env.example .env           # puis renseigner les variables (voir ci-dessous)
```

**Variables `.env` principales** (voir `settings.py`) :
```
DATABASE_URL=sqlite:///crm.db
API_KEYS=crm_xxx,crm_yyy        # clés d'accès à l'API, une par utilisateur/usage
COTISATION_LINK=https://...     # lien envoyé dans le mail de relance
```

Générer une clé API :
```bash
python -c "import secrets; print('crm_' + secrets.token_urlsafe(32))"
```

**Charger les données** (à partir de `data/members.csv`, colonnes `mail_principal, mail_secondaire, mail_alternatif, name, promo, city, last_active, membership_active, last_event`) :
```bash
uv run python scripts/seed.py
```

**Lancer le serveur** :
```bash
uv run uvicorn crm_mpa.main:app --reload
```
Le backend tourne sur `http://localhost:8000`. Documentation interactive : `http://localhost:8000/docs`.

## 2. Installation du complément Gmail

**Prérequis** : compte Google, [clasp](https://github.com/google/clasp) (`npm i -g @google/clasp`).

```bash
cd <dossier-gmail-addon>
clasp login
clasp push
```

Dans l'éditeur Apps Script (`clasp open`) :
1. **Propriétés du script** → ajouter `BACKEND_BASE_URL` = URL du backend déployé (ex. `https://votre-backend.fly.dev`).
2. **Déployer → Nouveau déploiement → Complément Google Workspace**.
3. Installer le complément sur son compte Gmail.

Dans Gmail, ouvrir le complément (icône dans la barre latérale) → **Paramètres** → coller la clé API générée à l'étape 1.

## 3. Utilisation

- Ouvrir un e-mail dans Gmail : la fiche du contact s'affiche automatiquement si son adresse (principale, secondaire ou perso) est reconnue dans l'annuaire.
- Si la cotisation n'est pas à jour, **ou** si le message vient d'une adresse différente de l'adresse principale enregistrée, un message de relance apparaît avec un bouton **« Répondre avec ce message »** qui crée un brouillon de réponse dans le fil en cours.
- **Paramètres** (icône engrenage) : renseigner/modifier la clé API, et gérer une liste noire d'adresses/domaines à ignorer (séparés par des virgules, ex. `spam@x.com, @newsletter.com`).


