# Run parallel jobs, allocate cpu cores per job and allocate memory per job

Fromager provides `cpu_cores_per_job` and `memory_per_job_gb` options which are
related to build systems and can be used as a per package setting when multiple
cores and significant amount of memory is available on a build system. On the
other hand, the `--jobs` overrides the default calculations based on the other
settings. For example, when you pass `--jobs 4` then at most 4 processes will
run in parallel when building a given wheel. By default, fromager computes a
number of jobs and that value can be influenced based on the per-package
settings. Note that the jobs are all within the context of building a single
wheel.

The `--jobs` option of fromager allows to set the maximum number of wheel build
jobs to run in parallel. Below is an example which uses the `--jobs` option
along with the bootstrap command:

```bash
fromager --jobs 4 bootstrap torch
```

For this example, the maximum number of jobs fromager will run in parallel is 4.

The `cpu_cores_per_job` is a package setting that allows to scale parallel jobs
by available CPU cores. The default value is set to 1 which indicates as many
parallel processes will be created as CPU cores are available on the system.

## Understanding Job Allocation

### Global Job Control

The `--jobs` option controls the maximum number of package builds that can run simultaneously:

```bash
# Build up to 8 packages in parallel
fromager --jobs 8 bootstrap requirements.txt

# Single-threaded build (useful for debugging)
fromager --jobs 1 bootstrap torch
```

### Per-Package Resource Settings

Configure resource allocation for specific packages using settings files:

```yaml title="overrides/settings/numpy.yaml"
# Numpy needs multiple cores for compilation
cpu_cores_per_job: 4
memory_per_job_gb: 8

build_options:
  env:
    NPY_NUM_BUILD_JOBS: "4"
```

```yaml title="overrides/settings/torch.yaml"  
# PyTorch is very resource-intensive
cpu_cores_per_job: 8
memory_per_job_gb: 16

build_options:
  env:
    MAX_JOBS: "8"
    USE_CUDA: "1"
```

### Automatic Job Calculation

Fromager automatically calculates optimal job counts based on:

1. **Available CPU cores**
2. **Available memory**
3. **Per-package requirements**
4. **Global job limits**

```python
# Fromager's internal calculation (simplified)
available_cores = os.cpu_count()
available_memory_gb = psutil.virtual_memory().total / (1024**3)

max_jobs_by_cpu = available_cores // cpu_cores_per_job  
max_jobs_by_memory = available_memory_gb // memory_per_job_gb

actual_jobs = min(max_jobs_by_cpu, max_jobs_by_memory, global_job_limit)
```

## Resource Planning Examples

### Small System (4 cores, 8GB RAM)

```bash  
# Conservative settings for small system
fromager --jobs 2 bootstrap requirements.txt
```

```yaml title="overrides/settings.yaml"
# Global settings for resource-constrained environment
changelog:
  - "Optimized for small systems"

# Package-specific settings
packages:
  torch:
    cpu_cores_per_job: 2
    memory_per_job_gb: 4
    
  numpy:
    cpu_cores_per_job: 2
    memory_per_job_gb: 2
    
  pandas:
    cpu_cores_per_job: 1
    memory_per_job_gb: 2
```

### Large System (32 cores, 128GB RAM)

```bash
# Aggressive parallelization for large system
fromager --jobs 16 bootstrap requirements.txt
```

```yaml title="overrides/settings.yaml"
changelog:
  - "Optimized for high-performance build system"

packages:
  torch:
    cpu_cores_per_job: 8
    memory_per_job_gb: 16
    
  tensorflow:
    cpu_cores_per_job: 8  
    memory_per_job_gb: 12
    
  numpy:
    cpu_cores_per_job: 4
    memory_per_job_gb: 4
    
  # Lightweight packages can use minimal resources
  requests:
    cpu_cores_per_job: 1
    memory_per_job_gb: 1
```

### Cloud CI/CD Environment

```yaml title="overrides/settings/ci.yaml"
# GitHub Actions (2 cores, 7GB RAM)
changelog:
  - "Optimized for GitHub Actions"

packages:
  torch:
    pre_built: true  # Too resource-intensive for CI
    
  numpy:
    cpu_cores_per_job: 2
    memory_per_job_gb: 3
    
  scipy:
    cpu_cores_per_job: 2
    memory_per_job_gb: 3
```

## Monitoring and Optimization

### Resource Usage Monitoring

Monitor resource usage during builds:

```bash
# Terminal 1: Run build
fromager --jobs 4 bootstrap requirements.txt

# Terminal 2: Monitor resources
top -p $(pgrep -f fromager)
htop
```

For detailed monitoring:

```bash
# Install monitoring tools
pip install psutil

# Custom monitoring script
python3 << 'EOF'
import psutil
import time

while True:
    cpu = psutil.cpu_percent(interval=1)
    memory = psutil.virtual_memory()
    
    print(f"CPU: {cpu}% | Memory: {memory.percent}% "
          f"({memory.used // (1024**3)}GB / {memory.total // (1024**3)}GB)")
    
    time.sleep(5)
EOF
```

### Build Performance Analysis

Track build times to optimize settings:

```bash
# Run with timing
time fromager --jobs 4 bootstrap torch

# Detailed timing with logs
fromager --debug --jobs 4 bootstrap torch 2>&1 | 
  grep -E "(INFO|ERROR|WARNING)" | 
  tee build-timing.log
```

Analyze the logs:

```bash
# Find longest build times
grep "built in" build-timing.log | 
  sort -k5 -nr | 
  head -10

# Identify resource bottlenecks
grep -E "(memory|cores|jobs)" build-timing.log
```

## Advanced Parallel Strategies

### Tiered Building

Build packages in dependency order with different resource allocations:

```bash
# Stage 1: Build lightweight dependencies quickly
fromager --jobs 8 build-order requirements.txt | 
  head -20 > stage1.txt

fromager --jobs 8 build-sequence stage1.txt

# Stage 2: Build heavy packages with more resources per job  
fromager --jobs 2 build-sequence remaining.txt
```

### Package Grouping

Group packages by resource requirements:

```yaml title="overrides/settings/groups.yaml"
changelog:
  - "Organized packages by resource requirements"

# Lightweight packages - many parallel jobs
packages:
  requests:
    cpu_cores_per_job: 1
    memory_per_job_gb: 1
  
  click:
    cpu_cores_per_job: 1
    memory_per_job_gb: 1
    
  pyyaml:
    cpu_cores_per_job: 1
    memory_per_job_gb: 1

# Medium packages - moderate resources
  pandas:
    cpu_cores_per_job: 2
    memory_per_job_gb: 4
    
  scikit-learn:
    cpu_cores_per_job: 4
    memory_per_job_gb: 6

# Heavy packages - maximum resources
  torch:
    cpu_cores_per_job: 8
    memory_per_job_gb: 16
    
  tensorflow:
    cpu_cores_per_job: 8
    memory_per_job_gb: 16
```

### Dynamic Resource Adjustment

Adjust resources based on system load:

```bash
#!/bin/bash
# dynamic-build.sh

# Check system load
LOAD=$(uptime | awk '{print $10}' | sed 's/,//')
CORES=$(nproc)

# Adjust job count based on load
if (( $(echo "$LOAD > $CORES" | bc -l) )); then
    JOBS=1  # System under stress
elif (( $(echo "$LOAD > $(($CORES / 2))" | bc -l) )); then
    JOBS=$(($CORES / 2))  # Moderate load
else
    JOBS=$CORES  # Light load, full utilization
fi

echo "System load: $LOAD, Using $JOBS jobs"
fromager --jobs $JOBS bootstrap requirements.txt
```

## Best Practices

1. **Start Conservative** - Begin with lower job counts and increase based on monitoring
2. **Monitor Continuously** - Watch CPU, memory, and I/O usage during builds
3. **Package-Specific Tuning** - Heavy packages need more resources per job
4. **Consider Dependencies** - Some packages may not parallelize well internally
5. **Account for System Overhead** - Leave resources for the OS and other processes
6. **Test Different Configurations** - Profile builds to find optimal settings
7. **Document Resource Requirements** - Track what works for different environments

## Troubleshooting

### Out of Memory Errors

```
ERROR: Package build failed - Out of memory
```

**Solutions:**
- Reduce `--jobs` count
- Increase `memory_per_job_gb` for heavy packages
- Add swap space
- Use pre-built wheels for memory-intensive packages

### CPU Oversubscription

```
WARNING: High system load detected
```

**Solutions:**
- Reduce `cpu_cores_per_job` for parallel packages
- Lower `--jobs` count
- Monitor with `htop` and adjust accordingly

### Build Timeouts

```
ERROR: Build timeout exceeded
```

**Solutions:**
- Increase build timeout in settings
- Allocate more resources to slow packages
- Check for infinite loops or hanging processes

This parallel processing capability allows you to significantly speed up wheel building while maintaining system stability through proper resource management.
