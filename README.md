# Naemon development environment

Contributing to an open source project can be a challenging task,
even without figuring out how to launch the corresponding software
inside an IDE.
We are more than happy to see that you are interested in
contributing to the Naemon Core project.

To help you getting started, we provide predefined configurations
for [Visual Studio Code](https://code.visualstudio.com/) which will
attach a debugger and has predefined tasks to run the tests.

Basically this is everything you need to start coding.

This repository will help you to setup a native development environment running
on Linux systems or inside a development container. This way you can contribute
to the project regardless of you are using Linux, macOS or Windows. The choice
is yours.

## Why a separate repository?

We decided to keep the Naemon source repository free from any dotfiles or
IDE-specific configurations.


## How to start

First of all you have to clone both repositories:
1. [Naemon Core](https://github.com/naemon/naemon-core)
2. [Naemon VS Code Configuration](https://github.com/naemon/naemon-vscode)

```sh
git clone https://github.com/naemon/naemon-core.git
git clone https://github.com/naemon/naemon-vscode.git
```

Now you need to copy the content of the `naemon-vscode` repository into the
`naemon-core` repository. Don't worry, the `naemon-core` repository will ignore
these files by default.

```sh
cp -r naemon-vscode/.vscode naemon-core/
cp -r naemon-vscode/.devcontainer naemon-core/
```


## Open the source code with Visual Studio Code

You can now open the `naemon-core` directory with Visual Studio Code. For an
optimal setup, we recommend installing the following extensions:

- [Visual Studio Code](https://code.visualstudio.com/)
  - [C/C++ Extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)
  - [Dev Containers Extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

> [!TIP]
> VS Code offers to install the recommended extensions as soon as you open
> the `naemon-core` repository, see `.vscode/extensions.json`. If you dismissed
> that notification, run `Extensions: Show Recommended Extensions` from the
> command palette (`F1`).

## Native Development on Linux or WSL2

In order to develop Naemon natively on Linux or WSL2, you need to have all the
required build dependencies installed, as described in the next section. We
provided instructions for Fedora and Ubuntu. Other distributions will work as
well, but you have to adapt the instructions to your specific distribution.

### Fedora 44
```
sudo dnf group install development-tools
sudo dnf install git glib2-devel help2man gperf gcc gcc-c++ gdb cmake pkgconfig automake autoconf nagios-plugins-all libtool perl-Test-Simple

sudo ln -s /usr/lib64/nagios /usr/lib/nagios
```

### Ubuntu 26.04
```
sudo apt-get install git build-essential automake gperf gcc g++ gdb cmake help2man libtool libglib2.0-dev pkg-config libtest-simple-perl monitoring-plugins
```

> [!NOTE]  
> This is the only difference between a native development setup and using a dev
> container. You can continue with the [VS Code Setup](#setup-vs-code) section.

## Development in a dev container

In case you do not want to use a native Linux system, you can use the
development container instead. This will require Docker to be installed and
running on your system. Nothing else will be required.


As soon as you open the `naemon-core` repository in VS Code, it will offer to
reopen the folder in the container. If it does not, run
`Dev Containers: Reopen in Container` from the command palette (`F1`).
**The first start builds the image and takes a few minutes.**

## Setup VS Code

1. Naemon requires a configuration file to launch.
Luckily there is a pre-configured task that will do all that for you.
From the menu select `Terminal > Run Task... > initial`
Normally you only need to run this task once.

2. You are ready to rock! Make your code changes, create breakpoints and so on.
To run Naemon with an debugger attached, select `Run and Debug > Start Debugging`
![VSCode with running Debugger](/images/vscode_debugger.png)

3. Before you push your code changes, please make sure that all the tests are still green.
Again there is a predefined task you can execute via
`Terminal > Run Task... > Run Tests`
If all tests passed, feel free to push you code and to create a pull request.

### Naemon configuration files
Just in case you want to provide your own `naemon.cfg` or any other configuration file
just copy the files to `build/etc/naemon/`



## Event broker modules

Naemon can load event broker modules such as
[mod_gearman](https://github.com/sni/mod_gearman) or the
[Statusengine broker](https://github.com/statusengine/broker) as shared
libraries. While on a native Linux system you can load them from anywhere,
inside the dev container things are a bit different. Continue reading
if you are interested in how to build and debug Naemon Event Broker modules
inside a dev container.

Clone the modules **next to** your naemon-core checkout:

```
git clone https://github.com/naemon/naemon-core.git
git clone https://github.com/sni/mod_gearman.git
git clone https://github.com/statusengine/broker.git
```

That parent directory is mounted at `/workspaces/modules` inside the container.
If you keep your module sources somewhere else, copy `.devcontainer/.env.example`
to `.devcontainer/.env` and set `NAEMON_MODULES_DIR`.

Both modules need a Gearman job server, which is started as a second container
alongside the dev container. It is reachable as `gearmand:4730` from within the
container, and as `localhost:4730` from your host.

Then, inside the container:

1. Run the task `initial` if you have not already. Both modules locate Naemon
   through `build/lib/pkgconfig/naemon.pc`, which is created by `make install`.

2. Build the module you are interested in:
   - `Terminal > Run Task... > broker module: build (mod_gearman)`
   - `Terminal > Run Task... > broker module: build (statusengine)`

3. Install the example configuration via
   `Terminal > Run Task... > broker modules: install config drop-ins`.
   This copies a `broker_module=` drop-in into
   `build/etc/naemon/module-conf.d/` and the matching module configuration into
   `build/etc/naemon/`. Existing files are never overwritten. Delete the drop-in
   of the module you do not want to load, and adjust the paths if needed.

4. Start the debugger with the launch configuration
   `Launch in gdb (with broker modules)`.

### Setting breakpoints inside a broker module

You do not need a second VS Code window. The module sources are mounted into
the very same container, so the running VS Code can open them directly.

The most comfortable way is to add the module to your workspace:
`File > Add Folder to Workspace...` and pick for example
`/workspaces/modules/statusengine`. It then shows up in the Explorer next to
naemon-core and you set breakpoints by clicking in the gutter, exactly like in
the Naemon sources. Alternatively just open a single file through
`File > Open File...` without adding the folder.

Two things are worth knowing:

- Until Naemon has `dlopen()`ed the module, a breakpoint in it stays **grey and
  unverified** and hovering it says the source file is not known yet. That is
  expected and not an error. The launch configuration sets
  `set breakpoint pending on`, so the breakpoint binds by itself the moment
  `neb_load_all_modules()` loads the module, and execution stops there.
- This only works if the module carries debug symbols. The build tasks above
  take care of that (`--enable-debug` for mod_gearman, `--buildtype=debug` for
  statusengine) and deliberately do not install the module, so it is loaded
  straight from the build directory where those symbols live.

Note that IntelliSense in the added folder uses its own defaults. Red squiggles
there say nothing about whether the module builds or whether the debugger
works.

To actually have checks executed by mod_gearman, start a worker in a second
terminal:

```
/workspaces/modules/mod_gearman/mod_gearman_worker \
    --config=/workspaces/naemon-core/build/etc/naemon/mod_gearman_worker.conf
```

`gearadmin --host gearmand --status` shows the queues Naemon has created, which
is a quick way to check that the module reached the job server.



## Screenshots

**Native Development on Linux:**

![VSCode with running Debugger](/images/vscode_debugger.png)

**Using the Dev Container on macOS:**

![Naemon running with loaded Broker Modules inside a dev container](/images/naemon_with_broker_via_devcontainer.png)

**Using WSL2 on Windows:**

![Using WSL2](/images/vscode_wsl.png)

## Known issues
If you get an error message like `Configured debug type 'cppdbg' is not supported` please make
sure you have the [C/C++ Extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools) for
VS Code installed and enabled.

