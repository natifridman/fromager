# Configuration Reference

## Per-package Settings

Settings for individual packages can be placed in the `overrides/settings/`
directory. Files should be named using the canonicalized name of the package.
For example `flash_attn.yaml`.

### PackageSettings

::: fromager.packagesettings.PackageSettings
    options:
      show_root_heading: true
      show_source: false

### BuildOptions

::: fromager.packagesettings.BuildOptions
    options:
      show_root_heading: true
      show_source: false

### DownloadSource

::: fromager.packagesettings.DownloadSource
    options:
      show_root_heading: true
      show_source: false

### GitOptions

::: fromager.packagesettings.GitOptions
    options:
      show_root_heading: true
      show_source: false

### ResolverDist

::: fromager.packagesettings.ResolverDist
    options:
      show_root_heading: true
      show_source: false

### ProjectOverride

::: fromager.packagesettings.ProjectOverride
    options:
      show_root_heading: true
      show_source: false

## Global Settings

The global changelogs can be placed in `overrides/settings.yaml`.

If you prefer managing a single settings file, per-package settings can also be
kept in this file.

### SettingsFile

::: fromager.packagesettings.SettingsFile
    options:
      show_root_heading: true
      show_source: false
