# Getting Started

The basic process for using fromager to build a collection of wheels is:

1. Make a list of the top-level dependencies (applications, extension libraries,
   etc.) in a `requirements.txt` file.
2. Make a list of known constraints for common dependencies. For example, if you
   are building 2 applications that both depend on the same library but express
   that dependency in different ways, you can select the version of that library
   that you want so only one version is built. Save this list in a
   `constraints.txt` file.
3. Run `fromager bootstrap`, passing your `requirements.txt` and
   `constraints.txt`, to try to build the collection.
4. When a package fails to build, create a settings file and mark it as
   pre-built. This lets you move through the full set of dependencies quickly,
   and build a list of the problematic packages.
5. When the build completes, review the set of pre-built packages and
   iteratively "fix" each one so that you are able to build it. Typical reasons
   for failure include missing system dependencies, packages that have no source
   distributions, and packages for which wheels cannot be built from the source
   distribution because it is incomplete.

!!! note
    It may be useful to use a container to run fromager so you can use the
    `Containerfile` to manage the build-time dependencies. Refer to
    [Using Containers](how-tos/containers.md) for more details.

## Example Bootstrap Session

Let's walk through an example bootstrap session.

First, create a working directory. This is where fromager will create the
build output directories.

```bash
mkdir work
cd work
```

Next, create the `requirements.txt` file with a top-level dependency.

```bash
echo "stevedore" > requirements.txt
```

The version of `stevedore` available when this guide was written depends on
`pbr`, which is one of the packages that can be difficult to build. To avoid
those issues, we'll create a constraint file to limit the build to an older
version that is easier to handle.

```bash
echo "pbr<6" > constraints.txt
```

Now run the bootstrap process, asking fromager to prepare wheels for all of
the dependencies of `stevedore`, including their transitive dependencies.

```bash
fromager \
  --logs-dir logs \
  --settings-dir overrides \
  bootstrap \
  --requirements-file requirements.txt \
  --constraints-file constraints.txt \
  --work-dir work-dir \
  --wheel-server-dir wheels
```

The command should produce output that looks like:

```
...
12:20:13 fromager.resolver INFO computing a solution for stevedore
12:20:15 fromager.commands.bootstrap INFO stevedore: 0 built, 3 downloaded, 1 pre-built, 0 failed
12:20:15 fromager.commands.bootstrap INFO pbr: 1 built, 0 downloaded, 0 pre-built, 0 failed
12:20:15 fromager.commands.bootstrap INFO setuptools: 0 built, 0 downloaded, 1 pre-built, 0 failed
...
```

When the bootstrap command completes, the wheels directory should contain
wheels for all of the dependencies:

```bash
ls wheels/
pbr-5.11.1-py2.py3-none-any.whl*
setuptools-65.5.0-py3-none-any.whl*
stevedore-5.2.0-py3-none-any.whl*
```

## Bootstrap Command Options

The bootstrap command supports several options:

### Output Directories

- `--work-dir WORK_DIR` - Base directory for working files (default: `work-dir`)
- `--wheel-server-dir WHEEL_SERVER_DIR` - Output directory for wheels (default: `wheels`)
- `--logs-dir LOGS_DIR` - Output directory for logs (default: `logs`)

### Input Files

- `--requirements-file FILENAME` - Top-level requirements to build
- `--constraints-file FILENAME` - Version constraints to apply
- `--settings-dir DIR` - Directory containing package settings and overrides

### Build Options

- `--sdists-repo URL` - Repository URL for source distributions
- `--wheels-repo URL` - Repository URL for wheel distributions  
- `--variant VARIANT` - Build variant to use
- `--cleanup / --no-cleanup` - Clean up temporary directories (default: cleanup)
- `--prepare-source / --no-prepare-source` - Prepare source directories (default: prepare)

### Parallel Processing  

- `--jobs JOBS` - Number of parallel jobs (default: 1)

For a complete list of options, run:

```bash
fromager bootstrap --help
```

## Next Steps

Once you have a working bootstrap build:

1. **Review build results** - Check the logs directory for any build warnings or errors
2. **Customize problematic packages** - Create settings files for packages that failed to build
3. **Iterate and improve** - Gradually reduce the number of pre-built packages
4. **Set up CI/CD** - Automate your builds with GitHub Actions or similar

For more advanced usage patterns, see the [How-To Guides](how-tos/index.md).
