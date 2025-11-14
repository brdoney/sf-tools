# Student First Tools

A collection of command-line tools built for my work at Student First. They are all contained in `bin` (hint: put the path to it in your `$PATH`).

- `build`: script for building and running the backend and frontend - I have aliases like `bb` for `build backend` and `bf` for `builf frontend` set up in my shell profile
- `docker-start`: manages a running instance of the backend, allowing it to be restarted (i.e. for code updates via the `build backend build` without restarting the containers)
- `db`: a small program for managing database migrations because I keep forgetting when to use `dotnet ef migrations` vs. `dotnet ef database update`
- `search`: general purpose tool for searching the codebase, like EF Core/database tables and Razor pages
- `workitems`: tool for pulling Azure work items in the terminal, in custom sorted order (I usually set it based on priority)
