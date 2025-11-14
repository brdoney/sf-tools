# Sans Visual Studio

## Background

Before I give a guide, a bit of background on how Visual Studio runs containerized C# applications, so that you know what we're emulating.

First, Visual Studio starts up the containers according to `docker-compose.yml`, `docker-compose.override.yml` and `obj/docker-compose.override.debug.g.yml`. The most important one is the last one, which configures the application with the bind mounts that are needed for the container to:

- See the source code and compiled files relevant to the container in `/app` (i.e. `src/Services/Sis/Academics/GlobalEdTech.Sis.Academics.API` for `academicsservice`)
- See *all* of the source code in `/src`
- Access the remote debugger binary in  `/remote_debugger` (binary should be stored in `/home/brendandoney/vsdbg` and can be installed using [Microsoft's instructions](/home/brendandoney/vsdbg:/remote_debugger), macOS users can use the Linux instructions) - I haven't finished the script for it yet, but this is necessary for running a debugger
- NuGet package sources, for things like protoc from `Grpc.Tools` - good to know, because this can cause trouble with things like macOS b/c of architecture mismatches (i.e. ARM host vs. x86 containers)
- Certificates for ASP.NET
- Microsoft user secrets

The containers all have their entrypoint set to `DistrolessHelper.dll --wait` program, which is Visual Studio's tool for managing the container's waiting, running, and reloading of code (so a bit more, but similar to my `build-refresh` script). The `--wait` flag means to wait for further input, like `tail -f /dev/null`, so the containers basically just do nothing at launch. Since we don't have Visual Studio's tools, I do something similar to this `tail` trick but with some extra work to preserve the output of our code (like `Console.WriteLine` output) in each container's log via a FIFO pipe.

Next, Visual Studio, starts the relevant code in the container. I'm not sure exactly how it does this, but it's probably by communicating with DistrolessHelper so it can start a `dotnet` process to run our code and track the PID of that process so it can kill it later to restart the system + refresh the code. To emulate this, in `backend-refresh`, I use `docker exec` (or equivalent for `podman`, `container`, etc.) on *each container* to:

1. Execute a command that kills any previously started `dotnet` processes (i.e. the PID stored in `/tmp/debuggee.pid`)
2. Starts a new `dotnet` process to run our code
3. Stores the PID of the started process in `/tmp/debuggee.pid` for future use

Visual Studio stores the commands it uses to run our code in each container's `labels` section in `docker-compose.override.debug.g.yml`. In particular, the fields `com.microsoft.visualstudio.debuggee.program` and `com.microsoft.visualstudio.debuggee.arguments` are relevant to us and our `backend-refresh` program. The other two aren't used by our setup because they have to do with `DistrolessHelper`. I read these relevant fields in `backend-refresh` and try to keep their edits to a minimum.

That's it! From there, the program is running and we can restart the app without bringing down/up the containers using steps 1-3 above. In particular, because we use bind mounts for everything, if we compile our program on the host machine, **restarting the running program will use the new binaries** (our newly compiled code) without restarting the container! This is basically the more primitive "hot-reloading" that Visual Studio does, since its not smart enough to only swap out relevant chunks of code like it can on the frontend. Maybe one day 🤔.

The only other thing about the backend is that each container/service has a corresponding `*.staticwebassets.runtime.json` in `bin/Debug/net8.0/` which tells the ASP.NET where to serve static files from. Some black magic must happen in DistrolessHelper though, because these aren't paths built for the container: they're paths on the host machine instead of for the container. To fix these paths to work *inside* of containers I wrote `fix-staticwebassets` script, which converts host machine paths to container paths (e.g. `C:\Users\BrendanDoney\.nuget\packages\microsoft.aspnetcore.components.webassembly.authentication\8.0.14\staticwebassets/` -> `/.nuget/packages/microsoft.aspnetcore.components.webassembly.authentication/8.0.14/staticwebassets/`).

As a side note, the frontend is just a regular, non-containerized (native) C# program, so it can run cross-platform without issue using `dotnet run`.

## Setup

### SQL Server (Database)

Check the install guide for your OS, then look at the [setup](#database-setup)

#### SQL Server

Just use the instructions already in our Wiki. Nice and simple.

#### Linux

Install `mssql-server` binary from a [Microsoft repo](https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-setup). I tested the 2025 preview on Ubuntu 25.04 and it works fine, even though it says it's for Ubuntu 24.04 and in preview. 

Then, run `sudo /opt/mssql/bin/mssql-conf setup` and **remember the `sa` password you set**, since you'll need it later!

#### macOS

Microsoft doesn't publish an `mssql-server` binary for macOS, so you have to run one of [Microsoft's SQL Server containers](https://hub.docker.com/r/microsoft/mssql-server) (which are really Ubuntu containers with `mssql-server` and some other stuff installed). Unfortunately, Microsoft only publishes x86-64 versions of these at the moment, so you have to emulate them to run on macOS's ARM chips. At the time of writing, I could only get [Docker](https://www.docker.com/) and [Apple's native containers](https://github.com/apple/container) to actually run this image. The image failed during startup with anything else (`podman`, `colima`) though ymmv.

Because its running in a container, the database's data will be deleted if you ever delete the container. So just make sure you don't do that and keep backups 😆. I never tried using bind mounts or volumes to get around this but definitely update this guide if you do this!

<!-- TODO: Update with docker command -->

#### Database Setup

Now the whole team is back together 👏. You can import the databases from a bacpac backup:

```bash
sqlpackage /a:Import /tsn:tcp:localhost,1433 /tdn:globaledtech.Sis.Tenants.API_db /tu:sa /tp:Pass@word /sf:globaledtech.Sis.Tenants.API_db.bacpac /TargetEncryptConnection:False
sqlpackage /a:Import /tsn:tcp:localhost,1433 /tdn:QA2.Sis.Common.API_db /tu:sa /tp:Pass@word /sf:QA2.Sis.Common.API_db.bacpac /TargetEncryptConnection:False
```

And create a user (using a query via some SQL Server client like a VSCode extension or via [Miscoroft's go-sqlcmd](https://github.com/microsoft/go-sqlcmd) using the `sa` account and password set earlier):

```sql
CREATE LOGIN dbadmin
WITH PASSWORD = 'Pass@word', CHECK_POLICY = OFF;
```

Grant the user sysadmin privileges:

```sql
EXEC sp_addsrvrolemember 'dbadmin', 'sysadmin';
```

In case you ever need to export, it's basically the same as the `sqlpackage` import command but with the source and target arguments flipped (obviously change the target file name to whatever you want):

```bash
sqlpackage /a:Export /ssn:tcp:localhost,1433 /sdn:QA2.Sis.Common.API_db /su:sa /sp:Pass@word /tf:QA20251016.bacpac /TargetEncryptConnection:False
```

### ASP.NET Certificates

The commands in our Wiki worked have file names in all lowercase and worked fine on Windows, but it seems like the containers use PascalCase certificates on macOS/Linux. Maybe it's a difference in `dotnet` version or implementation? In any case, to be safe I created and trusted both lowercase and PascalCase versions.

```bash
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-academics-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-Academics-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-admissions-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-Admissions-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-careerservices-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-CareerServices-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-communications-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-Communications-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-documentmanagement-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-DocumentManagement-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-facilities-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-Facilities-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-financialaid-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-FinancialAid-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-studentaccounts-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-StudentAccounts-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-studentportal-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-StudentPortal-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-systemconfiguration-api.pfx -p globalsisdevkey
dotnet dev-certs https -ep ~/.aspnet/https/globaledtech-SystemConfiguration-api.pfx -p globalsisdevkey
dotnet dev-certs https --trust
```

### Scripts to Edit

I publish all of the scripts I use in a [GitHub repository](https://github.com/brdoney/sf-tools). Feel free to download it and even pitch in a PR if you add something! I definitely haven't tested every OS and edge case.

You do **have to edit some scripts to work on your machine** though, so take a look at them before running any. In particular:

- `backend-refresh`: set your container provider and Student First repo path
- `build`: same as above
- `db`: set Student First repo path (whelp, I'm seeing a pattern here, I should probably make a `.env` file)
- `fix-staticwebassets`: same as above

### macOS Only: Protocol Buffers and gRPC

For some reason, gRPC only publishes x86-64 binaries of the `protobuf` compiler and C# gRPC plugin for macOS in their [Grpc.Tools NuGet package](https://www.nuget.org/packages/grpc.tools/). `dotnet` uses these binaries to compile all of our `proto` files, so this breaks builds on macOS ARM.

Until this gets fixed, a hacky workaround is to replace these broken binaries with symlinks to ARM equivalents. The `protobuf` and `grpc` packages in [Homebrew](https://brew.sh/) provide these binaries. So the sequence is:

```
cd ~/.nuget/packages/grpc.tools/2.71.0/tools/macosx_x64
mv protoc protoc.bak
mv grpc_csharp_plugin grpc_csharp_plugin.bak
ln -s /opt/homebrew/bin/protoc protoc
ln -s /opt/homebrew/bin/grpc_csharp_plugin grpc_csharp_plugin
```

### Editor

Most likely, you're trying to use Visual Studio Code or JetBrains' Rider. Not sure if Rider is ready out of the box, but for VSCode I *heavily* recommend Microsoft's [C# Dev Kit extension](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit). Along with all of the regular C# extension's features, it adds a Visual Studio-style **Solution Explorer**, since the file explorer isn't great. Take a look at the `dotnet.automaticallySyncWithActiveItem` setting too. Just make sure to select the appropriate solution file (via the `.NET: Open Solution` command) so all of the features work. I tend to leave `Student First Full Stack.sln` selected so I can edit the backend and frontend in one single window.

I did have to edit some settings to make performance better and stop resource issues (*tons* of memory and file watchers):

```json
{
	"dotnet.preferVisualStudioCodeFileSystemWatcher": true,
	"files.watcherExclude": {
		"**/bin": true,
		"**/obj": true,
		"**/wwwroot": true
	},
}
```

As an aside, I tried Neovim but the Razor file editing experience was pretty bad at the moment. Way too slow and really bad highlighting, even with Treesitter.

## Development Workflow

1. Any time you're compiling from a clean build (or for the first time with this setup):

```bash
fix-staticwebassets
```

2. Start the backend. Hit the `r` key to recompile and restart the backend or the `q` key to quit.

```bash
build backend
```

3. Start the frontend. Same as the backend: `r` recompiles and restarts, `q` quits.

```bash
build frontend
```

If I ever need to see what's going on with a backend service I use VSCode's container extension or [lazydocker](https://github.com/jesseduffield/lazydocker), which can also use podman via `DOCKER_HOST=unix:///run/user/1000/podman/podman.sock ~/Documents/lazydocker/lazydocker` (I have this set to an alias).

## Additional OS-Specific Notes

In case you're curious how I arrived at the current tooling, here's how it happened:

### Linux

`docker-compose.vs.debug.g.yml` tweaks:

- Use Linux paths where appropriate
- Don't use a mount+environment variable for nuget fallback packages, since there isn't a parallel to VS's shared NuGetPackages folder
- Don't use VSTools (for DistrolessHelper and HotReloadAgent)
  - Instead, set entrypoint to `tail -f /dev/null`
  - Deleted `com.microsoft.visualstudio.debuggee.workingdirectory` and `com.microsoft.visualstudio.debuggee.killprogram`, since they're not used by my script anyway
- `C:\Users\BrendanDoney\AppData\Roaming\Microsoft\UserSecrets` becomes `~/.microsoft/usersecrets/`
- `C:\Users\BrendanDoney\vsdbg\vs2017u5:/remote_debugger:rw` becomes `~/vsdbg/`, a locally downloaded copy of `vsdbg`
- Redirect to named pipe so logs appear in Podman: 
  - `entrypoint: ["/bin/sh", "-c", "rm -f /var/log/studentfirst.log && mkfifo /var/log/studentfirst.log && (while :; do cat /var/log/studentfirst.log; done) & tail -f /dev/null"]`
  - Add ` > /var/log/studentfirst.log` to the end of every `com.microsoft.visualstudio.debuggee.arguments` string
  - Use the following environment variables in each applicable container:
    - `ASPNETCORE_LOGGING__CONSOLE__DISABLECOLORS=false` (existed before, was `true`)
    - `DOTNET_SYSTEM_CONSOLE_ALLOW_ANSI_COLOR_REDIRECTION=true`
    - `TERM=xterm`
- Remove `DOTNET_USE_POLLING_FILE_WATCHER=1` to improve performance a bit, since we're not using hot-reloading in the container anymore

### macOS

TODO
