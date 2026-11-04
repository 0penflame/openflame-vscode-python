# OpenFlame build of the Python Debugger

This branch is the upstream `microsoft/vscode-python-debugger` release named
in the branch (`openflame-<upstream version>`) plus a small patch so that a
Python service running in an OpenFlame workspace can be attached to from
code-server with the OpenFlame process picker
(`0penflame/openflame-vscode-process-picker`).

`openflame-2026.6.0` started from the `v2026.6.0` tag. This fork's `main`,
which tracks upstream `main`, was later merged into it so the branch merges
cleanly into `main`. The branch therefore also carries the upstream commits
made on `main` after `v2026.6.0`: dependency and GitHub Actions updates,
debugpy 1.8.21 in place of 1.8.20 (`DEBUGPY_VERSION` in `noxfile.py`), the
`breakOnSystemExit` launch/attach setting, the pytest multiprocessing
child-exit fix, the Windows process tree loading fix and the
`noConfigScripts` shebang fix. The version stays
`2026.6.0-openflame.1` rather than upstream `main`'s `2026.7.0-dev`.

The OpenFlame workspace image installs `ms-python.python` from Open VSX, which
pulls in `ms-python.debugpy` as a dependency, then installs this build over it
from `extensions/python-debugger.vsix`.

## Changes

1. **`${command:PickOpenflameProcess}` and `${command:RunOpenflameProcess}`
   for `debugpy` configurations** (`package.json`, `contributes.debuggers[debugpy].variables`).
   The OpenFlame service templates attach with
   `"processId": "${command:PickOpenflameProcess}"` (or `RunOpenflameProcess`,
   which first restarts the service through the workspace API). Those names
   are debugger variables, so they only resolve for debugger types that
   declare them. The picker declares them for the legacy `python` type only;
   upstream has moved Python debugging to the `debugpy` type, and without
   this change a `debugpy` attach fails with
   `command 'PickOpenflameProcess' not found`. The variables map to the
   picker's commands, which return the process id of the service's
   `python server.py OPENFLAME_SERVICE=<service> OPENFLAME_LANG=PYTHON ...`
   process.

2. **Inject with gdb for OpenFlame attaches**
   (`src/extension/debugger/configuration/resolvers/attach.ts`, tests in
   `src/test/unittest/configuration/resolvers/attach.unit.test.ts`).
   When `processId` is `${command:PickOpenflameProcess}` or
   `${command:RunOpenflameProcess}`, the resolver adds
   `--disable-sys-remote-exec` to `debugpyArgs`. On Python 3.14+ debugpy
   otherwise injects with `sys.remote_exec()`, which runs only when the
   target next executes Python code; an OpenFlame service is a gRPC server
   idle between requests, so the attach times out ("Timed out waiting for
   debug server to connect") unless a request happens to arrive. With the
   flag debugpy uses its gdb injector, which works on an idle process. The
   workspace image installs gdb for this. Configurations with any other
   `processId` are unchanged.

3. **Identity** (`package.json`): version `<upstream version>-openflame.<n>`
   and display name `Python Debugger (OpenFlame)`. The extension id stays
   `ms-python.debugpy`: `ms-python.python` declares a dependency on that id,
   and installing under another id would leave the stock extension installed
   beside this one, both registering the `debugpy` debug type. VS Code marks
   extensions installed from a `.vsix` as pinned, so auto-update does not
   replace this build with the Open VSX release; the Extensions view may
   still offer the stock version as an update.

## Not carried over

The 2022 OpenFlame debugger forks also changed process-list parsing (wider
Linux PID columns), process-name matching, and code-server-over-https
behavior. None of that is needed here:

- The OpenFlame path never uses this extension's own process list: the
  picker returns the process id directly. Upstream's `ps` parser
  (`src/extension/debugger/attachQuickPick/psProcessParser.ts`) matches the
  PID with `[0-9]+`, so 7-digit PIDs already parse for `${command:pickProcess}`.
- debugpy's adapter accepts `processId` as a string and converts it to an
  int, so the picker's string result needs no conversion.
- Attach-by-PID runs entirely on the workspace side: the adapter and the
  target process are both in the workspace container, and the
  extension opens no browser windows or local URLs.

## Building

```sh
npm ci
VSCETARGET=linux-x64 python -m nox --session install_bundled_libs   # bundled debugpy wheels for linux-x64
npx @vscode/vsce package --target linux-x64 -o python-debugger.vsix
```

`npm run lint` and `npm run format-check` must pass. The OpenFlame workspace
image is linux-x64; attach by PID uses gdb and the
`attach_linux_amd64.so` bundled with debugpy, so the workspace container
must allow ptrace between processes of the same user.

## Moving to the next upstream release

```sh
git fetch upstream --tags
git checkout -b openflame-<new version> v<new version>
git cherry-pick <the OpenFlame commits on the previous openflame-* branch>
```

Then set `version` to `<new version>-openflame.1`, check that the
`debugpy` debugger contribution in `package.json` still lists the two
OpenFlame variables, and check whether the bundled debugpy still prefers
`sys.remote_exec()` for attach by PID (`debugpy/server/cli.py`,
`attach_to_pid`) and still accepts `--disable-sys-remote-exec`. Build, run
the unit tests, and attach to a running service from the workspace to
confirm a breakpoint is hit.

If `main` has moved past the release tag when the pull request is opened,
merge `main` into the branch (no rebase or force-push), keep the
`<new version>-openflame.<n>` version when resolving the `package.json`
conflict, and list the upstream changes that came in with it at the top of
this file.
