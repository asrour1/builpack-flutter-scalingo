# Scalingo Buildpack Flutter Web 🚀

Buildpack pour compiler et déployer des applications **Flutter Web** sur [Scalingo](https://scalingo.com).

Inspiré de [diezep/heroku-buildpack-flutter](https://github.com/diezep/heroku-buildpack-flutter) et adapté aux spécificités de la plateforme Scalingo (stack Ubuntu 22.04, buildpack Nginx, multi-buildpacks).

## Fonctionnalités

- ✅ Téléchargement et cache automatique du Flutter SDK
- ✅ Cache des dépendances pub entre les builds
- ✅ Version de Flutter configurable
- ✅ Nettoyage automatique (pas de SDK en production, slug léger)
- ✅ Support des projets Flutter dans un sous-répertoire
- ✅ Compatible avec le multi-buildpack Scalingo (Flutter + Nginx)

## Installation

### Méthode recommandée : Flutter + Nginx (multi-buildpack)

Cette méthode utilise le buildpack Flutter pour compiler l'app, puis le buildpack Nginx de Scalingo pour servir les fichiers statiques.

**1. Créez un fichier `.buildpacks` à la racine de votre projet :**

```
https://github.com/<votre-user>/scalingo-buildpack-flutter.git
https://github.com/Scalingo/nginx-buildpack.git
```

**2. Créez un fichier `nginx.conf` à la racine :**

```nginx
root /app/build/web;

location / {
    try_files $uri $uri/ /index.html =404;
}

# Cache des assets Flutter
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

**3. Créez un `Procfile` :**

```
web: bin/run
```

> `bin/run` est généré automatiquement par le buildpack Nginx de Scalingo.

**4. Configurez l'app Scalingo :**

```bash
scalingo create mon-app-flutter
git push scalingo main
```

### Méthode alternative : buildpack seul

Si vous utilisez un autre serveur (Node.js, Python, etc.) pour servir les fichiers :

```bash
scalingo env-set BUILDPACK_URL=https://github.com/<votre-user>/scalingo-buildpack-flutter.git
```

Vous devrez alors configurer votre propre serveur pour servir le dossier `build/web`.

## Variables d'environnement

| Variable | Défaut | Description |
|---|---|---|
| `FLUTTER_VERSION` | *Dernière version stable* | Version de Flutter à utiliser (ex: `3.24.0`) |
| `FLUTTER_CHANNEL` | `stable` | Canal Flutter (`stable`, `beta`, `dev`) |
| `FLUTTER_BUILD_ARGS` | `--release` | Arguments passés à `flutter build web` |
| `FLUTTER_CLEANUP` | `true` | Supprime les artefacts de build pour réduire le slug |
| `FLUTTER_SOURCE_DIR` | *(racine)* | Sous-répertoire contenant le projet Flutter |
| `FLUTTER_DEPLOY_DIR` | `build/web` | Répertoire de sortie des fichiers compilés |

### Exemples

```bash
# Fixer une version Flutter spécifique (recommandé)
scalingo env-set FLUTTER_VERSION=3.24.0

# Build avec des options supplémentaires
scalingo env-set FLUTTER_BUILD_ARGS="--release --dart-define=API_URL=https://api.example.com"

# Projet Flutter dans un sous-dossier
scalingo env-set FLUTTER_SOURCE_DIR=frontend

# Déployer les fichiers dans un dossier personnalisé
scalingo env-set FLUTTER_DEPLOY_DIR=public
```

## Structure du projet

### Projet simple (Flutter à la racine)

```
mon-projet/
├── .buildpacks
├── nginx.conf
├── Procfile
├── pubspec.yaml
├── lib/
│   └── main.dart
└── web/
    └── index.html
```

### Projet avec Flutter dans un sous-dossier

```
mon-projet/
├── .buildpacks
├── nginx.conf
├── Procfile
├── frontend/          ← FLUTTER_SOURCE_DIR=frontend
│   ├── pubspec.yaml
│   ├── lib/
│   └── web/
└── backend/
    └── ...
```

## Dépannage

### Le build échoue au téléchargement du SDK

Vérifiez que la version Flutter spécifiée existe. Consultez les [releases Flutter](https://docs.flutter.dev/release/archive) pour les versions disponibles.

```bash
# Vérifier la version actuelle
scalingo env | grep FLUTTER
```

### Slug trop volumineux

Assurez-vous que `FLUTTER_CLEANUP=true` (c'est le défaut). Si le slug reste trop gros, vérifiez que vous n'avez pas de fichiers lourds dans votre repo (assets non optimisés, etc.).

### Erreur 404 sur les routes

Flutter Web utilise le routing côté client. Votre configuration Nginx doit inclure le fallback vers `index.html` :

```nginx
location / {
    try_files $uri $uri/ /index.html =404;
}
```

### Cache de build obsolète

Pour forcer un build propre :

```bash
# Faire un commit vide et redéployer
git commit --allow-empty -m "clean rebuild"
git push scalingo main
```

## Compatibilité

- **Stack Scalingo** : scalingo-22 (Ubuntu 22.04)
- **Flutter** : 3.x+ (testé avec 3.19+)
- **Dart** : Inclus avec le Flutter SDK

## Comment ça marche

1. **`bin/detect`** : Détecte un projet Flutter par la présence de `pubspec.yaml`
2. **`bin/compile`** :
   - Télécharge le Flutter SDK (avec cache entre les builds)
   - Installe les dépendances (`flutter pub get`)
   - Compile l'app web (`flutter build web`)
   - Nettoie les artefacts de build
3. **`bin/release`** : Fournit la commande de démarrage par défaut

Le buildpack Nginx de Scalingo prend ensuite le relais pour servir les fichiers statiques compilés.

## Licence

MIT

## Crédits

- Inspiré par [diezep/heroku-buildpack-flutter](https://github.com/diezep/heroku-buildpack-flutter)
- Architecture inspirée de [EE/heroku-buildpack-flutter-light](https://github.com/EE/heroku-buildpack-flutter-light)
- Adapté pour [Scalingo](https://scalingo.com) par [Agence Digitale Iroko](https://iroko.digital)
