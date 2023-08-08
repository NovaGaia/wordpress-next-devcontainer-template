# Création d'un projet avec WordPress et NextJS dans un devcontainer

> la base de données est gérée hors de ce devcontainer, créér-la en premier.

## Récupération de WordPress

Aller à https://wordpress.org/download/ et télécharger la dernière version de WordPress.

## Installation de WordPress

1. Créer un dossier `packages/back-wp` à la racine du projet.
2. Dézipper le dossier WordPress dans `packages/back-wp`.
3. Créer un dossier vide `packages/back-wp/wp-content/uploads` et y créer un fichier `.gitkeep`.
4. Dupliquer le fichier`package/back-wp/wp-config-sample.php` et nommer-le `wp-config.php`. Modifier le fichier `packages/back-wp/wp-config.php` pour y ajouter la possibilité de recevoir sa configuration via Docker.

## A. Ajouter

```php
// a helper function to lookup "env_FILE", "env", then fallback
if (!function_exists('getenv_docker')) {
	// https://github.com/docker-library/wordpress/issues/588 (WP-CLI will load this file 2x)
	function getenv_docker($env, $default) {
		if ($fileEnv = getenv($env . '_FILE')) {
			return rtrim(file_get_contents($fileEnv), "\r\n");
		}
		else if (($val = getenv($env)) !== false) {
			return $val;
		}
		else {
			return $default;
		}
	}
}
```

## B. Modifier les lignes suivantes:

```php
define( 'DB_HOST', 'localhost' );
// en
define( 'DB_HOST', getenv_docker('WORDPRESS_DB_HOST', 'db') );
```

## Initialisation du mono-repo et autres fichiers de configuration

1. Initialiser git `git init`.
1. Créer un fichier `package.json` à la racine du projet en lançant `npm init --yes`.
   > Le fichier `package.json` à la racine du projet est le fichier de configuration du mono-repo.
   > Ajouter ces devDependencies:

```json
"devDependencies": {
  "prettier": "^2.8.8",
  "prettier-plugin-tailwindcss": "^0.4.1"
}
```

3. Créer un fichier `.gitignore` à la racine du projet avec le contenu suivant:
   > Ce fichier permet de ne pas versionner les dossiers `node_modules` et `packages/back-wp/wp-content/uploads`.

```ini
# .gitignore
.DS_Store
node_modules
packages/back-wp/wp-content/uploads/*
.pnpm-store
.devcontainer/wp.env
```

4. Créer un fichier `.gitattributes` à la racine du projet avec le contenu suivant:
   > Ce fichier permet de gérer les sauts de ligne entre Windows et Linux.

```ini
# .gitattributes
# Set the default behavior, in case people don't have core.autocrlf set.
* text=auto
```

5. Créer un fichier `.npmrc` à la racine du projet avec le contenu suivant:
   > Ce fichier permet de gérer la gestion des packages en utilisant pnpm.

```ini
# .npmrc
# Use pnpm
shamefully-hoist=true
strict-peer-dependencies=false
audit=false
```

6. Créer un fichier `pnpm-workspace.yaml` à la racine du projet avec le contenu suivant:
   > Ce fichier permet de gérer les dépendances entre les packages.

```yml
# pnpm-workspace.yaml
packages:
  - "packages/*"
```

7. Créer un fichier `.nvmrc` à la racine du projet avec le contenu suivant:
   > Ce fichier permet de gérer la version de NodeJS utilisée dans le projet. La version doit être la même que celle utilisée dans le .devcontainer.

```
18.16.1
```

8. Créer un fichier `.devcontainer/wp.env` à la racine du projet avec le contenu suivant:
   > Ce fichier permet de gérer les variables d'environnement de WordPress.

```ini
# .devcontainer/wp.env
WORDPRESS_DB_HOST=db
WORDPRESS_DB_USER=wordpress
WORDPRESS_DB_PASSWORD=wordpress
WORDPRESS_DB_NAME=wordpress
```

9. Créer un fichier `prettier.config.js` à la racine du projet avec le contenu suivant:
   > Ce fichier permet de gérer la configuration de Prettier.

```js
// prettier.config.js
module.exports = {
  pluginSearchDirs: ["./node_modules", "node_modules"],
  plugins: ["prettier-plugin-tailwindcss"],
};
```

## Initialisation du devcontainer

> Objectif: avoir un environnement de développement identique pour tous les développeurs.
> Son fonctionnement est basé sur Docker.
> Contenu :
>
> - NodeJS (NextJS)
> - PHP (php-fpm) aka WordPress
> - Nginx

1. Création du dossier `.devcontainer` à la racine du projet.
2. création des dossiers suivants dans `.devcontainer`:

```sh
.devcontainer/postCreateCommand
.devcontainer/postStartCommand
.devcontainer/docker/php-fpm
```

3. Création des fichiers suivants dans `.devcontainer`:

```sh
# Configuration de l'environnement de développement
.devcontainer/devcontainer.json
# Configuration de Docker pour NodeJS
.devcontainer/docker-compose.yml
# Configuration de Docker pour WordPress
.devcontainer/docker-compose.wp.yml
# Scripts de configuration
.devcontainer/postCreateCommand/_add-ssh-keys.sh
.devcontainer/postCreateCommand/_chown-folders.sh
.devcontainer/postCreateCommand/custom.sh
.devcontainer/postStartCommand/custom.sh
# Configuration de Nginx pour son container
.devcontainer/default.conf
# Configuration de PHP pour son container
.devcontainer/docker/php-fpm/Dockerfile
.devcontainer/docker/php-fpm/php.ini
```

### Contenu des fichiers

> Les fichiers sont commentés pour expliquer leur fonctionnement.
> deux élements de configuration sont importants:
>
> - Il faut un créer, au travers des fichiers un reseau commun entre les containers.
> - Il faut que les noms de containers ne soient pas identiques avec ceux d'autres projets. Pour cela, il ne faut jamais les précisier et faire en sorte que le nom du dossier du projet root (celui contenant ce projet) soit unique.

#### .devcontainer/devcontainer.json

> A adapter en fonction des besoins.

```json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the
// README at: https://github.com/devcontainers/templates/tree/main/src/javascript-node
{
  "name": "Projet (WordPress/NextJS/TailwindCSS)",
  // Or use a Dockerfile or Docker Compose file. More info: https://containers.dev/guide/dockerfile
  // "image": "mcr.microsoft.com/devcontainers/javascript-node:1-18-bullseye",
  "dockerComposeFile": ["docker-compose.yml", "docker-compose.wp.yml"],
  // la valeur de service doit être la même que le nom du service dans le docker-compose.yml
  "service": "node",
  "workspaceFolder": "/workspace",

  // Features to add to the dev container. More info: https://containers.dev/features.
  "features": {
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/rio/features/chezmoi:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },

  // Use 'forwardPorts' to make a list of ports inside the container available locally.
  "forwardPorts": [8080, 3000],
  "portsAttributes": {
    "8080": {
      "label": "WordPress",
      "onAutoForward": "notify"
    },
    "3000": {
      "label": "NextJS",
      "onAutoForward": "notify"
    }
  },

  // Use 'initializeCommand' A command string or list of command arguments to run on the host machine before the container is created.
  "initializeCommand": {},

  // Use 'postStartCommand' to run commands each time the container is successfully started..
  // https://www.kenmuse.com/blog/avoiding-dubious-ownership-in-dev-containers/
  "postStartCommand": {
    "git.config.safe.directory": "git config --global --add safe.directory ${containerWorkspaceFolder}",
    "git.config.pull.rebase": "git config pull.rebase false",
    "custom-cmd": "sh /workspace/.devcontainer/postStartCommand/custom.sh"
  },

  // Use 'postCreateCommand' to run commands after the container is created.
  "postCreateCommand": {
    "chown-folders": "sh /workspace/.devcontainer/postCreateCommand/_chown-folders.sh",
    "add-ssh-keys": "sh /workspace/.devcontainer/postCreateCommand/_add-ssh-keys.sh",
    "pnpm install": "npm install -g pnpm",
    "custom-cmd": "sh /workspace/.devcontainer/postCreateCommand/custom.sh"
  },

  // Configure tool-specific properties.
  "customizations": {
    "vscode": {
      "extensions": [
        "oderwat.indent-rainbow",
        "esbenp.prettier-vscode",
        "amatiasq.sort-imports",
        "vscode-icons-team.vscode-icons",
        "dbaeumer.vscode-eslint",
        "formulahendry.auto-rename-tag",
        "mutantdino.resourcemonitor",
        "yzhang.markdown-all-in-one",
        "adam-bender.commit-message-editor",
        "mhutchie.git-graph",
        "eamodio.gitlens",
        "felixfbecker.php-pack",
        "wordpresstoolbox.wordpress-toolbox",
        "johnbillion.vscode-wordpress-hooks",
        "GitHub.copilot",
        "bradlc.vscode-tailwindcss",
        "kamikillerto.vscode-colorize",
        "bmewburn.vscode-intelephense-client"
      ],
      "settings": {
        "dotfiles.targetPath": "~/dotfiles",
        "terminal.integrated.profiles.linux": {
          "bash": {
            "path": "bash",
            "icon": "terminal-bash"
          },
          "zsh": {
            "path": "/bin/zsh",
            "icon": "terminal-powershell"
          }
        },
        "terminal.integrated.defaultProfile.linux": "zsh",
        "editor.formatOnSave": true,
        "prettier.prettierPath": "./node_modules/prettier",
        "php.suggest.basic": false // avoids duplicate autocomplete
      }
    }
  },

  "mounts": [
    "source=project-node_modules,target=${containerWorkspaceFolder}/node_modules,type=volume",
    "source=project-pnpm-store,target=${containerWorkspaceFolder}/.pnpm-store,type=volume",
    // Persisting user profile https://code.visualstudio.com/docs/devcontainers/tips-and-tricks#_persisting-user-profile
    "source=profile,target=/root,type=volume",
    "target=/root/.vscode-server,type=volume"
  ],

  // Uncomment to connect as root instead. More info: https://aka.ms/dev-containers-non-root.
  // "remoteUser": "root"
  // "remoteUser": "vscode"
  "remoteUser": "node"
}
```

#### .devcontainer/docker-compose.yml

> A adapter en fonction des besoins.
> Attention au nom du network qui doit être unique à votre machine/équipe.

```yml
version: "3.8"

services:
  # le nom du service doit être celui présent dans le devcontainer.json
  node:
    # build:
    #   context: .
    #   dockerfile: Dockerfile
    image: mcr.microsoft.com/devcontainers/javascript-node:0-18
    # I want to use the same
    volumes:
      - ..:/workspace:cached
      # Docker
      - ~/.docker:/node/.docker
      # Docker socket to access Docker server
      - /var/run/docker.sock:/var/run/docker.sock
      # SSH directory for Linux, OSX and WSL
      # On Linux and OSX, a symlink /mnt/ssh <-> ~/.ssh is
      # created in the container. On Windows, files are copied
      # from /mnt/ssh to ~/.ssh to fix permissions.
      - ~/.ssh:/mnt/ssh
      # Shell history persistence
      - ~/.zsh_history:/node/.zsh_history
      # Git config
      - ~/.gitconfig:/node/.gitconfig

    # Overrides default command so things don't shut down after the process ends.
    command: sleep infinity
    networks:
      - synapsysdigital_node_net

networks:
  synapsysdigital_node_net:
```

#### .devcontainer/docker-compose.wp.yml

> A adapter en fonction des besoins.
> Attention au nom du network qui doit être le même que celui du docker-compose.yml du projet.

```yml
version: "3.8"
# https://github.com/docker/awesome-compose/tree/master/official-documentation-samples/wordpress/

services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ../packages/back-wp:/var/www/html
      - ./default.conf:/etc/nginx/conf.d/default.conf
    links:
      - wordpress
    networks:
      - synapsysdigital_node_net
  wordpress:
    build: docker/php-fpm
    volumes:
      - ../packages/back-wp:/var/www/html
    env_file: wp.env
    restart: always
    networks:
      - synapsysdigital_node_net

networks:
  synapsysdigital_node_net:
```

#### .devcontainer/default.conf

> A adapter en fonction des besoins.

```apacheconf
server {
    index index.php index.html;
    server_name phpfpm.local;
    error_log /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/html;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

#### .devcontainer/docker/php-fpm/Dockerfile

> A adapter en fonction des besoins.

```dockerfile
FROM php:8.2-fpm
RUN docker-php-ext-install mysqli
COPY ./php.ini /usr/local/etc/php/conf.d/app.ini
```

#### .devcontainer/docker/php-fpm/php.ini

> A adapter en fonction des besoins.

```ini
; php.ini
date.timezone = Europe/Paris

opcache.enable = 1
opcache.enable_cli = 1
opcache.memory_consumption = 128
opcache.revalidate_freq = 0
apc.enable_cli = On

upload_max_filesize = 16M
post_max_size = 16M

realpath_cache_size = 4096k
realpath_cache_ttl = 7200

display_errors = Off
display_startup_errors = Off
; output_buffer = off
output_buffering = 4096

; maximum_execution_time = 300
```

## Initialisation du projet Next.js

> A adapter en fonction des besoins.

```bash
# Création du projet
npx create-next-app@latest packages/front-nextjs --use-npm
```

## Lancez le projet dans le Devcontainer

**IL FAUT TOUJOURS TRAVAILLER DANS LE DEVCONTAINER POUR PROFITER DU SERVEUR NGINX, WORDPRESS etc.**

> Prérequis

- Docker
- VSCODE
- Extention pack vscode : `[ms-vscode-remote.vscode-remote-extensionpack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)`

**Ouvrir l'environnement dans Docker**

1. Ouvrir le projet normalement.
2. Soit faire [CTRL/CMD]+[SHIFT]+P ou cliquer dans le coin à gauche de la fenètre VSCode et faire `Dev Containers: Reopen in Container` et le laisser se lancer ⏳

**Relancer si problème**  
Faire [CTRL/CMD]+[SHIFT]+P ou cliquer dans le coin à gauche de la fenètre VSCode et faire `Dev Containers: Rebuild Without Cache and Reopen in Container` et le laisser se lancer ⏳

**Sortir du mode Devcontainer**  
Faire [CTRL/CMD]+[SHIFT]+P ou cliquer dans le coin à gauche de la fenètre VSCode et faire `Dev Containers: Reopen Folder Locally` et le laisser se fermer ⏳

## Finir l'initalisation du projet

Faire `pnpm i` ou `pnpm install`.

## Lancer le projet Next.js

Soit faire `pnpm -r dev` ou `pnpm -r run dev` à la racine du projet.

> Le `-r` permet de lancer la commande dans tous les sous-dossiers.
