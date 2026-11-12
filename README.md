# home-links

Un petit dashboard (Vite + Vue 3) pour accéder rapidement à tous les outils de ton archi DevOps (Harbor, ArgoCD, Umami, Rancher, etc.).

Tous les liens sont configurables via un unique fichier JSON, sans avoir à toucher au code.

## Démarrer

```bash
npm install
npm run dev
```

## Configurer les liens

Édite `public/links.json`. Le fichier est chargé au runtime (fetch), donc en dev un simple refresh du navigateur suffit — pas besoin de rebuild.

```json
{
  "title": "DevOps Links",
  "subtitle": "Tous mes outils d'infra, au même endroit",
  "categories": [
    {
      "name": "GitOps & CI/CD",
      "links": [
        {
          "name": "ArgoCD",
          "url": "https://argocd.example.com",
          "description": "Déploiement continu GitOps",
          "icon": "🚀",
          "color": "#ef7b4d"
        }
      ]
    }
  ]
}
```

Champs par lien :

- `name` (obligatoire) : nom affiché
- `url` (obligatoire) : lien cliquable
- `description` (optionnel) : sous-titre
- `icon` (optionnel) : soit un emoji/caractère, soit un chemin/URL vers un logo (`.svg`, `.png`, `.webp`...). Toute valeur commençant par `/`, `./`, `http://` ou `https://` est traitée comme une image et affichée dans un chip blanc arrondi ; sinon elle est affichée telle quelle (emoji).
- `color` (optionnel) : couleur d'accent au survol de la carte

La barre de recherche filtre sur le nom, la description et l'URL.

### Logos des outils

Les logos fournis par défaut (`public/icons/*.svg` : ArgoCD, Harbor, Rancher, Umami) viennent de la librairie open source [dashboard-icons](https://github.com/homarr-labs/dashboard-icons) (Apache-2.0), qui référence plus de 1800 logos d'outils self-hosted.

Pour ajouter le logo d'un nouvel outil :

```bash
curl -o public/icons/<nom>.svg https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/<nom>.svg
```

puis référence-le dans `links.json` avec `"icon": "/icons/<nom>.svg"`. Cherche le nom exact sur [dashboardicons.com](https://dashboardicons.com).

### Mot de passe d'accès

Un écran de mot de passe basique protège l'accès à la page. Il est configurable dans `links.json` :

```json
{
  "auth": {
    "enabled": true,
    "passwordHash": "<sha256 du mot de passe>"
  }
}
```

Pour changer le mot de passe, génère le hash SHA-256 puis remplace `passwordHash` :

```bash
printf '%s' 'monNouveauMotDePasse' | sha256sum
```

Pour désactiver la protection, passe `enabled` à `false` (ou supprime le bloc `auth`).

**Important : ce n'est pas une vraie protection de sécurité.** L'app est une SPA 100% statique : le fichier `links.json` (donc le hash) et tout le JS sont téléchargeables par n'importe qui ayant accès réseau à la page, et le hash pourrait être cassé par une attaque par dictionnaire/brute-force hors-ligne. Ça sert juste à éviter qu'un clic accidentel ou une personne non-technique tombe sur la page. Pour une vraie protection, mets une couche devant (Basic Auth nginx, middleware Traefik `forward-auth`/`basicAuth`, VPN, etc.).

## Build de production

```bash
npm run build
npm run preview
```

Le fichier `public/links.json` est copié tel quel dans `dist/`, tu peux donc aussi le modifier directement après le build (ex: monté en volume Docker) sans rebuild.

## Docker

```bash
docker compose up -d --build
```

L'app est alors dispo sur [http://localhost:8080](http://localhost:8080). `public/links.json` est monté en volume (lecture seule) dans le conteneur : tu peux l'éditer sur l'hôte, un simple refresh du navigateur suffit (le fichier est servi avec `Cache-Control: no-store`).

Build/run manuel sans compose :

```bash
docker build -t home-links .
docker run --rm -p 8080:80 -v $(pwd)/public/links.json:/usr/share/nginx/html/links.json:ro home-links
```

## CI/CD

Le workflow `.github/workflows/docker-build.yml` build l'image Docker et la pousse sur Harbor (`${HARBOR_REGISTRY}/${HARBOR_PROJECT}/home-links:latest`) à chaque push, puis met à jour le manifeste ArgoCD dans le repo `Paul-Garcia/Gitops` (`apps/home-links/prod|dev/deployment.yaml`) avec le digest de l'image.

Secrets/variables GitHub à configurer sur le repo :

- `HARBOR_USERNAME`, `HARBOR_PASSWORD` — identifiants du registre Harbor
- `GITOPS_TOKEN` — token avec accès push sur `Paul-Garcia/Gitops`
- (optionnel) variables `HARBOR_REGISTRY`, `HARBOR_PROJECT` si différentes des valeurs par défaut (`harbor.kirby.love` / `library`)

Assure-toi que le fichier `apps/home-links/prod/deployment.yaml` (et `dev/`) existe côté repo GitOps, avec une ligne `image: ...home-links...` à faire correspondre par le `sed` du workflow.
