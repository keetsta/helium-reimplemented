<div align="center">
    <img src="resources/branding/app_icon/raw.png"
        title="Helium Reimplemented" alt="Helium Reimplemented logo" width="120" />
    <h1>Helium Reimplemented</h1>
    <p>
        A personal fork of <a href="https://github.com/imputnet/helium">Helium</a>
        that reimplements features cut from upstream and adds self-hosted
        cross-device sync.
    </p>
</div>

## What this is

This is the **core** repo: a cross-platform patch set applied on top of
Chromium. Platform packaging and builds live in the thin platform repos
(`helium-windows`, `helium-macos`, `helium-linux`), which pull this core,
download the Chromium source, and apply these patches. Builds target all three
desktop platforms.

The Chromium version is pinned in
[`chromium_version.txt`](chromium_version.txt).

## Features

On top of everything inherited from Helium, this fork adds:

- **Zoom bubble** — the page-zoom popup (percentage + reset button) on
  <kbd>Ctrl</kbd> <kbd>+</kbd>/<kbd>−</kbd> and <kbd>Ctrl</kbd>+scroll, which
  upstream removed.
- **Self-hosted sync** — cross-device sync of the extension list and bookmarks
  through your own
  [services](https://github.com/keetsta/helium-reimplemented-services) instance.
  Bookmarks use a tombstone model so deletions propagate safely; bookmarks
  removed on another device land in a recoverable trash bin. Extensions are
  listed for you to install, never installed automatically.
- **Send tab to device** — push an open tab to another of your devices through
  the same self-hosted instance.
- **Import from Helium** — onboarding can import history, bookmarks, and the
  extension list from a stock Helium install.

The browser identifies itself as "Helium Reimplemented", with its own profile
directory and install identifiers, so it can run alongside stock Helium.

## Building

Builds are driven from the platform repos, not from here. Pick a platform and
follow its build script:

- [helium-windows](https://github.com/keetsta/helium-windows)
- [helium-macos](https://github.com/imputnet/helium-macos)
- [helium-linux](https://github.com/imputnet/helium-linux)

Patches are plain diff text applied with `quilt` (or by the platform build
script). They live under [`patches/`](patches/), sorted by vendor.

## License

Code, patches, and modified portions unique to this project (and to upstream
Helium) are licensed under GPL-3.0. See [LICENSE](LICENSE).

Content imported from other projects retains its original license — for
example, unmodified code from
[ungoogled-chromium](https://github.com/ungoogled-software/ungoogled-chromium)
remains under its
[BSD 3-Clause license](LICENSE.ungoogled_chromium).

## Credits

Built on [Helium](https://github.com/imputnet/helium),
[ungoogled-chromium](https://github.com/ungoogled-software/ungoogled-chromium),
and [the Chromium project](https://www.chromium.org/). Patches are also drawn
from [Brave](https://github.com/brave/brave-core),
[Inox](https://github.com/gcarq/inox-patchset),
[Bromite](https://github.com/bromite/bromite),
[Iridium](https://iridiumbrowser.de/), and
[Debian](https://tracker.debian.org/pkg/chromium-browser).
