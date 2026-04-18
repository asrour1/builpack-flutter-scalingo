# Scalingo Buildpack for Flutter Web 🚀

Deploy **Flutter Web** applications on [Scalingo](https://scalingo.com) with zero configuration headaches.

This buildpack downloads the Flutter SDK, compiles your project to static web files, and pairs with the [Scalingo Nginx buildpack](https://github.com/Scalingo/nginx-buildpack) to serve them — producing a **lightweight slug** with no SDK bloat in production.

Tested and working on Scalingo's `scalingo-22` stack (Ubuntu 22.04).

---

## Quick Start

### 1. Add three files to your Flutter project root

**`.buildpacks`**

```
https://github.com/asrour1/builpack-flutter-scalingo.git
https://github.com/Scalingo/nginx-buildpack.git
```

**`nginx.conf`**

```nginx
root /app/build/web;

location / {
    try_files $uri $uri/ /index.html =404;
}

location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot|map)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

location = /index.html {
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header Pragma "no-cache";
    expires 0;
}
```

**`Procfile`**

```
web: bin/run
```

> `bin/run` is generated automatically by the Scalingo Nginx buildpack.

### 2. Create and deploy on Scalingo

```bash
# Create the app
scalingo create my-flutter-app

# Set your Flutter version (required — see "Choosing a version" below)
scalingo -a my-flutter-app env-set FLUTTER_VERSION=3.41.7

# Add the Scalingo git remote
git remote add scalingo git@ssh.osc-fr1.scalingo.io:my-flutter-app.git

# Deploy
git add .buildpacks nginx.conf Procfile
git commit -m "Add Scalingo deployment config"
git push scalingo main
```

That's it. Your Flutter Web app is live.

---

## Choosing the right Flutter version

**This is the most important step.** Your `pubspec.yaml` declares a minimum Dart SDK version. Each Flutter release bundles a specific Dart SDK. If they don't match, `flutter pub get` will fail.

Check your `pubspec.yaml`:

```yaml
environment:
  sdk: ^3.10.1  # ← This is the Dart SDK constraint
```

Then find a Flutter version that includes a compatible Dart SDK:

| Dart SDK needed | Flutter version to use |
|----------------|----------------------|
| `^3.5.0`       | `3.24.0`             |
| `^3.6.0`       | `3.27.0`             |
| `^3.10.0`      | `3.38.0`             |
| `^3.10.1`      | `3.41.7`             |

Full list: [Flutter SDK releases](https://docs.flutter.dev/release/archive)

```bash
scalingo -a my-flutter-app env-set FLUTTER_VERSION=3.41.7
```

> **Tip:** If the build fails with `version solving failed`, the error message usually suggests the correct Flutter version. Use it.

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `FLUTTER_VERSION` | `3.24.0` | Flutter SDK version to download. **Always set this explicitly.** |
| `FLUTTER_CHANNEL` | `stable` | Flutter channel (`stable`, `beta`, `dev`) |
| `FLUTTER_BUILD_ARGS` | `--release` | Arguments passed to `flutter build web` |
| `FLUTTER_CLEANUP` | `true` | Remove source files and platform dirs after build to reduce slug size |
| `FLUTTER_SOURCE_DIR` | *(root)* | Subdirectory containing the Flutter project |
| `FLUTTER_DEPLOY_DIR` | `build/web` | Output directory for compiled files |

### Examples

```bash
# Inject environment variables at build time
scalingo -a my-app env-set FLUTTER_BUILD_ARGS="--release --dart-define=API_URL=https://api.example.com"

# Flutter project is in a subdirectory
scalingo -a my-app env-set FLUTTER_SOURCE_DIR=frontend

# Deploy compiled files to a custom directory
scalingo -a my-app env-set FLUTTER_DEPLOY_DIR=public
```

---

## Project Structure

### Simple project (Flutter at root)

```
my-app/
├── .buildpacks          ← points to Flutter + Nginx buildpacks
├── nginx.conf           ← Nginx config for SPA routing
├── Procfile             ← web: bin/run
├── pubspec.yaml
├── lib/
│   └── main.dart
└── web/
    └── index.html
```

### Monorepo (Flutter in a subdirectory)

```
my-project/
├── .buildpacks
├── nginx.conf
├── Procfile
├── frontend/            ← set FLUTTER_SOURCE_DIR=frontend
│   ├── pubspec.yaml
│   ├── lib/
│   └── web/
└── backend/
    └── ...
```

For a monorepo setup, update your `nginx.conf` root if `FLUTTER_DEPLOY_DIR` changes:

```nginx
root /app/public;  # if FLUTTER_DEPLOY_DIR=public
```

---

## How It Works

The buildpack follows the standard [Scalingo buildpack API](https://doc.scalingo.com/platform/deployment/buildpacks/custom):

1. **`bin/detect`** — Detects a Flutter project by checking for `pubspec.yaml`
2. **`bin/compile`** — The main build script:
   - Downloads the Flutter SDK archive (cached between deploys)
   - Runs `flutter pub get` to install dependencies
   - Runs `flutter build web --release` to compile
   - Removes source code, platform dirs, and build caches to minimize slug size
3. **`bin/release`** — Provides the default process type

The Scalingo Nginx buildpack then serves the compiled static files from `build/web/`.

### Build cache

The Flutter SDK and pub dependencies are cached between deploys. A version change triggers a fresh SDK download; otherwise, rebuilds reuse the cached SDK for faster deploys.

---

## Troubleshooting

### `version solving failed`

Your Flutter version doesn't include the Dart SDK your project needs. Read the error message — it usually tells you exactly which Flutter version to use:

```
You can try the following suggestion to make the pubspec resolve:
* Try using the Flutter SDK version: 3.41.7.
```

Then:
```bash
scalingo -a my-app env-set FLUTTER_VERSION=3.41.7
git commit --allow-empty -m "rebuild" && git push scalingo main
```

### `getcwd: cannot access parent directories`

This was fixed in version 1.1.0 of this buildpack. Make sure your `.buildpacks` file points to the latest version.

### 404 errors on routes

Flutter Web uses client-side routing. Your `nginx.conf` **must** include the fallback to `index.html`:

```nginx
location / {
    try_files $uri $uri/ /index.html =404;
}
```

### Slug too large

Make sure `FLUTTER_CLEANUP=true` (the default). The buildpack removes `android/`, `ios/`, `linux/`, `macos/`, `windows/`, `test/`, `lib/`, and `.dart_tool/` from the slug. Only the compiled web files are kept.

If it's still too large, check your `web/` directory for unoptimized assets.

### Force a clean rebuild

```bash
git commit --allow-empty -m "clean rebuild"
git push scalingo main
```

To clear the SDK cache entirely, you can use the Scalingo dashboard to purge the build cache.

---

## Compatibility

| Component | Supported |
|---|---|
| **Scalingo stack** | `scalingo-22` (Ubuntu 22.04) |
| **Flutter SDK** | 3.x (tested with 3.24.0, 3.38.0, 3.41.7) |
| **Dart SDK** | Included with Flutter SDK |

---

## Using with other server technologies

If you don't want to use Nginx and prefer Node.js, Python, or another server, you can use this buildpack alone:

```bash
scalingo -a my-app env-set BUILDPACK_URL=https://github.com/asrour1/builpack-flutter-scalingo.git
```

Set `FLUTTER_DEPLOY_DIR` to the directory your server expects to serve static files from, and write your own `Procfile` to start your server.

---

## Contributing

Issues and PRs welcome. If you run into a problem, include your build logs — they contain all the info needed to debug.

---

## License

MIT — see [LICENSE](LICENSE)

## Credits

- Inspired by [diezep/heroku-buildpack-flutter](https://github.com/diezep/heroku-buildpack-flutter) and [EE/heroku-buildpack-flutter-light](https://github.com/EE/heroku-buildpack-flutter-light)
- Built for [Scalingo](https://scalingo.com) by [Agence Digitale Iroko](https://iroko.io)