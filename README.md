# devcontainers

Template [Copier](https://copier.readthedocs.io/) pour générer le dossier `.devcontainer` de mes projets.

Stacks disponibles : **Ruby on Rails** (une par dossier, choisie par la première question).

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

| Question | Défaut | Effet |
| --- | --- | --- |
| `stack` | Ruby on Rails | Dossier du template utilisé (`rails/`) |
| `project_name` | nom du dossier | Nom du devcontainer dans VS Code |
| `project_slug` | dérivé du nom | Nom du projet compose, hostname du conteneur et dossier de travail (`/workspaces/<slug>`) |
| `username` | `dev` | Utilisateur créé dans le conteneur |
| `database` | SQLite | Feature client, service compose et `DB_HOST` pour PostgreSQL ou MySQL (plus `DB_USER` et `DB_PASSWORD` pour PostgreSQL) |
| `redis` | non | Service Valkey et `REDIS_URL` |
| `system_tests` | oui | Service Selenium (jusqu'à 4 navigateurs en parallèle, 2 Go de `/dev/shm`), `SELENIUM_HOST` et `CAPYBARA_SERVER_PORT` |

Pour ne pas retaper les mêmes réponses, Copier lit des valeurs par défaut dans `~/.config/copier/settings.yml` sous Linux (ou dans le fichier indiqué par `COPIER_SETTINGS_PATH`) :

```yaml
defaults:
  username: nico
```

## Contenu du devcontainer

- Image Debian slim, utilisateur non root avec sudo et zsh.
- Accès au Docker/Podman de l'hôte depuis le conteneur (feature docker-outside-of-docker).
- mise installé dans l'image par la feature `devcontainers-extra/features/mise` (`/usr/local/bin/mise`). Il installe ce que le projet déclare : Ruby (`.ruby-version`), Node (`.node-version`) ou Bun (`.bun-version`), et rien de plus pour un projet sans JavaScript. Avec Bun, le `.bun-version` est à créer soi-même : `bun --version > .bun-version`.
- Ruby, Node ou Bun et les gems (le dossier de mise) dans un volume nommé, `<slug>_mise-data` : une reconstruction du conteneur ne les réinstalle pas. Pour repartir de zéro, par exemple après une montée de version de Debian dans l'image : `podman volume rm <slug>_mise-data` (ou `docker volume rm`).
- À la création du conteneur (`post-create.sh`) : dotfiles chezmoi, `gh auth login`, `.ruby-version` proposé avec la dernière version stable de Ruby pour un nouveau projet (ni `.ruby-version` ni `Gemfile`), `mise install`, la gem `ruby-lsp`, plugins Claude Code (`stack-rails` du marketplace `codecloud-sh/claude-plugins`, qui tire `codecloud-core` et `superpowers-ruby`, `ruby-lsp` du marketplace officiel, et caveman), puis `bin/setup --skip-server` s'il existe.
- Plugins du marketplace `codecloud` toujours à jour, sans intervention :
  - à chaque démarrage du conteneur (`post-start.sh`), `codecloud-core` et `stack-rails` sont mis à jour avant l'ouverture de toute session ;
  - pendant une session, l'auto-update du marketplace (`autoUpdate` dans `.claude/settings.json`, posé par `post-create.sh`) télécharge les nouvelles versions en arrière-plan ; elles se chargent à la session suivante, ou tout de suite avec `/reload-plugins`.
- Gem `ruby-lsp` dans le Ruby de mise : c'est le serveur que lance le plugin `ruby-lsp` de Claude Code (l'extension VS Code gère sa propre copie).
- Configuration de Claude Code et de `gh` dans le `tmp/` du projet : elle survit à une reconstruction du conteneur.
- Réglages utilisateur de Claude Code (`tmp/claude/settings.json`) : pas d'attribution dans les commits et les PR. `post-create.sh` les fusionne avec les réglages existants.
- Extensions VS Code pour Rails : Ruby LSP, Herb, Tailwind, i18n, GitLens, etc.

## Pré-requis côté hôte

- `GIT_EMAIL` défini dans l'environnement de l'hôte : il est passé au conteneur (`${localEnv:GIT_EMAIL}`) et lu par les dotfiles. Sinon, chezmoi demande l'email à la création du conteneur.
- Avec Podman rootless : `userns = "keep-id"` dans `containers.conf` et le wrapper `getenforce` de mes dotfiles. Le template lui-même ne contient rien de propre à Podman.
- Avec PostgreSQL : `config/database.yml` doit lire `DB_HOST`, `DB_USER` et `DB_PASSWORD`. Avec MySQL : `DB_HOST`.

## Faire évoluer le template

Les fichiers de chaque stack sont dans son dossier (`rails/`), les questions dans `copier.yml`. Les questions propres à une stack ont une condition `when` sur `stack` : `database`, `redis` et `system_tests` ne sont posées que pour Rails.

Pour ajouter une stack : créer son dossier avec un `.devcontainer/` et le fichier `{{ _copier_conf.answers_file }}.jinja`, l'ajouter aux choix de `stack` et conditionner ses questions propres.

Après une modification, taguer une nouvelle version : `copier update` se base sur les tags Git.

```bash
git tag v0.2.0 && git push --tags
```
