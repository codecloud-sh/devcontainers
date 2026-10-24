# devcontainers

Template [Copier](https://copier.readthedocs.io/) pour générer le dossier `.devcontainer` de mes projets.

Stacks disponibles : **Ruby on Rails** et **Astro** (une par dossier, choisie par la première question).

## Utilisation

Dans le dossier du projet :

```bash
uvx copier copy gh:codecloud-sh/devcontainers .
```

Copier pose quelques questions, puis écrit `.devcontainer/`. Les réponses sont enregistrées dans `.devcontainer/.copier-answers.yml`, à commiter avec le reste.

Pour récupérer plus tard les évolutions du template (projet sous Git, sans modifications en cours) :

```bash
uvx copier update -a .devcontainer/.copier-answers.yml
```

Copier réapplique les changements du template en gardant les modifications faites dans le projet. En cas de conflit, il laisse des marqueurs à résoudre comme pour un merge.

Les projets générés quand le repo s'appelait `nicolascochin/devcontainers` gardent cette adresse dans le `_src_path` de `.devcontainer/.copier-answers.yml`. GitHub redirige l'ancienne adresse, donc `copier update` marche toujours, mais il vaut mieux la remplacer par `gh:codecloud-sh/devcontainers` et commiter.

## Questions

| Question | Stack | Défaut | Effet |
| --- | --- | --- | --- |
| `stack` | toutes | Ruby on Rails | Dossier du template utilisé (`rails/` ou `astro/`) |
| `project_name` | toutes | nom du dossier | Nom du devcontainer dans VS Code |
| `project_slug` | toutes | dérivé du nom | Hostname du conteneur et préfixe des volumes. Pour Rails, aussi nom du projet compose et dossier de travail (`/workspaces/<slug>`) |
| `username` | toutes | `dev` | Utilisateur créé dans le conteneur |
| `database` | Rails | SQLite | Feature client, service compose et `DB_HOST` pour PostgreSQL ou MySQL (plus `DB_USER` et `DB_PASSWORD` pour PostgreSQL) |
| `redis` | Rails | non | Service Valkey et `REDIS_URL` |
| `system_tests` | Rails | oui | Service Selenium (jusqu'à 4 navigateurs en parallèle, 2 Go de `/dev/shm`), `SELENIUM_HOST` et `CAPYBARA_SERVER_PORT` |

Pour ne pas retaper les mêmes réponses, Copier lit des valeurs par défaut dans `~/.config/copier/settings.yml` sous Linux (ou dans le fichier indiqué par `COPIER_SETTINGS_PATH`) :

```yaml
defaults:
  username: nico
```

## Contenu commun

- Image Debian slim, utilisateur non root avec sudo et zsh.
- mise installé dans l'image par la feature `devcontainers-extra/features/mise` (`/usr/local/bin/mise`). Il installe ce que le projet déclare : Ruby (`.ruby-version`), Node (`.node-version`) ou Bun (`.bun-version`). Avec Bun, le `.bun-version` est à créer soi-même : `bun --version > .bun-version`.
- Ce qu'installe mise (Ruby, Node ou Bun, gems) dans un volume nommé, `<slug>_mise-data` : une reconstruction du conteneur ne le réinstalle pas. Pour repartir de zéro, par exemple après une montée de version de Debian dans l'image : `podman volume rm <slug>_mise-data` (ou `docker volume rm`).
- Un vrai PID 1 (`init`) : le conteneur s'arrête tout de suite et les processus zombies sont récoltés.
- Configuration de Claude Code dans le `tmp/` du projet : elle survit à une reconstruction du conteneur.
- Connexion de `gh` dans un volume nommé commun à tous les projets de la machine, Rails comme Astro, `devcontainers-gh` (monté sur `~/.config/gh`) : un seul jeton par machine. GitHub garde au plus 10 jetons de GitHub CLI par compte et révoque au-delà le moins récemment utilisé : avec un jeton par projet, rouvrir un vieux projet finissait en `HTTP 401: Bad credentials`. Hors du workspace, le volume survit aux reconstructions et n'est pas synchronisé entre machines. Pour se déconnecter de tous les conteneurs de la machine : `podman volume rm devcontainers-gh` (ou `docker volume rm`), conteneurs arrêtés. Un `docker compose down -v` sur un projet Rails le supprime aussi : il suffit de se reconnecter.
- Projets générés avant le volume `devcontainers-gh` : après `copier update` et une reconstruction, `tmp/gh` ne sert plus et peut être supprimé.
- Réglages utilisateur de Claude Code (`tmp/claude/settings.json`) : pas d'attribution dans les commits et les PR. `post-create.sh` les fusionne avec les réglages existants.
- Node toujours disponible pour les hooks de caveman, qui ne font rien sans lui (caveman ne démarre alors qu'avec `/caveman`) : si le projet ne déclare pas de Node (Rails avec importmap, projet Bun), `post-create.sh` lance `mise use -g --pin node@lts`. La dernière LTS, épinglée, sert de version globale de secours, dans le volume de mise. La version du projet reste prioritaire là où il en déclare une.
- rtk (feature `jsburckhardt/devcontainer-features/rtk`) branché sur Claude Code : `post-create.sh` lance `RTK_TELEMETRY_DISABLED=1 rtk init -g --auto-patch`, qui ajoute son hook dans `tmp/claude/settings.json` sans rien demander. Télémétrie jamais acceptée, donc désactivée.
- À la création du conteneur (`post-create.sh`), dans l'ordre : dotfiles chezmoi, `gh auth login` si la machine n'a pas encore de jeton valide, proposition d'un fichier de version pour un nouveau projet, `mise install`, Node de secours si le projet n'en déclare pas, la gem `ruby-lsp` pour Rails, réglages de Claude Code, `rtk init`, plugins Claude Code (marketplace `codecloud-sh/claude-plugins` et caveman), Playwright pour Astro, puis installation du projet.

## Ruby on Rails

- Services compose selon les réponses : Selenium, PostgreSQL ou MySQL, Redis. Leurs ports ne sont pas exposés : l'app les joint par le réseau compose.
- Accès au Docker/Podman de l'hôte depuis le conteneur (feature docker-outside-of-docker).
- Nouveau projet (ni `.ruby-version` ni `Gemfile`) : `.ruby-version` proposé avec la dernière version stable de Ruby.
- Plugins Claude Code `stack-rails`, qui tire `codecloud-core` et `superpowers-ruby`, et `ruby-lsp` du marketplace officiel.
- Plugins des marketplaces `codecloud` et `superpowers-ruby` toujours à jour, sans intervention : à chaque démarrage du conteneur (`post-start.sh`), `codecloud-core`, `stack-rails` et `superpowers-ruby` sont mis à jour avant l'ouverture de toute session ; pendant une session, l'auto-update des marketplaces (`autoUpdate` dans `.claude/settings.json`, posé par `post-create.sh`) télécharge les nouvelles versions en arrière-plan, chargées à la session suivante ou avec `/reload-plugins`. Les plugins codecloud n'ont pas de numéro de version : chaque commit sur `main` est une mise à jour. superpowers-ruby fixe le sien : une mise à jour arrive quand son auteur publie une nouvelle version.
- Gem `ruby-lsp` dans le Ruby de mise : c'est le serveur que lance le plugin `ruby-lsp` de Claude Code (l'extension VS Code gère sa propre copie).
- Installation du projet : `bin/setup --skip-server` s'il existe.
- Ports redirigés : 3000 (Rails) et 8592 (serveur de dev Herb).
- Extensions VS Code pour Rails : Ruby LSP, Herb, Tailwind, i18n, GitLens, etc.

## Astro

- Un seul conteneur, sans compose.
- Nouveau projet (ni `.node-version`, `.nvmrc`, `.bun-version` ni `package.json`) : `.node-version` proposé avec la dernière version LTS de Node.
- Plugin Claude Code `stack-astro`, qui tire `codecloud-core` et déclare le serveur MCP de la doc Astro : rien à ajouter avec `claude mcp add`.
- Plugins du marketplace `codecloud` toujours à jour, sans intervention : à chaque démarrage du conteneur (`post-start.sh`), `codecloud-core` et `stack-astro` sont mis à jour avant l'ouverture de toute session ; pendant une session, l'auto-update du marketplace (`autoUpdate` dans `.claude/settings.json`, posé par `post-create.sh`) télécharge les nouvelles versions en arrière-plan, chargées à la session suivante ou avec `/reload-plugins`.
- Playwright (paquet npm global) et Chromium avec ses bibliothèques système, pour les captures d'écran des PR et des maquettes (skill `stack-astro:captures`), hors du projet. Le paquet est dans le volume de mise et Chromium dans le volume `<slug>_playwright-browsers` : une reconstruction ne retélécharge pas Chromium, seules ses bibliothèques système sont réinstallées. Une nouvelle version de Playwright télécharge sa version de Chromium et supprime l'ancienne.
- Installation du projet selon le fichier de verrouillage : `bun install` (`bun.lock`), `npm ci` (`package-lock.json`) ou `npm install`.
- `tmp/.gitignore` créé s'il manque : le `.gitignore` d'Astro ne couvre pas `tmp/`, qui contient la configuration de Claude Code, jetons compris.
- Port redirigé : 4321 (serveur de dev Astro).
- Extensions VS Code : Astro, Prettier (formatage à l'enregistrement), Tailwind, GitLens, etc.

## Pré-requis côté hôte

- `GIT_EMAIL` défini dans l'environnement de l'hôte : il est passé au conteneur (`${localEnv:GIT_EMAIL}`) et lu par les dotfiles. Sinon, chezmoi demande l'email à la création du conteneur.
- Avec Podman rootless : `userns = "keep-id"` dans `containers.conf` et le wrapper `getenforce` de mes dotfiles. Le template lui-même ne contient rien de propre à Podman.
- Rails avec PostgreSQL : `config/database.yml` doit lire `DB_HOST`, `DB_USER` et `DB_PASSWORD`. Avec MySQL : `DB_HOST`.

## Faire évoluer le template

Les fichiers de chaque stack sont dans son dossier (`rails/`, `astro/`), les questions dans `copier.yml`. Les questions propres à une stack ont une condition `when` sur `stack` : `database`, `redis` et `system_tests` ne sont posées que pour Rails.

Pour ajouter une stack : créer son dossier avec un `.devcontainer/` et le fichier `{{ _copier_conf.answers_file }}.jinja`, l'ajouter aux choix de `stack` et conditionner ses questions propres.

Après une modification, taguer une nouvelle version : `copier update` se base sur les tags Git.

```bash
git tag v0.2.0 && git push --tags
```
