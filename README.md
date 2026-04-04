# heroku-buildpack-libreoffice

A Heroku buildpack that installs LibreOffice for document conversion (e.g. DOCX to PDF) on the heroku-24 stack.

This is a single buildpack that replaces the need for both [heroku-buildpack-apt](https://github.com/heroku/heroku-buildpack-apt) and a separate LibreOffice buildpack. It installs LibreOffice via AppImage and its required X11 system dependencies via apt, all in one step.

## Why?

The standard apt buildpack approach (`libreoffice-writer` or `libreoffice-writer-nogui`) is broken on heroku-24 due to missing shared libraries and broken UNO configuration paths. The AppImage approach works reliably.

This buildpack also:

- **Pins a specific LibreOffice version** rather than pulling "latest"
- **Verifies downloads with SHA-256** to prevent dependency chain abuse
- **Produces clean build logs** instead of dumping apt-get output

## Usage

Add the buildpack to your Heroku app, before your language buildpack (e.g. Ruby, Node):

```bash
heroku buildpacks:add --index 1 https://github.com/velocity-labs/heroku-buildpack-libreoffice -a your-app
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
LIBREOFFICE_URL="https://appimages.libreitalia.org/LibreOffice-${LIBREOFFICE_VERSION}.basic-x86_64.AppImage"
LIBREOFFICE_SHA256="9b9d436e..."
```

To get the SHA-256 for a new version:

```bash
curl -sL "https://appimages.libreitalia.org/LibreOffice-VERSION.basic-x86_64.AppImage" -o /tmp/LibreOffice.AppImage
shasum -a 256 /tmp/LibreOffice.AppImage
```

After updating, purge the Heroku build cache so the new version is downloaded:

```bash
heroku repo:purge_cache -a your-app
```

## How It Works

1. Downloads the LibreOffice AppImage from appimages.libreitalia.org
2. Verifies the SHA-256 checksum (fails the build on mismatch)
3. Extracts the AppImage to `vendor/libreoffice/`
4. Installs X11 shared libraries (libsm6, libice6, etc.) via apt
5. Writes a `.profile.d` script to configure PATH and LD_LIBRARY_PATH at runtime

The AppImage and extracted files are cached between builds. Changing the version in `bin/compile` invalidates the cache automatically.

## License

MIT
