FROM mcr.microsoft.com/devcontainers/cpp:2-ubuntu24.04

ARG LLVM_VERSION=19
ARG CMAKE_VERSION=3.31.12
ARG USERNAME=vscode
# Set WITH_CUDA=true (via .devcontainer/cuda/devcontainer.json) to bake the
# CUDA userspace toolkit into the image. The default (false) keeps the image
# lean for hosts without an NVIDIA GPU.
ARG WITH_CUDA=false
ARG CUDA_APT_VERSION=12-6

# Use bash with pipefail for all RUN commands
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# ── LLVM / Clang ─────────────────────────────────────────────────────────────
# clang-19 and friends ship in Ubuntu 24.04's universe repo, so no third-party
# apt source is needed. update-alternatives wires the unversioned tool names
# (clang, clang++, clangd, ...) to the versioned binaries.
RUN apt-get update && apt-get install -y --no-install-recommends \
        clang-${LLVM_VERSION} \
        clang-format-${LLVM_VERSION} \
        clang-tidy-${LLVM_VERSION} \
        clang-tools-${LLVM_VERSION} \
        clangd-${LLVM_VERSION} \
        lld-${LLVM_VERSION} \
        lldb-${LLVM_VERSION} \
        llvm-${LLVM_VERSION} \
        libc++-${LLVM_VERSION}-dev \
        libc++abi-${LLVM_VERSION}-dev \
    && for tool in clang clang++ clangd clang-format clang-tidy \
                   scan-build scan-view \
                   lldb lld ld.lld \
                   llvm-ar llvm-as llvm-cov llvm-dis llvm-dwarfdump \
                   llvm-link llvm-nm llvm-objcopy llvm-objdump \
                   llvm-profdata llvm-ranlib llvm-readelf llvm-size \
                   llvm-strings llvm-strip llvm-symbolizer; do \
         bin="/usr/bin/${tool}-${LLVM_VERSION}"; \
         [ -f "$bin" ] && update-alternatives --install "/usr/bin/${tool}" "${tool}" "${bin}" 100 || true; \
       done \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# ── Static analyzers / runtime checkers / docs ───────────────────────────────
# These back the matching CMake options:
#   ENABLE_CPPCHECK          → cppcheck
#   ENABLE_IWYU              → iwyu / include-what-you-use
#   ENABLE_DOXYGEN           → doxygen + graphviz (for `dot` graphs)
# And these are external tools the README/tutorial reference:
#   valgrind                 → ctest -T memcheck
#   shellcheck               → pre-commit hook
RUN apt-get update && apt-get install -y --no-install-recommends \
        cppcheck \
        iwyu \
        doxygen \
        graphviz \
        valgrind \
        shellcheck \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# ── Rust-side system tools ──────────────────────────────────────────────────
# Make this container a superset of the sibling rust_template's dev container.
# - mold        : fast linker — Rust's recommended modern linker on Linux.
# - musl-tools  : cross-compile to *-unknown-linux-musl targets.
# - libclang-dev: required by Rust crates that use `bindgen` (e.g. rocksdb,
#                 oniguruma) — pairs with the LLVM toolchain installed above.
# Note: lld is already wired through the LLVM apt block (Step "LLVM / Clang").
RUN apt-get update && apt-get install -y --no-install-recommends \
        mold \
        musl-tools \
        libclang-dev \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# ── ccache ───────────────────────────────────────────────────────────────────
RUN apt-get update && apt-get install -y --no-install-recommends \
        ccache \
    && rm -rf /var/lib/apt/lists/* \
    && echo "max_size = 0" > /etc/ccache.conf \
    && ln -sf /usr/bin/ccache /usr/local/bin/clang \
    && ln -sf /usr/bin/ccache /usr/local/bin/clang++ \
    && ln -sf /usr/bin/ccache /usr/local/bin/cc \
    && ln -sf /usr/bin/ccache /usr/local/bin/c++

# ── CMake ────────────────────────────────────────────────────────────────────
# Ubuntu 24.04 ships CMake 3.28; full C++20 modules support (esp. `import std`)
# needs 3.30+. Install Kitware's prebuilt tarball into /opt and symlink into
# /usr/local/bin so it shadows the apt-installed cmake without uninstalling it.
RUN ARCH="$(uname -m)" \
    && curl -fsSL "https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}/cmake-${CMAKE_VERSION}-linux-${ARCH}.tar.gz" \
        | tar -xz -C /opt \
    && for bin in cmake ctest cpack ccmake; do \
         ln -sf "/opt/cmake-${CMAKE_VERSION}-linux-${ARCH}/bin/${bin}" "/usr/local/bin/${bin}"; \
       done

# ── CUDA toolkit (optional) ──────────────────────────────────────────────────
# Installed only when WITH_CUDA=true (set by .devcontainer/cuda/devcontainer.json).
# The driver itself stays on the host and is passed in via --gpus=all at run
# time; this layer ships only the userspace toolkit (nvcc, headers, libs).
# Pick a toolkit version <= the host driver's max CUDA (driver 565 -> 12.7,
# so 12-6 packages are safe).
RUN if [ "${WITH_CUDA}" = "true" ]; then \
        curl -fsSLO "https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb" \
        && dpkg -i cuda-keyring_1.1-1_all.deb && rm cuda-keyring_1.1-1_all.deb \
        && apt-get update && apt-get install -y --no-install-recommends \
               "cuda-toolkit-${CUDA_APT_VERSION}" \
        && apt-get clean && rm -rf /var/lib/apt/lists/* ; \
    fi
# PATH/LD_LIBRARY_PATH entries are harmless when /usr/local/cuda is absent
# (the dynamic linker silently skips missing directories), so they can stay
# unconditional and keep this layer simple.
ENV PATH=/usr/local/cuda/bin:${PATH} \
    LD_LIBRARY_PATH=/usr/local/cuda/lib64

# ── User-level setup ──────────────────────────────────────────────────────────
USER ${USERNAME}
WORKDIR /home/${USERNAME}

# uv — single quotes intentional: $HOME must expand at shell runtime, not build time
# hadolint ignore=SC2016
RUN curl -LsSf https://astral.sh/uv/install.sh | sh \
    && echo 'export PATH="$HOME/.local/bin:$PATH"' >> "${HOME}/.bashrc" \
    && "${HOME}/.local/bin/uv" python install 3.12 3.13
# Older Python versions (3.10, 3.11) are tested in CI on ephemeral runners.
# Add them locally on demand:  uv python install 3.10 3.11

# Python-based dev tools — installed as standalone uv tools so each gets its
# own isolated env but lives on PATH (~/.local/bin):
#   C++ side (back the matching CMake options / pre-commit hooks):
#     cpplint        → ENABLE_CPPLINT
#     cmakelang      → provides cmake-format / cmake-lint executables
#                      (the `cmake-format` PyPI package is a stub with no
#                      entrypoints; uv tool install rejects it)
#     pre-commit     → .pre-commit-config.yaml runner
#     conan          → DEPENDENCY_MANAGER=CONAN
#   Python side (sibling python_template uses these — keeps one container
#   useful for both repos without switching):
#     ruff           → lint + format
#     mypy           → strict type check
#     bandit[toml]   → security lint
#     pip-audit      → CVE check for installed deps
#     commitizen     → optional Conventional Commits helper (`cz commit`)
RUN "${HOME}/.local/bin/uv" tool install cpplint \
    && "${HOME}/.local/bin/uv" tool install cmakelang \
    && "${HOME}/.local/bin/uv" tool install pre-commit \
    && "${HOME}/.local/bin/uv" tool install conan \
    && "${HOME}/.local/bin/uv" tool install ruff \
    && "${HOME}/.local/bin/uv" tool install mypy \
    && "${HOME}/.local/bin/uv" tool install "bandit[toml]" \
    && "${HOME}/.local/bin/uv" tool install pip-audit \
    && "${HOME}/.local/bin/uv" tool install commitizen

# Rust via rustup — stable only (saves ~1.4 GB vs. installing beta + nightly).
# The rust_template's CI matrix tests beta on ephemeral GitHub runners; local
# devs almost never need beta/nightly. Add `rustup toolchain install beta
# nightly` here if you want them in the image.
#
# `--profile default` already pulls in rustfmt + clippy; we add rust-src,
# rust-analyzer, llvm-tools-preview on top (for IDE source navigation,
# the bundled rust-analyzer LSP, and cargo-llvm-cov respectively).
# Single quotes intentional: $HOME must expand at shell runtime, not build time.
# hadolint ignore=SC2016
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | \
        sh -s -- -y --default-toolchain stable --profile default \
    && echo 'source "$HOME/.cargo/env"' >> "${HOME}/.bashrc" \
    && "${HOME}/.cargo/bin/rustup" component add \
            rust-src rust-analyzer llvm-tools-preview

# Cargo extensions — match the rust_template image so this container drives
# the sibling repo without switching. `cargo install --locked` uses each
# crate's pinned Cargo.lock for build reproducibility.
# hadolint ignore=SC2016
RUN "${HOME}/.cargo/bin/cargo" install --locked \
        cargo-nextest \
        cargo-llvm-cov \
        cargo-audit \
        cargo-deny \
        cargo-machete \
        cargo-msrv \
        cargo-watch \
        cargo-edit \
        cargo-outdated \
        mdbook

# vcpkg — backs DEPENDENCY_MANAGER=VCPKG. --depth 1 keeps the image lean
# (full history is ~600 MB; shallow clone is ~250 MB). VCPKG_ROOT is
# exported so cmake's --toolchain flag finds it without manual config.
# hadolint ignore=SC2016
RUN git clone --depth 1 https://github.com/microsoft/vcpkg.git "${HOME}/vcpkg" \
    && "${HOME}/vcpkg/bootstrap-vcpkg.sh" -disableMetrics \
    && echo 'export VCPKG_ROOT="$HOME/vcpkg"' >> "${HOME}/.bashrc" \
    && echo 'export PATH="$VCPKG_ROOT:$PATH"' >> "${HOME}/.bashrc"

# ccache / compiler env
RUN echo 'export CCACHE_CONFIGPATH=/etc/ccache.conf' >> "${HOME}/.bashrc" \
    && echo 'export CC="clang"' >> "${HOME}/.bashrc" \
    && echo 'export CXX="clang++"' >> "${HOME}/.bashrc"

# devcontainer runtime expects root as the final USER; remoteUser in
# devcontainer.json controls the actual connected user (vscode).
# hadolint ignore=DL3002
USER root
