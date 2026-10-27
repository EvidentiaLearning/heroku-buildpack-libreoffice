# heroku-buildpack-libreoffice

A Heroku buildpack that installs LibreOffice for document conversion (e.g. DOCX to PDF) on the heroku-24 stack.

This is a single buildpack that replaces the need for both [heroku-buildpack-apt](https://github.com/heroku/heroku-buildpack-apt) and a separate LibreOffice buildpack. It installs LibreOffice via AppImage and its required X11 system dependencies via apt, all in one step.

**This fork** of [velocity-labs/heroku-buildpack-libreoffice](https://github.com/velocity-labs/heroku-buildpack-libreoffice) downloads LibreOffice from a release asset on this repo instead of appimages.libreitalia.org, so builds do not fail when that host is down. The asset is the AppImage's extracted `usr/` and `opt/` trees as a tarball. It works on heroku-24 and heroku-26.

## Why?

The standard apt buildpack approach (`libreoffice-writer` or `libreoffice-writer-nogui`) is broken on heroku-24 due to missing shared libraries and broken UNO configuration paths. The AppImage approach works reliably.

This buildpack also:

- **Pins a specific LibreOffice version** rather than pulling "latest"
- **Verifies downloads with SHA-256** to prevent dependency chain abuse
- **Produces clean build logs** instead of dumping apt-get output

## Usage

Add the buildpack to your Heroku app, before your language buildpack (e.g. Ruby, Node):

```bash
heroku buildpacks:add --index 1 https://github.com/EvidentiaLearning/heroku-buildpack-libreoffice -a your-app
```

No `Aptfile` or configuration is needed. Just deploy.

At runtime, `soffice` is on your PATH:

```bash
soffice --headless --convert-to pdf document.docx
```

## Build Output

Cached builds:

```
-----> Using cached LibreOffice 26.2.2.2
       Installing to vendor/libreoffice
-----> Installing 9 system dependencies
       * libsm6 (2:1.2.3-1build3)
       * libice6 (2:1.0.10-1build3)
       ...
-----> LibreOffice buildpack complete (libreoffice26.2)
```

## Updating LibreOffice

Edit the three variables at the top of `bin/compile`:

```bash
LIBREOFFICE_VERSION="26.2.2.2"
LIBREOFFICE_URL="https://github.com/EvidentiaLearning/heroku-buildpack-libreoffice/releases/download/libreoffice-${LIBREOFFICE_VERSION}/libreoffice-${LIBREOFFICE_VERSION}-x86_64.tar.gz"
LIBREOFFICE_SHA256="87c43e74..."
```

To publish a new version, extract the AppImage on Linux, tar its `usr/` and `opt/` trees, and attach the tarball to a release tagged `libreoffice-VERSION`:

```bash
curl -fL "https://appimages.libreitalia.org/LibreOffice-VERSION.basic-x86_64.AppImage" -o LibreOffice.AppImage
chmod +x LibreOffice.AppImage && ./LibreOffice.AppImage --appimage-extract
tar -C squashfs-root -czf libreoffice-VERSION-x86_64.tar.gz usr opt
shasum -a 256 libreoffice-VERSION-x86_64.tar.gz
gh release create libreoffice-VERSION libreoffice-VERSION-x86_64.tar.gz
```

After updating, purge the Heroku build cache so the new version is downloaded:

```bash
heroku repo:purge_cache -a your-app
```

## How It Works

1. Downloads the LibreOffice tarball from this repo's releases
2. Verifies the SHA-256 checksum (fails the build on mismatch)
3. Extracts it to `vendor/libreoffice/`
4. Installs X11 shared libraries (libsm6, libice6, etc.) via apt
5. Writes a `.profile.d` script to configure PATH and LD_LIBRARY_PATH at runtime

The extracted files are cached between builds. Changing the version in `bin/compile` invalidates the cache automatically.

## License

MIT
