# Using Build Containers

We can revisit the [getting started](../getting-started.md) example using containers in podman to
manage build dependencies and tools in a repeatable way.

We will use `pydantic-core` and a Universal Base Image (UBI) for Red Hat
Enterprise Linux 9 to demonstrate debugging and fixing a build failure.

## Inputs

We start with the same `requirements.txt` file:

```text title="requirements.txt"
pydantic-core
```

and an empty `constraints.txt`.

The build container includes Python, rust, and a virtualenv with fromager
installed:

```dockerfile title="Containerfile"
FROM registry.access.redhat.com/ubi9/ubi:latest

# Install system dependencies
RUN dnf install -y \
    python3 \
    python3-pip \
    python3-venv \
    git \
    rust \
    cargo \
    gcc \
    gcc-c++ \
    make \
    cmake \
    pkg-config \
    openssl-devel \
    libffi-devel

# Create virtual environment
RUN python3 -m venv /opt/fromager-env
ENV PATH="/opt/fromager-env/bin:$PATH"

# Install fromager
RUN pip install --upgrade pip
RUN pip install fromager

# Set working directory
WORKDIR /work

# Copy requirements
COPY requirements.txt constraints.txt ./

# Default command
CMD ["bash"]
```

## Building the Container

Build the container image:

```bash
podman build -t fromager-build .
```

## Running the Bootstrap

Run the bootstrap process in the container:

```bash
podman run -it --rm \
  -v $(pwd):/work:z \
  -v $(pwd)/wheels:/work/wheels:z \
  -v $(pwd)/logs:/work/logs:z \
  fromager-build \
  fromager bootstrap \
    --requirements-file requirements.txt \
    --constraints-file constraints.txt \
    --work-dir work-dir \
    --wheel-server-dir wheels
```

## Common Build Issues

### Missing System Dependencies

**Problem**: Package fails to build due to missing system libraries.

**Solution**: Add the dependencies to your Containerfile:

```dockerfile
# For packages requiring specific libraries
RUN dnf install -y \
    postgresql-devel \
    mysql-devel \
    sqlite-devel \
    redis \
    memcached-devel
```

### Rust Compilation Issues

**Problem**: Rust packages fail to compile.

**Solution**: Set up Rust environment properly:

```dockerfile  
# Install specific Rust version
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:$PATH"

# Set Rust compilation flags
ENV RUSTFLAGS="-C target-cpu=native"
ENV CARGO_NET_GIT_FETCH_WITH_CLI=true
```

### Memory Issues

**Problem**: Builds fail due to insufficient memory.

**Solution**: Run container with more memory and limit parallel builds:

```bash
podman run -it --rm \
  --memory=8g \
  --memory-swap=16g \
  -v $(pwd):/work:z \
  fromager-build \
  fromager --jobs 2 bootstrap pydantic-core
```

## Advanced Container Patterns

### Multi-stage Builds

Use multi-stage builds to optimize container size:

```dockerfile
# Build stage
FROM registry.access.redhat.com/ubi9/ubi:latest AS builder

RUN dnf install -y python3 python3-pip rust cargo gcc
RUN pip3 install fromager

# Production stage  
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest

COPY --from=builder /usr/local /usr/local
COPY --from=builder /opt/fromager-env /opt/fromager-env

WORKDIR /work
```

### Caching Build Dependencies

Mount cache directories to speed up builds:

```bash
podman run -it --rm \
  -v $(pwd):/work:z \
  -v fromager-cache:/root/.cache:z \
  -v cargo-cache:/root/.cargo:z \
  fromager-build \
  fromager bootstrap requirements.txt
```

### Using Docker Compose

For complex setups, use Docker Compose:

```yaml title="docker-compose.yml"
version: '3.8'

services:
  fromager:
    build: .
    volumes:
      - .:/work
      - ./wheels:/work/wheels
      - ./logs:/work/logs
      - fromager-cache:/root/.cache
    environment:
      - FROMAGER_JOBS=4
    command: >
      fromager bootstrap
      --requirements-file requirements.txt
      --constraints-file constraints.txt
      --work-dir work-dir
      --wheel-server-dir wheels

volumes:
  fromager-cache:
```

Run with:

```bash
docker-compose up fromager
```

## Debugging Container Builds

### Interactive Debugging

Drop into a shell when builds fail:

```bash
podman run -it --rm \
  -v $(pwd):/work:z \
  fromager-build bash

# Inside container
fromager --debug bootstrap pydantic-core
```

### Log Analysis  

Mount logs directory and analyze build failures:

```bash
# Run build with detailed logging
podman run -it --rm \
  -v $(pwd):/work:z \
  -v $(pwd)/logs:/work/logs:z \
  fromager-build \
  fromager --logs-dir logs \
    --debug bootstrap pydantic-core

# Analyze logs
ls logs/
cat logs/pydantic-core-build.log
```

### System Resource Monitoring

Monitor resource usage during builds:

```bash
# In another terminal
podman stats

# Or run with resource limits
podman run -it --rm \
  --cpus=4 \
  --memory=8g \
  -v $(pwd):/work:z \
  fromager-build \
  fromager --jobs 2 bootstrap torch
```

## Container Best Practices

1. **Use specific base image tags** - Avoid `:latest` for reproducibility
2. **Multi-layer optimization** - Group related RUN commands
3. **Cache mount points** - Speed up repeated builds
4. **Security scanning** - Regularly scan images for vulnerabilities
5. **Resource limits** - Set appropriate CPU and memory limits
6. **Volume management** - Use named volumes for persistent data

## Production Deployment

### CI/CD Integration

```yaml title=".github/workflows/build.yml"
name: Build Wheels
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build container
        run: podman build -t fromager-build .
        
      - name: Run bootstrap
        run: |
          podman run --rm \
            -v $PWD:/work \
            fromager-build \
            fromager bootstrap requirements.txt
            
      - name: Upload wheels
        uses: actions/upload-artifact@v4
        with:
          name: wheels
          path: wheels/
```

### Registry Publishing

```bash
# Tag for registry
podman tag fromager-build registry.example.com/fromager:latest

# Push to registry  
podman push registry.example.com/fromager:latest
```

This approach provides isolation, reproducibility, and makes it easier to manage complex build dependencies across different environments.
