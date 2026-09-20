# Ansible Development Languages

An idempotent, cross-platform Ansible role for installing development language
runtimes, compilers, package managers, and common developer tools. It supports
side-by-side versions, per-language tool selection, PATH configuration, and
optional project setup without requiring a separate playbook for each operating
system.

## What it does

- Detects the operating system, Linux distribution, CPU architecture, and
  Windows versus Unix installation strategy.
- Installs the requested entries from `dev_languages` and validates language
  names before changing the host.
- Keeps supported runtimes side by side and lets you mark one version as the
  default on PATH.
- Installs platform-appropriate build dependencies and developer tools.
- Is safe to run repeatedly: package tasks use present state, existing runtime
  directories are reused, and PATH changes are managed idempotently.

## Supported languages

| Language | Key | Default status | Default tools or notes |
| --- | --- | --- | --- |
| Python | `python` | Included | uv, Poetry, Hatch, pytest, Ruff, mypy, debugpy, IPython |
| Java | `java` | Included | Temurin JDK, Maven, Gradle |
| JavaScript | `javascript` | Included | Node.js, npm, pnpm, Yarn, ESLint, Prettier, npm-check-updates |
| TypeScript | `typescript` | Included | TypeScript, ts-node, tsx; uses Node.js |
| Ruby | `ruby` | Included | Bundler, Rake, RuboCop, RSpec, Solargraph |
| Go | `go` | Included | gopls, Delve, govulncheck, gofumpt |
| Rust | `rust` | Included | rust-analyzer, rustfmt, Clippy, cargo-audit, cargo-deny, cargo-nextest |
| PowerShell | `powershell` | Included | PSScriptAnalyzer, Pester, PSReadLine |
| C | `c` | Optional | Native compiler plus build, formatting, lint, and pkg-config tools |
| C++ | `c++` | Optional | C toolchain plus clangd, GDB, and ccache |
| C# | `c#` | Optional | .NET SDK, dotnet-format, CSharpier, dotnet-outdated-tool |
| Visual Basic | `visual_basic` | Optional | .NET SDK, dotnet-format, dotnet-outdated-tool |
| Perl | `perl` | Optional | Perl::Critic, Perl::Tidy, Perl::LanguageServer |
| R | `r` | Optional | pak, renv, lintr, styler, languageserver, testthat |
| PHP | `php` | Optional | Composer, PHPUnit 12, PHP-CS-Fixer, PHPStan, Psalm, Rector |
| Ada | `ada` | Optional | GNAT, GPRbuild, gnatcheck, gnatpp |
| VBScript | `vbscript` | Optional | Windows `cscript` and `wscript`; Windows only |
| Lua | `lua` | Optional | LuaRocks, luacheck, busted, StyLua |
| Swift | `swift` | Optional | swift-format and SourceKit-LSP |
| Objective-C | `objective_c` | Optional | Clang, Foundation support where available, clang-format, clang-tidy |

The included set is Python, Java, JavaScript, TypeScript, Ruby, Go, Rust, and
PowerShell. Optional languages are installed by adding an entry to
`dev_languages`.

## Platform support

- **Linux:** Debian/Ubuntu, RHEL/CentOS/Fedora/Rocky/Alma, SUSE, Alpine, and
  Arch on x86_64 and arm64/aarch64. Python is built through pyenv so versions
  behave consistently across distributions and architectures.
- **macOS:** Intel and Apple Silicon. Build dependencies are installed with
  Homebrew and Python uses pyenv.
- **Windows:** x64 and ARM64 where the upstream package supports it. Python,
  Java, Node.js, Go, PowerShell, and other runtimes use native installers or
  official release packages under `C:\DevLangs` as appropriate.

Some ecosystems have platform limits. For example, Apple frameworks such as
SwiftUI and UIKit require macOS and Xcode, and Windows VBScript is restored
only where the Windows feature is available.

## Requirements

On the control machine:

```bash
python3 -m pip install ansible
ansible-galaxy collection install -r requirements.yml
```

The repository includes an `ansible.cfg` that points to
`inventory/hosts.ini`. For Windows targets, configure WinRM and install the
collections from `requirements.yml` before running the playbook. The target
must also have the privileges needed to install OS packages and update PATH.

## Quick start

The default inventory targets the local machine:

```bash
ansible-playbook site.yml
```

Preview the hosts and collected facts first:

```bash
ansible all --list-hosts
ansible all -m ansible.builtin.setup
```

Run in check mode when the platform modules support it:

```bash
ansible-playbook site.yml --check --diff
```

After installation, open a new shell so PATH changes are loaded, then verify
the tools you selected:

```bash
python3 --version
java -version
node --version
go version
rustc --version
pwsh --version
```

## Configuration model

The role is driven by `dev_languages`, a list of dictionaries. Each entry
requires `name`; the other fields are optional and language-specific.

```yaml
dev_languages:
  - name: python
    version: "3.12.4"
    pip_version: latest
    pipx_version: latest
    tools:
      - uv
      - pytest
      - ruff
  - name: java
    version: "21"
    distribution: temurin
    default: true
```

Common fields:

| Field | Meaning |
| --- | --- |
| `name` | One of the keys in the supported languages table. Required. |
| `version` | Requested runtime version. `latest` is supported broadly; Java also supports `latest_lts`. |
| `tools` | Replaces the language's default tool list. Use `[]` for runtime/compiler only. |
| `default` | Makes this entry the preferred version when the language supports side-by-side defaults. |

An entry with `tools: []` deliberately installs no optional tools. Omitting
`tools` installs that language's defaults. If you declare the same language
multiple times, use distinct versions and mark at most one as `default`.

## Common recipes

### Choose a custom language set

Extra vars replace the default `dev_languages` list for that run:

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"python","version":"3.12.4","tools":["uv","pytest","ruff"]},
  {"name":"javascript","version":"latest_lts","default":true},
  {"name":"c","tools":[]}
]}'
```

For repeatable setup, put the same YAML in `group_vars/all.yml` or a host/group
vars file instead of passing JSON on the command line.

### Install every cross-platform language

Run the complete supported cross-platform set with:

```bash
ansible-playbook site.yml -e '{
  "dev_languages": [
    {"name":"python"},
    {"name":"c"},
    {"name":"c++"},
    {"name":"java","default":true},
    {"name":"javascript","default":true},
    {"name":"typescript","default":true},
    {"name":"go","default":true},
    {"name":"rust","default":true},
    {"name":"powershell","default":true},
    {"name":"perl"},
    {"name":"c#","default":true},
    {"name":"visual_basic","default":true},
    {"name":"r"},
    {"name":"php"},
    {"name":"ada"},
    {"name":"ruby","default":true},
    {"name":"lua"},
    {"name":"swift"},
    {"name":"objective_c"}
  ]
}'
```

On Windows, add `{"name":"vbscript"}` to the list. VBScript is Windows-only
and should not be included on Linux or macOS.

### Install multiple Java versions

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"java","version":"17"},
  {"name":"java","version":"21","default":true,"tools":["maven","gradle"]}
]}'
```

Java uses Temurin archives from Adoptium and installs each version under the
configured Java root. Use `latest` for the newest GA feature release or
`latest_lts` for the newest Adoptium LTS release.

### Configure Python and virtual environments

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"python","version":"3.12.4","venvs":[
    {"path":"/Users/me/.venvs/project","system_site_packages":false}
  ]}
]}'
```

The parent directory of each virtual environment must already exist. On
Windows, use a Windows path such as `C:\\Users\\me\\.venvs\\project`.

### Configure JavaScript project version files

The `projects` option writes `.nvmrc` and `.node-version` in existing project
directories:

```yaml
dev_languages:
  - name: javascript
    version: latest
    default: true
    projects:
      - path: /workspace/my-app
      - path: /workspace/another-app
        version_files:
          - .nvmrc
```

The project directories must already exist. TypeScript uses Node.js as its
runtime and accepts `node_version` independently from the TypeScript version:

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"typescript","version":"5.7.3","node_version":"latest_lts","tools":["typescript","tsx"]}
]}'
```

### Install an optional language

```bash
ansible-playbook site.yml -e '{"dev_languages":[
  {"name":"go","version":"latest","tools":[]},
  {"name":"lua","version":"latest"},
  {"name":"swift","version":"latest"}
]}'
```

The role validates every `name`, so a typo fails early with the supported
language list rather than partially installing an unknown language.

## Remote hosts

Add hosts to `inventory/hosts.ini`, then target a group:

```ini
[linux]
workstation1.example.com ansible_user=devuser

[windows]
win-workstation.example.com

[windows:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_port=5986
ansible_winrm_server_cert_validation=ignore
```

Run against a selected group or host:

```bash
ansible-playbook site.yml -l linux
ansible-playbook site.yml -l win-workstation.example.com
```

Keep credentials outside the repository, preferably in Ansible Vault or your
normal secret-management system.

## Paths and overrides

The main defaults are defined in
`roles/dev_languages/defaults/main.yml` and can be overridden in vars files or
extra vars:

| Variable | Unix default | Windows default |
| --- | --- | --- |
| `pyenv_root` | `{{ ansible_env.HOME }}/.pyenv` | Not used |
| `java_root` | `{{ ansible_env.HOME }}/.jdks` | `C:\\DevLangs\\Java` |
| `node_root` | `{{ ansible_env.HOME }}/.nodejs` | `C:\\DevLangs\\Node` |
| `go_root` | `{{ ansible_env.HOME }}/.go/versions` | `C:\\DevLangs\\Go` |
| `powershell_root` | `{{ ansible_env.HOME }}/.powershell/versions` | `C:\\DevLangs\\PowerShell` |
| `windows_install_root` | Not used | `C:\\DevLangs` |

## Troubleshooting

- **Unsupported language:** use the exact key from the supported languages
  table, including `c++`, `c#`, and `objective_c`.
- **Command not found after success:** start a new shell or source the shell
  configuration file updated by the role.
- **Python build failure:** confirm the target can install OS build
  dependencies and has network access to pyenv and Python source downloads.
- **Windows connection failure:** test WinRM separately with
  `ansible windows -m ansible.windows.win_ping`.
- **A project version file was not written:** confirm the project directory
  exists on the target, not only on the control machine.
- **A tool is missing:** specifying `tools` replaces the defaults; add the tool
  explicitly or remove `tools` from the entry.

## Adding a language

1. Add its key to `supported_languages` in `group_vars/all.yml`.
2. Add `roles/dev_languages/tasks/<name>/main.yml` and dispatch to the relevant
   platform task files.
3. Add platform-specific build dependencies to `roles/dev_languages/vars/` when
   needed.
4. Make installation and PATH changes idempotent, and document the new entry in
   the supported-language table.
5. Run syntax checks on every supported platform path you can exercise.

## License

This project is distributed under the MIT License. See [LICENSE](LICENSE) for
the full text.
