# Plugin Catalog — Staging Area

This directory holds the **publishable artifacts** for the MagicBoxLab
plugin-hosting site at `https://plugins.magicboxlab.ai/`. The site is a
single GitHub Pages deploy in `magicboxlab-ai/plugins`; each
product gets its own subdirectory so we can host more than just GridLens
on the same domain later.

The contents here mirror exactly what the GridLens client sees when it
hits the catalog URL configured in *Preferences → Plugins → Catalog*.

## Layout

```
published-catalog/
  CNAME                                  # plugins.magicboxlab.ai
  gridlens/
    catalog.json                         # catalog index (schema v1)
    bundles/
      dbip_country-1.0.0-macosx_11_0_arm64.glplugin
      dbip_country-1.0.0-macosx_11_0_x86_64.glplugin
      dbip_country-1.0.0-manylinux2014_aarch64.glplugin
      dbip_country-1.0.0-manylinux2014_x86_64.glplugin
      dbip_country-1.0.0-win_amd64.glplugin
```

## Deployment target

The URLs inside `catalog.json` point at:

```
https://plugins.magicboxlab.ai/gridlens/
  ├── catalog.json
  └── bundles/
      └── dbip_country-1.0.0-<platform>.glplugin
```

`plugins.magicboxlab.ai` is a CNAME for the
`magicboxlab-ai/plugins` GitHub Pages site (custom-domain
support enabled via the `CNAME` file at the repo root). The default
catalog URL hard-coded in `src/pluginCatalogStore.ts` matches, so end
users get this catalog out of the box — and we can move providers later
without breaking installed clients.

## How to publish to GitHub Pages

### One-time setup

1. In the `magicboxlab-ai` GitHub org, create a public repo named
   `plugins` (one umbrella repo per the domain — each MagicBoxLab
   product gets its own top-level subdirectory under it).
2. Push this directory's contents (everything under
   `published-catalog/`, including `CNAME` at the repo root and
   `gridlens/…` underneath) to the repo's `main` branch.
3. Repo Settings → Pages → Source: `main` branch, `/` root → Save.
4. Repo Settings → Pages → Custom domain: enter `plugins.magicboxlab.ai`
   and click *Save*. (GitHub picks up the `CNAME` file automatically;
   this UI step also enables HTTPS provisioning.)
5. At the DNS provider for `magicboxlab.ai`, add a `CNAME` record:
   `plugins → magicboxlab-ai.github.io`. Wait for HTTPS to provision
   (usually <15 min after DNS resolves).
6. Verify `https://plugins.magicboxlab.ai/gridlens/catalog.json`
   returns the catalog over HTTPS.

### Each release

From this GridLens repo:

```bash
# 1. Rebuild bundles for every platform you support.
for platform in any macosx_11_0_arm64 macosx_11_0_x86_64 \
                manylinux2014_x86_64 manylinux2014_aarch64 win_amd64; do
    python -m gridlens.plugins bundle docs/examples/plugins/dbip_country \
        --output published-catalog/gridlens/bundles \
        --platform "$platform" 2>/dev/null || true
done

# 2. Rebuild catalog.json pointing at the magicboxlab.ai/gridlens URL.
python -m gridlens.plugins catalog published-catalog/gridlens/bundles \
    --output published-catalog/gridlens/catalog.json \
    --base-url https://plugins.magicboxlab.ai/gridlens/bundles

# 3. Commit, then mirror to the publishing repo.
git add published-catalog
git commit -m "publish: dbip_country 1.0.0"
rsync -a --delete published-catalog/ /path/to/plugins-repo/
cd /path/to/plugins-repo && git commit -am "publish: dbip_country 1.0.0" && git push
```

> **Note**: `--platform any` will fail for `dbip_country` because
> `maxminddb` ships only C-extension wheels. That's expected — skip it.

### Automated publishing

The workflow in
[`../docs/examples/plugin-catalog-action.yml`](../docs/examples/plugin-catalog-action.yml)
does steps 1 and 2 on every push to `main` and pushes the result to
GitHub Pages automatically. Drop it into
`.github/workflows/build-catalog.yml` in `magicboxlab-ai/plugins`
and remember to keep a `CNAME` file at the repo root (or have the
workflow write one) so GitHub Pages keeps serving from
`plugins.magicboxlab.ai` after each deploy.

## How end users discover the catalog

No action required — GridLens ships with
`https://plugins.magicboxlab.ai/gridlens/catalog.json` pre-configured.
Users open *Preferences → Plugins → Catalog*, click **Install** on
`dbip_country`, and GridLens picks the correct platform bundle, verifies
the SHA-256, and installs it.

To add a private catalog alongside the official one, the user pastes a
second URL into the *Catalog URLs* list — results from every configured
catalog are merged in the UI.

## Verifying a bundle

```bash
python -m gridlens.plugins validate \
    published-catalog/gridlens/bundles/dbip_country-1.0.0-macosx_11_0_arm64.glplugin
```

Exits `0` on clean (possibly with warnings), `2` on any error.
