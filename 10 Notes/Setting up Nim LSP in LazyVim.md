# Setting up Nim LSP in LazyVim

This guide configures [Nim Language Server](https://github.com/nim-lang/langserver) in LazyVim. It covers:

1. projects that use a global Nim installation; and
2. projects that use the Nimbus Build System and its vendored Nim toolchain.

The important distinction is:

- `nimlangserver` speaks LSP to Neovim;
- `nimsuggest` performs the actual Nim code analysis.

The language server can be installed globally in both cases. The `nimsuggest` executable must match the toolchain and dependency paths used by the project.

## Prerequisites

Install Nim Language Server globally:

```bash
nimble install -g nimlangserver
```

Check the executables:

```bash
~/.nimble/bin/nimlangserver --version
~/.nimble/bin/nimsuggest --version
```

`nimlangserver` requires a `nimsuggest` with protocol v3 support, available in Nim 1.6 and newer.

## Configure LazyVim globally

Create `~/.config/nvim/lua/plugins/nim.lua`:

```lua
return {
  {
    "neovim/nvim-lspconfig",
    opts = {
      servers = {
        nim_langserver = {
          -- Use the explicitly installed version instead of Mason's copy.
          mason = false,
          cmd = { vim.fn.expand("~/.nimble/bin/nimlangserver") },
          filetypes = { "nim" },
          root_markers = { "*.nimble", ".git" },
        },
      },
    },
  },
}
```

This provides the common configuration for every Nim project. Project-specific choices belong in a `.nvim.lua` file in the repository root.

Using an explicit path is useful when a stale Mason installation crashes while the current globally installed `nimlangserver` works.

## Enable repository-local configuration

Neovim only loads `.nvim.lua` files when the `exrc` option is enabled. Add this to the LazyVim configuration:

```lua
vim.opt.exrc = true
```

A local configuration file can execute arbitrary code. Inspect a repository's `.nvim.lua` before trusting it. Open the file in Neovim and run:

```vim
:trust
```

Trust is associated with the file's content hash. After modifying `.nvim.lua`, inspect and trust it again, then restart Neovim.

Neovim searches for `.nvim.lua` from the current working directory upwards. It therefore also works when Neovim is started without a filename and files are later opened through session restore or MiniFiles, provided Neovim was started inside the project.

## Case 1: project using global Nim

Create `.nvim.lua` in the repository root:

```lua
local nimble_bin = vim.fn.expand("~/.nimble/bin")

vim.lsp.config("nim_langserver", {
  cmd = { nimble_bin .. "/nimlangserver" },

  -- Make nimlangserver use the settings pushed during startup instead of
  -- requesting them after opening the first source file.
  capabilities = {
    workspace = {
      configuration = false,
    },
  },

  settings = {
    nim = {
      nimsuggestPath = nimble_bin .. "/nimsuggest",
      projectMapping = {
        {
          projectFile = "my_project.nim",
          fileRegex = "^(my_project[.]nim|src/.*[.]nim)$",
        },
      },
    },
  },
})
```

Replace the example project file and regular expression with paths appropriate for the repository.

### Project mappings

`nimsuggest` analyses code from an entry point. A source file must therefore be associated with an entry point whose import graph contains it.

Both `projectFile` and the path tested by `fileRegex` are relative to the LSP project root:

```lua
projectMapping = {
  {
    projectFile = "examples/ping.nim",
    fileRegex = "^examples/ping[.]nim$",
  },
  {
    projectFile = "my_library.nim",
    fileRegex = "^(my_library[.]nim|my_library/.*[.]nim)$",
  },
}
```

Put specific mappings before broad mappings. Do not map every `.nim` file to a library entry point if examples or tests are separate program graphs. Navigation may attach successfully but return no definition when a file is outside the selected graph.

It is also useful to declare real entry points in the `.nimble` file:

```nim
entryPoints = @[
  "my_library.nim",
  "examples/ping.nim",
]
```

However, explicit `projectMapping` entries are still helpful because behavior around automatic Nimble entry-point discovery differs between Nimble and language-server versions.

## Case 2: project using Nimbus Build System

Nimbus repositories vendor a matching Nim compiler and `nimsuggest`. Keep using the global `nimlangserver`, but point it at the repository's vendored `nimsuggest`.

First complete the project's setup so the vendored toolchain exists. In Logos Storage this is:

```bash
make setup
```

Then create `.nvim.lua` in the repository root:

```lua
local root = vim.fn.fnamemodify(
  debug.getinfo(1, "S").source:sub(2),
  ":p:h"
)
local nimble_bin = vim.fn.expand("~/.nimble/bin")
local nimbus_nimsuggest =
  root .. "/vendor/nimbus-build-system/vendor/Nim/bin/nimsuggest"

vim.lsp.config("nim_langserver", {
  cmd = { nimble_bin .. "/nimlangserver" },

  -- Causes config.nims to load nimbus-build-system.paths.
  cmd_env = {
    NIMBUS_BUILD_SYSTEM = "yes",
  },

  capabilities = {
    workspace = {
      configuration = false,
    },
  },

  settings = {
    nim = {
      nimsuggestPath = nimbus_nimsuggest,
      projectMapping = {
        {
          projectFile = "library/my_library.nim",
          fileRegex = "^(library/my_library[.]nim|library/.*[.]nim)$",
        },
        {
          projectFile = "tests/testAll.nim",
          fileRegex = "^(tests/testAll[.]nim|tests/.*[.]nim)$",
        },
        {
          projectFile = "application.nim",
          fileRegex = "^(application[.]nim|src/.*[.]nim)$",
        },
      },
    },
  },
})
```

Adjust the vendored `nimsuggest` path if the repository lays out Nimbus Build System differently.

`NIMBUS_BUILD_SYSTEM=yes` is significant: Nimbus-based `config.nims` files commonly use it to load generated dependency paths. Without it, `nimsuggest` may fail to resolve vendored packages even though the project builds normally through `make`.

### Exception inlay hints in large Nimbus projects

On a large project graph, exception inlay-hint calculation can make `nimsuggest` initialization appear to hang. If this happens, disable only exception hints:

```lua
settings = {
  nim = {
    nimsuggestPath = nimbus_nimsuggest,
    inlayHints = {
      exceptionHints = {
        enable = false,
      },
    },
    projectMapping = {
      -- ...
    },
  },
}
```

This keeps navigation and the other inlay hints enabled.

## Why `workspace.configuration = false`?

Neovim normally advertises support for pulling workspace configuration. Nim Language Server may then open the first Nim document and infer its project before the asynchronous settings response arrives.

Setting:

```lua
capabilities = {
  workspace = {
    configuration = false,
  },
}
```

makes it consume the configuration pushed at startup. This ensures `nimsuggestPath` and `projectMapping` are available before the first document is analysed.

## Verify the setup

Restart Neovim in the repository root, open a Nim file, and check:

```vim
:LspInfo
```

Confirm that:

- `nim_langserver` is attached;
- its command is `~/.nimble/bin/nimlangserver`; and
- the detected root is the intended repository.

Then place the cursor on a symbol and use:

```vim
gd
```

For more detail, inspect:

```vim
:messages
:LspLog
```

On a standard Linux/XDG installation, the same LSP log can be read directly at:

```text
~/.local/state/nvim/lsp.log
```

Accessing the file from the shell is often more convenient:

```bash
tail -f ~/.local/state/nvim/lsp.log
less ~/.local/state/nvim/lsp.log
rg 'Nimsuggest initialized|RegEx matched|ERROR' ~/.local/state/nvim/lsp.log
```

To ask Neovim for the exact path instead of assuming the default:

```vim
:lua print(vim.lsp.log.get_filename())
```

Useful successful log messages include:

```text
RegEx matched ...
Nimsuggest initialized for ...
```

The second line should name the intended project entry point.

## Troubleshooting

### The language server does not attach

1. Check `:set exrc?`.
2. Reopen `.nvim.lua`, inspect it, and run `:trust`.
3. Restart Neovim after changing or trusting the file.
4. Check `:LspInfo` and either `:LspLog` or `~/.local/state/nvim/lsp.log`.
5. Verify that the configured executables exist and are executable.

### `gd` returns no definition

Check which entry point initialized `nimsuggest`. The file may have matched the wrong `projectMapping`, or the chosen entry point may not import that file.

Remember:

- mappings are tested against root-relative paths;
- specific mappings should precede broad mappings;
- examples and tests often require their own entry points.

### Imports from vendored dependencies cannot be resolved

In a Nimbus project, confirm both:

```lua
nimsuggestPath = root .. "/vendor/nimbus-build-system/vendor/Nim/bin/nimsuggest"
```

and:

```lua
cmd_env = {
  NIMBUS_BUILD_SYSTEM = "yes",
}
```

Also make sure the repository setup completed successfully.

### The client exits with a crash or signal 11

Run the exact server shown by `:LspInfo` manually with `--version`. If LazyVim is using an older Mason copy, configure the explicit global path and set `mason = false`.

### The setup was cleaned or dependencies were rebuilt

After `make clean-all`, `make setup`, or replacement of the vendored Nim toolchain, restart Neovim. A running language-server process may still refer to an old `nimsuggest` process or socket.

### Nimbus initialization never finishes

Try disabling exception inlay hints as described above, then restart Neovim and inspect `:LspLog`.

## Working example: logos-storage-nim

The `logos-storage-nim` repository uses:

- global `~/.nimble/bin/nimlangserver`;
- vendored `vendor/nimbus-build-system/vendor/Nim/bin/nimsuggest`;
- `NIMBUS_BUILD_SYSTEM=yes`;
- precise mappings for the application, library, tests, and MIX transport entry points;
- disabled exception inlay hints because they stalled this large project graph.

This combination was verified with `gd` from `storage/node.nim` into a definition under the vendored `nim-libp2p` source tree.

## References

- [Nim Language Server](https://github.com/nim-lang/langserver)
- [LazyVim LSP configuration](https://www.lazyvim.org/plugins/lsp)
- [Neovim project-local configuration and `exrc`](https://neovim.io/doc/user/options/#'exrc')
- [Neovim trusted files and `:trust`](https://neovim.io/doc/user/editing/#trust)
