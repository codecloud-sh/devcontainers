# devcontainers

Template [Copier](https://copier.readthedocs.io/) pour générer le dossier `.devcontainer` de mes projets.

Stacks disponibles : **Ruby on Rails** et **Astro** (une par dossier, choisie par la première question).

## Utilisation

Dans le dossier du projet :

```bash
uvx copier copy gh:nicolascochin/devcontainers .
```

Copier pose quelques questions, puis écrit `.devcontainer/`. Les réponses sont enregistrées dans `.devcontainer/.copier-answers.yml`, à commiter avec le reste.

Pour récupérer plus tard les évolutions du template (projet sous Git, sans modifications en cours) :

```bash
uvx copier update -a .devcontainer/.copier-answers.yml
```

Copier réapplique les changements du template en gardant les modifications faites dans le projet. En cas de conflit, il laisse des marqueurs à résoudre comme pour un merge.

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
- Configuration de Claude Code et de `gh` dans le `tmp/` du projet : elle survit à une reconstruction du conteneur.
- Réglages utilisateur de Claude Code (`tmp/claude/settings.json`) : pas d'attribution dans les commits et les PR. `post-create.sh` les fusionne avec les réglages existants.
- À la création du conteneur (`post-create.sh`), dans l'ordre : dotfiles chezmoi, `gh auth login`, proposition d'un fichier de version pour un nouveau projet, `mise install`, réglages de Claude Code, plugins Claude Code (marketplace `codecloud-sh/claude-plugins` et caveman), puis installation du projet.

## Ruby on Rails

- Services compose selon les réponses : Selenium, PostgreSQL ou MySQL, Redis. Leurs ports ne sont pas exposés : l'app les joint par le réseau compose.
- Accès au Docker/Podman de l'hôte depuis le conteneur (feature docker-outside-of-docker).
- Nouveau projet (ni `.ruby-version` ni `Gemfile`) : `.ruby-version` proposé avec la dernière version stable de Ruby.
- Plugin Claude Code `stack-rails`, qui tire `codecloud-core` et `superpowers-ruby`.
- Installation du projet : `bin/setup --skip-server` s'il existe.
- Ports redirigés : 3000 (Rails) et 8592 (serveur de dev Herb).
- Extensions VS Code pour Rails : Ruby LSP, Herb, Tailwind, i18n, GitLens, etc.

## Astro

- Un seul conteneur, sans compose.
- Nouveau projet (ni `.node-version`, `.nvmrc`, `.bun-version` ni `package.json`) : `.node-version` proposé avec la dernière version LTS de Node.
- Plugin Claude Code `codecloud-core` et serveur MCP de la doc Astro.
- Installation du projet selon le fichier de verrouillage : `bun install` (`bun.lock`), `npm ci` (`package-lock.json`) ou `npm install`.
- `tmp/.gitignore` créé s'il manque : le `.gitignore` d'Astro ne couvre pas `tmp/`, qui contient les jetons de `gh` et de Claude Code.
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
