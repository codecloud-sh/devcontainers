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
| `git_email` | vide | Email Git passé aux dotfiles. Vide : chezmoi le demande à la création du conteneur |
| `database` | SQLite | Feature client, service compose et `DB_HOST` pour PostgreSQL ou MySQL |
| `redis` | non | Service Valkey et `REDIS_URL` |
| `system_tests` | oui | Service Selenium, `SELENIUM_HOST` et `CAPYBARA_SERVER_PORT` |
| `kamal` | oui | Accès au Docker/Podman de l'hôte et `KAMAL_REGISTRY_PASSWORD` |
| `stripe` | non | Stripe CLI installé dans l'image |

Pour ne pas retaper les mêmes réponses, Copier lit des valeurs par défaut dans `~/.config/copier/settings.yml` sous Linux (ou dans le fichier indiqué par `COPIER_SETTINGS_PATH`) :

```yaml
defaults:
  git_email: moi@example.com
```

## Contenu du devcontainer

- Image Debian slim, utilisateur non root avec sudo et zsh.
- Ruby et Node installés par mise à partir de `.ruby-version` et `.node-version`.
- À la création du conteneur (`post-create.sh`) : dotfiles chezmoi, `gh auth login`, `mise install`, plugins Claude Code (ruby-lsp, superpowers, superpowers-ruby, caveman), puis `bin/setup --skip-server` s'il existe.
- Configuration de Claude Code, de `gh` et session Proton Pass dans le `tmp/` du projet : elles survivent à une reconstruction du conteneur.
- Extensions VS Code pour Rails : Ruby LSP, Herb, Tailwind, i18n, GitLens, etc.

## Pré-requis côté hôte

- `PROTON_PASS_ENCRYPTION_KEY` défini dans l'environnement de l'hôte (lu avec `${localEnv:...}`).
- Avec Podman rootless : `userns = "keep-id"` dans `containers.conf` et le wrapper `getenforce` de mes dotfiles. Le template lui-même ne contient rien de propre à Podman.
- Avec PostgreSQL ou MySQL : `config/database.yml` doit lire `DB_HOST`, comme le génère `rails new --devcontainer`.

## Faire évoluer le template

Les fichiers de chaque stack sont dans son dossier (`rails/`), les questions dans `copier.yml`. Les questions propres à une stack ont une condition `when` sur `stack` : `database`, `redis`, `system_tests`, `kamal` et `stripe` ne sont posées que pour Rails.

Pour ajouter une stack : créer son dossier avec un `.devcontainer/` et le fichier `{{ _copier_conf.answers_file }}.jinja`, l'ajouter aux choix de `stack` et conditionner ses questions propres.

Après une modification, taguer une nouvelle version : `copier update` se base sur les tags Git.

```bash
git tag v0.2.0 && git push --tags
```
