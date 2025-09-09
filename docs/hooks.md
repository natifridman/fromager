# Fromager hooks and override plugins

For more complex customization requirements than are supported by the
configuration file, create an override plugin.

Plugins are registered using [entry
points](https://packaging.python.org/en/latest/specifications/entry-points/)
so they can be discovered and loaded at runtime. In `pyproject.toml`,
configure the entry point in the
`project.entry-points."fromager.project_overrides"` namespace to
link the canonical distribution name to an importable module.

```toml title="pyproject.toml snippet"
[project.entry-points."fromager.project_overrides"]
flit_core = "package_plugins.flit_core"
pyarrow = "package_plugins.pyarrow"
torch = "package_plugins.torch"
triton = "package_plugins.triton"
```

The plugins are treated as providing overriding implementations of
functions with default implementations, so it is only necessary to
implement the functions needed to make it possible to build the
package.

## Package settings hooks

!!! hook "Fromager Hook"
    These hooks allow you to customize package building behavior dynamically.

::: fromager.packagesettings.default_update_extra_environ
    options:
      show_root_heading: true
      show_source: false

The `update_extra_environ` can modify the extra environment variables
from settings file with dynamic values. The hook must update the
`extra_environ` dict in-place.

The hook is called multiple times during a build. The `version`
argument is *None* for `get_build_backend_dependencies` and
`get_build_sdist_dependencies`. For `get_install_dependencies_of_sdist`,
`build_sdist`, and `build_wheel`, the `version` argument
contains the resolved version.

*Added in version 0.60*

## Dependency hooks

!!! hook "Fromager Hook"
    These hooks control how dependencies are resolved and handled.

### Build System Dependencies

::: fromager.dependencies.default_get_build_system_dependencies
    options:
      show_root_heading: true
      show_source: false

### Build Backend Dependencies

::: fromager.dependencies.default_get_build_backend_dependencies
    options:
      show_root_heading: true
      show_source: false

### Build Sdist Dependencies  

::: fromager.dependencies.default_get_build_sdist_dependencies
    options:
      show_root_heading: true
      show_source: false

## Resolver hooks

!!! hook "Fromager Hook"
    These hooks customize package resolution and discovery.

::: fromager.resolver.default_resolver_provider
    options:
      show_root_heading: true
      show_source: false

## Source hooks

!!! hook "Fromager Hook"
    These hooks control how source code is downloaded and prepared.

### Download Source

::: fromager.sources.default_download_source
    options:
      show_root_heading: true
      show_source: false

### Resolve Source

::: fromager.sources.default_resolve_source
    options:
      show_root_heading: true
      show_source: false

### Build Sdist

::: fromager.sources.default_build_sdist
    options:
      show_root_heading: true
      show_source: false

## Wheel hooks

!!! hook "Fromager Hook"  
    These hooks customize wheel building behavior.

::: fromager.wheels.default_build_wheel
    options:
      show_root_heading: true
      show_source: false

## Hook Implementation Examples

### Basic Hook Structure

```python
# package_plugins/example.py

def update_extra_environ(
    req_type, 
    package_name, 
    version, 
    variant, 
    extra_environ
):
    """Custom environment setup for package building."""
    if req_type == "build_wheel":
        extra_environ["CUSTOM_BUILD_FLAG"] = "1"
        extra_environ["OPTIMIZATION_LEVEL"] = "3"

def download_source(
    req_type,
    package_name, 
    version,
    sdist_server_url,
    patches_dir
):
    """Custom source download logic."""
    # Custom download implementation
    pass
```

### Complex Plugin Example

```python
# package_plugins/torch.py
import os
import subprocess
from pathlib import Path

def update_extra_environ(req_type, package_name, version, variant, extra_environ):
    """Set up CUDA environment for PyTorch."""
    if req_type in ("build_wheel", "build_sdist"):
        # Enable CUDA support
        extra_environ["USE_CUDA"] = "1" 
        extra_environ["CUDA_HOME"] = "/usr/local/cuda"
        
        # Memory optimization
        extra_environ["MAX_JOBS"] = "4"

def download_source(req_type, package_name, version, sdist_server_url, patches_dir):
    """Custom PyTorch source preparation."""
    # Clone submodules needed for PyTorch
    subprocess.run([
        "git", "submodule", "update", "--init", "--recursive"
    ], check=True)
```

## Hook Registration

Register your hooks in `pyproject.toml`:

```toml
[project.entry-points."fromager.project_overrides"]
torch = "package_plugins.torch"
numpy = "package_plugins.numpy"
tensorflow = "package_plugins.tensorflow"

[project.entry-points."fromager.override_methods"]
# Global method overrides
download_source = "my_plugins.sources:custom_download_source"
build_wheel = "my_plugins.wheels:custom_build_wheel"
```

## Testing Hooks

Test your hooks with specific packages:

```bash
# Test with a specific package
fromager build --package torch --variant cuda

# Enable debug logging to see hook calls
fromager --debug build --package numpy
```

For more examples, see the [Fromager source code](https://github.com/python-wheel-build/fromager/tree/main/src/fromager) where default implementations are defined.
