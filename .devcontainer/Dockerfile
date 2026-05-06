FROM mcr.microsoft.com/devcontainers/cpp:2-ubuntu24.04

ARG LLVM_VERSION=19
ARG NODE_VERSION=22
ARG USERNAME=vscode

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

# ── Node.js ───────────────────────────────────────────────────────────────────
RUN curl -fsSL "https://deb.nodesource.com/setup_${NODE_VERSION}.x" | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*

# ── LSP servers (TypeScript / JavaScript / Python) ───────────────────────────
# Major versions are pinned to keep image rebuilds reproducible while still
# picking up patch/minor fixes. Bump the majors deliberately when needed.
RUN npm install -g \
    typescript@^5 \
    typescript-language-server@^4 \
    pyright@^1 \
    vscode-langservers-extracted@^4 \
    @typescript-eslint/parser@^8 \
    @typescript-eslint/eslint-plugin@^8 \
    eslint@^9

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
#     cmake-format   → matches .cmake-format.yaml + pre-commit hook
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
    && "${HOME}/.local/bin/uv" tool install cmake-format \
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
