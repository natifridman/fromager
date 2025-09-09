# Using Constraints to Build Collections

Constraints are a way to specify the versions of packages that should be used
when building a collection. They are useful when you want to specify a version
of a package other than the default (usually the latest version).

Because several commands in fromager use constraints, you pass them to the base
command using the `--constraints-file` option.

For example, if you want to bootstrap a package that requires `setuptools`
and you want to avoid a breaking change in `setuptools` you can create a
constraints file that tells fromager to avoid using the latest version of
`setuptools`:

```text title="constraints.txt"
setuptools<80.0.0
```

Then you would run the following command:

```bash
fromager --constraints-file constraints.txt bootstrap my-package
```

This will use the constraints in the `constraints.txt` file to build
`my-package`.

Use the same constraints file with `fromager build-sequence` when building the
production packages.

## Common Constraint Patterns

### Version Pinning

Pin to exact versions for reproducible builds:

```text
setuptools==79.1.0
wheel==0.42.0
pip==24.0
```

### Range Constraints

Use version ranges for flexibility:

```text
# Avoid known problematic versions
setuptools>=60.0.0,<80.0.0
pydantic>=2.0.0,<3.0.0

# Stay on major version
numpy>=1.20.0,<2.0.0
```

### Pre-release Handling

Include or exclude pre-releases:

```text
# Exclude pre-releases (default)
tensorflow>=2.13.0

# Allow pre-releases for specific package
torch>=2.1.0a0
```

## Advanced Usage

### Multiple Constraint Files

Combine constraint files for different scenarios:

```bash
# Base constraints + environment-specific
fromager \
  --constraints-file base-constraints.txt \
  --constraints-file cuda-constraints.txt \
  bootstrap torch
```

### Dynamic Constraints

Use constraints to manage different build variants:

```text title="cpu-constraints.txt"
torch==2.1.0+cpu
torchvision==0.16.0+cpu
```

```text title="cuda-constraints.txt"  
torch==2.1.0+cu121
torchvision==0.16.0+cu121
```

### Dependency Resolution Order

Fromager resolves constraints in this order:

1. Command-line constraint files
2. Requirements file constraints  
3. Package settings constraints
4. Default resolution

## Troubleshooting

### Conflicting Constraints

If you see constraint conflicts:

```
ERROR: Cannot install package-a 1.0 and package-b 2.0 
because these package versions have conflicting dependencies
```

**Solution**: Adjust constraints to find compatible versions:

```text
package-a>=1.0,<2.0
package-b>=1.5,<3.0
```

### Over-constraining

Avoid being too restrictive:

```text
# Too restrictive - may cause resolution failures
setuptools==79.1.0
pip==24.0.0

# Better - allows compatible versions
setuptools>=79.0.0,<80.0.0
pip>=23.0.0
```

### Debug Resolution

Use verbose logging to understand constraint resolution:

```bash
fromager --debug \
  --constraints-file constraints.txt \
  bootstrap my-package
```

## Best Practices

1. **Start broad, then narrow** - Begin with loose constraints and tighten as needed
2. **Test constraint combinations** - Validate that all constraints work together  
3. **Document rationale** - Add comments explaining why specific constraints exist
4. **Version control constraints** - Track changes to constraint files
5. **Regular updates** - Periodically review and update constraints

## Examples

### Machine Learning Stack

```text title="ml-constraints.txt"
# Core ML libraries - stay compatible
numpy>=1.21.0,<2.0.0
pandas>=1.3.0,<3.0.0  
scikit-learn>=1.0.0,<2.0.0

# Deep learning - specific CUDA versions
torch==2.1.0+cu121
torchvision==0.16.0+cu121
tensorflow-gpu==2.13.0

# Avoid problematic versions
protobuf>=3.19.0,<5.0.0
```

### Web Development Stack

```text title="web-constraints.txt"
# Framework versions
django>=4.2.0,<5.0.0
fastapi>=0.100.0,<1.0.0

# Database connectors
psycopg2-binary>=2.9.0
sqlalchemy>=2.0.0,<3.0.0

# Async libraries  
asyncio-mqtt>=0.13.0
aiohttp>=3.8.0,<4.0.0
```
