# ansible-dev-langs

Idempotent, platform-agnostic Ansible playbook that installs development
language toolchains on a workstation. Auto-detects OS family/distribution,
architecture (x86_64/arm64/aarch64), and installs the requested versions
correctly on PATH. Installs Python tooling and optional C compiler support.

## Supported platforms

- Linux: Debian/Ubuntu, RHEL/CentOS/Fedora/Rocky/Alma, SUSE, Alpine, Arch
  (x86_64 and arm64/aarch64) — installed via [pyenv](https://github.com/pyenv/pyenv),
  which builds from source so it works identically across distros/arches.
- macOS: Intel and Apple Silicon — same pyenv approach, build deps via Homebrew.
- Windows: x64 and ARM64 — official python.org installer, one directory per
  version under `C:\DevLangs`, added to the machine `PATH`.

## Requirements

```bash
python3 -m pip install ansible
ansible-galaxy collection install -r requirements.yml
```

For Windows targets, ensure WinRM is configured and reachable, and that the
required Ansible collections are installed from `requirements.yml`.

## Usage

Install the default language set (Python, default version) on localhost:

```bash
ansible-playbook site.yml
```

Install multiple Temurin JDK versions and select the default:

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"java","version":"17"},
  {"name":"java","version":"21","default":true,"tools":["maven","gradle"]}
]}'
```

Use `version: "latest"` for the newest GA feature release or
`version: "latest_lts"` for the newest Adoptium LTS release. Java uses Temurin
archives from Adoptium and installs each version side by side.

Install the C compiler alongside Python:

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"python","version":"latest"},
  {"name":"c"}
]}'
```

C uses the native package manager: GCC on Linux/macOS and LLVM/Clang via
winget on Windows.

C++ additionally installs clangd, GDB, and ccache by default. Override the
tool list with `tools: []` or a custom list in the C++ entry.

Install C++ with the same toolchain:

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"c++"}
]}'
```

By default, C also installs CMake, Ninja, formatting/lint tools, and
pkg-config. Install only the compiler with an empty tool list:

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"c","tools":[]}
]}'
```

Select Python and its optional tooling versions via extra vars:

```bash
ansible-playbook site.yml -e '{"dev_languages":[{"name":"python","version":"3.12.4","pip_version":"25.0","pipx_version":"1.7.1"}]}'
```

The default language set installs the latest stable Python, pip, and pipx.
Omit `pip_version` or `pipx_version` when that tool is not needed:

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"python","version":"3.12.4","pip_version":"25.0"}
]}'
```

Python also installs common development tools by default: uv, Poetry, Hatch,
pytest, Ruff, mypy, debugpy, and IPython. Override the list or disable them:

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"python","tools":["uv","pytest","ruff"]}
]}'
```

Use `"tools":[]` to install no additional Python tools.

Create virtual environments by adding `venvs` to a Python entry. Each
environment is created with that entry's Python interpreter:

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"python","version":"3.12.4","venvs":[
    {"path":"/Users/me/.venvs/project","system_site_packages":false}
  ]}
]}'
```

The target directory's parent must already exist. Virtual environments are
created with pip by Python's standard `venv` module.

Install multiple versions/languages in one run (as more languages are added):

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"python","version":"3.12.4"},
  {"name":"python","version":"3.11.9"}
]}'
```

Target remote hosts by adding them to `inventory/hosts.ini` and running:

```bash
ansible-playbook site.yml -l linux
ansible-playbook site.yml -l windows
```

## Adding a new language

1. Add its name to `supported_languages` in [group_vars/all.yml](group_vars/all.yml).
2. Create `roles/dev_languages/tasks/<name>/main.yml` that dispatches to
   `linux.yml` / `macos.yml` / `windows.yml` based on `ansible_system`,
   following the pattern used in `tasks/python/`.
3. Each OS-specific task file must be idempotent (check before installing)
   and must add the installed version's binaries to PATH (user shell rc
   files on Linux/macOS, `ansible.windows.win_path` on Windows).
4. If Linux build dependencies differ per distro, add entries to
   `roles/dev_languages/vars/<family-or-distro>.yml`.

## Idempotency notes

- Package installs use `state: present`, safe to re-run.
- pyenv/version builds are skipped if the version directory already exists.
- Windows installs are skipped if the version's install directory already
  contains `python.exe`.
- PATH updates use `blockinfile` (Linux/macOS) and `win_path` (Windows),
  both of which are safe to apply repeatedly.
