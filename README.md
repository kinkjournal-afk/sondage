# Sondage anonyme — déploiement

Le site (`index.html` + `formulaire.html`) va sur GitHub Pages. Le stockage des
réponses se fait via un petit Worker Cloudflare + KV. Pas de base de données à
gérer, pas de mot de passe dans le code.

## 1. Héberger le site sur GitHub Pages

1. Crée un nouveau repo GitHub (public ou privé selon tes besoins).
2. Mets-y `index.html`, `formulaire.html` et `style.css` à la racine.
3. Dans **Settings > Pages**, choisis la branche `main` et le dossier `/root`.
4. Ton site sera accessible à `https://TON-PSEUDO.github.io/NOM-DU-REPO/`.

## 2. Déployer le Worker Cloudflare

Prérequis : un compte Cloudflare (gratuit) et Node.js installé.

```bash
npm install -g wrangler
wrangler login

# Crée le namespace KV qui stockera les réponses
wrangler kv namespace create SONDAGE_KV
```

Copie l'`id` renvoyé dans `wrangler.toml`, à la place de `REMPLACE_MOI`.

Modifie `ALLOWED_ORIGIN` dans `wrangler.toml` avec l'URL exacte de ton site
GitHub Pages (sans slash final).

Définis le secret d'administration (sert à récupérer les réponses ensuite) :

```bash
wrangler secret put ADMIN_KEY
# entre une longue chaîne aléatoire quand demandé, par ex. générée avec :
# openssl rand -hex 32
```

Déploie :

```bash
wrangler deploy
```

Wrangler te donne une URL du type
`https://sondage-worker.TON-SOUS-DOMAINE.workers.dev`.

## 3. Relier le formulaire au Worker

Dans `formulaire.html`, remplace :

```js
const ENDPOINT = "https://TON-WORKER.TON-SOUS-DOMAINE.workers.dev/submit";
```

par l'URL réelle donnée par `wrangler deploy`, en gardant `/submit` à la fin.
Recommite et repousse sur GitHub.

## 4. Récupérer les réponses

```bash
curl -H "Authorization: Bearer TA_ADMIN_KEY" \
  https://sondage-worker.TON-SOUS-DOMAINE.workers.dev/export
```

Ça renvoie toutes les réponses en JSON, sans IP ni aucune donnée de contact —
elles ne sont pas collectées.

## Ce qui a changé par rapport à ta version initiale

- Plus de champ Snapchat, plus de canaux de diffusion (Discord/Snap/Omegle) :
  rien qui pousse à diffuser le lien sur des espaces fréquentés par des
  mineurs, et rien qui permette de recontacter un répondant.
- Plus d'IP stockée.
- La limite d'âge de 18 ans est vérifiée **côté serveur** dans le Worker (le
  champ côté client peut être contourné, celui du Worker non).
- Plus d'identifiants de base de données en clair dans le code : la seule clé
  sensible (`ADMIN_KEY`) est un secret Cloudflare chiffré, jamais commité.
- Les champs envoyés sont validés et bornés côté serveur (type, plage de
  valeurs), donc même une requête bricolée à la main ne peut pas insérer de
  champ ou de valeur hors de la liste prévue.

## Limite à connaître

La case à cocher "je confirme avoir 18 ans" reste une déclaration sur
l'honneur : ni ce dispositif ni la plupart des sondages en ligne ne
vérifient une identité réelle. C'est un choix assumé pour un sondage
anonyme, mais ce n'est pas une vérification d'âge au sens légal du terme.
