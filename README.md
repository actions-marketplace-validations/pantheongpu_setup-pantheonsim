# PantheonSim for GitHub Actions

GitHub's hosted runners have no GPU, so GPU code usually goes untested until
someone has a card free. This action gives a job simulated NVIDIA or AMD GPUs,
so CUDA and HIP code compiles and its tests run on every push and pull request,
on the standard `ubuntu-24.04` and `ubuntu-22.04` runners.

The GPUs are simulated by [PantheonSim](https://pantheonsim.com), an open-source
GPU simulator that runs unmodified programs on the CPU.
`nvidia-smi` and `rocm-smi` report the GPUs you asked for. Your compiler is the
real one. Your test suite runs as it is.

## CUDA

```yaml
jobs:
  cuda-tests:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: pantheongpu/setup-pantheonsim@v0
        with:
          gpu: nvidia/h100
          count: 2
      - run: |
          nvcc -arch=compute_90 my_test.cu -o my_test
          ./my_test
```

## HIP

Name an AMD Instinct card and the action installs ROCm's own `hipcc`:

```yaml
      - uses: pantheongpu/setup-pantheonsim@v0
        with:
          gpu: amd/mi300x
      - run: |
          hipcc --offload-arch=gfx942 my_test.hip -o my_test
          ./my_test
```

## CMake and ctest

Projects build and test as they are. CMake's own CUDA and HIP language
support works, and `ctest` runs the programs directly:

```yaml
      - uses: pantheongpu/setup-pantheonsim@v0
        with:
          gpu: nvidia/t4
      - run: |
          cmake -S . -B build -G Ninja
          cmake --build build
          ctest --test-dir build --output-on-failure
```

## Inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `gpu` | `nvidia/t4` | The GPU to simulate: `nvidia/t4`, `nvidia/a100`, `nvidia/h100`, `nvidia/b200`, `amd/mi300x`, `amd/mi350x` and [more](https://pantheonsim.com/#gpus) |
| `count` | `1` | How many GPUs |
| `cuda-toolkit` | `apt` on 24.04, `12.6` on 22.04 | `apt` installs Ubuntu's toolkit; `12.6` or `13.0` installs that `nvcc` from NVIDIA; `none` uses one the job already installed |
| `rocm` | `7.1` for AMD | The ROCm version whose `hipcc` to install; `none` uses one the job already installed |
| `library-path` | `true` | Puts the simulator's libraries on `LD_LIBRARY_PATH`, so programs run directly as well as under `vgpu run` |

## Things to know

- **Build for an architecture the card supports.** `compute_75` runs on a T4
  and every newer NVIDIA card, `compute_90` needs Hopper. On AMD, `gfx942` is
  the MI300X and MI325X, `gfx950` the MI350X.
- **It checks behaviour, not performance.** Timings mean nothing here. Keep a
  run on physical GPUs before a release.
- **Check return codes in your tests.** An unsupported call returns its CUDA or
  HIP error and prints why, so a test that ignores the code can still pass.
- **PyTorch: the ROCm build runs, the CUDA build doesn't.** PyTorch for ROCm
  loads on a simulated MI300X and runs common ops through its libraries;
  PyTorch for CUDA's bundled runtime refuses a simulated driver. Numba works.
- **Linux runners only.** Container jobs work too. The first run builds the
  simulator; later runs restore it from the cache.

## Versions

`@v0` follows the latest `v0.x.y` release. Each release pins a simulator
version that was tested with it, so nothing changes under your workflow
until you move to a new release. Pin an exact release, such as `@v0.1.0`,
to control when that happens.

More detail: [pantheonsim.com/ci](https://pantheonsim.com/ci/).
Questions and bugs: [pantheongpu/pantheonsim](https://github.com/pantheongpu/pantheonsim/issues).

Apache-2.0. CUDA and NVIDIA are trademarks of NVIDIA Corporation. AMD, Instinct
and ROCm are trademarks of Advanced Micro Devices, Inc. PantheonSim is not
affiliated with or endorsed by NVIDIA or AMD.
