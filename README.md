## `fik`: manage + generate TOML route files for Traefik

[traefik](https://traefik.io/) is a great tool... with a lousy configuration file format.  TOML's redundancy and verbosity combined with traefik's deeply nested data structures makes it *really* hard to write routes manually.  What's more, route files have to be placed in a global, common directory, rather than being versioned with their projects.

`fik` is a tool that solves both of these problems, by allowing traefik .toml route files to be generated from shell scripts, markdown files, YAML, or JSON, using simple directives like this in a configuration file:

~~~bash
backend foo "http://bar:80"
match something "Host:something.com"
match another1  "Host:another.com;PathPrefix:/foo"

backend bar "http://baz:80"

match complex-example
priority 1000
must-have "Host:www.example.com,example.com"
must-have "PathPrefix:/foo,/bar"
~~~

Or using the equivalent  YAML:

~~~yaml
backends:
  foo: { servers: { foo: { url: "http://bar:80" } } }
  bar: { servers: { foo: { url: "http://baz:80" } } }
frontends:
  another1:
    passHostHeader: true
    routes: { route001: { rule: "Host:another.com;PathPrefix:/foo" } }
  complex-example:
    passHostHeader: true
    priority: 1000
    routes:
      route001: { rule: "Host:www.example.com,example.com" }
      route002: { rule: "PathPrefix:/foo,/bar" }
  something:
    passHostHeader: true
    routes: { route001: { rule: "Host:something.com" } }
~~~

By placing the above shell script in a `.fik` file, or the above YAML in a `fik.yml` file (or one or both as `shell` and `yaml` blocks in a `fik.md` file!), you can then use `fik up` to convert the route specifications to TOML and install them in an appropriate global directory  (`/etc/traefik/routes/fik` by default).

Your projects aren't limited to a single YAML block or even configuration file, though: shell or `.md` files can include others, and `.md` files can contain multiple YAML or shell blocks.  You can even generate your routes programmatically!

However you generate them, you can use `fik toml` or `fik json` to see the resulting configuration, `fik diff` and `fik watch` to compare them against the last published routes, and `fik down` to remove all the routes from Traefik (without deleting your configuration).

### Contents

<!-- toc -->

- [Installation and Requirements](#installation-and-requirements)
  * [Optional Enhancements](#optional-enhancements)
- [Basic Use](#basic-use)
- [Traefik Configuration](#traefik-configuration)
- [Command-Line Interface](#command-line-interface)
- [Routing Directives](#routing-directives)
  * [Setting Arbitrary Proprties](#setting-arbitrary-proprties)
  * [Custom Directives Using YAML Blocks](#custom-directives-using-yaml-blocks)
- [TOML Filenames](#toml-filenames)

<!-- tocstop -->

### Installation and Requirements

`fik` is a bash script that requires bash 4.4 or better to run, along with the following minimum requirements:

* [jq](https://stedolan.github.io/jq/) 1.5 (if you're on linux, your distro probably has a package for it)
* [pytoml](https://pypi.org/project/pytoml/) installed for the default Python interpreter (via `pip install pytoml`, or your Linux distro's `pytoml` package)

To install `fik` itself, simply copy [the binary](bin/fik) to a directory on your `PATH`.  (If you have [basher](https://github.com/basherpm/basher), you can do this with `basher install bashup/fik`.)

#### Optional Enhancements

You may also want some additional tools, depending on what fik features you plan to use:

* If you are using `.yml` files or YAML blocks, you will need at least **one** of the following:
  * a `yaml2json` command (like [this one](https://github.com/bronze1man/yaml2json)) on your `PATH`,
  * PyYAML installed for the default Python interpreter (e.g. via `pip install PyYAML`), OR
  * [yaml2json.php](https://packagist.org/packages/dirtsimple/yaml2json) on your `PATH`
* If you're using `fik watch`, you need a `tput` command (typically installed with ncurses)
* Installing `colordiff` and/or `diff-highlight` will enable colorized diffs
* Installing `pygmentize` will enable colorized TOML output

### Basic Use

To use `fik`, simply create one or more of the following files:

* `fik.json` -- route information in JSON format
* `fik.yml` -- route information in YAML format
* `.fik` -- route information expressed as shell commands, plus optional configuration or extension functions
* `fik.md` -- a [jqmd](https://github.com/bashup/jqmd)-style literate devops file, containing `shell`, `json`, and/or `yaml` blocks

When run, `fik` searches upward from the current directory until it finds one of the above file(s), then executes in that directory (the "project"), loading any of those files that exist in the listed order, combining and overwriting any parts of the configuration that are defined by earlier files.

Shell code in `.fik` and `fik.md` both have access to the full [jqmd](https://github.com/bashup/jqmd) and [mdsh](https://github.com/bashup/mdsh) APIs, as well as the routing directives supplied by `fik`.  Shell code can also add new `fik` subcommands by defining shell functions named `fik.X`, where `X` is the name of the desired subcommand.  (So the `fik.foo` function would be run if you invoked `fik foo` on the command line.)

`fik` also runs shell code found in `/etc/traefik/fikrc` and `$HOME/.config/fikrc` (if they exist), reading them *before* searching for the project directory.  You can use these files to change the default traefik routes directory, or to add new commands or custom directives just as in a `.fik` or `fik.md` file.

### Traefik Configuration

By default, `fik` writes TOML files to `/etc/traefik/routes/fik`, under the assumption that Traefik is watching for .toml route files in `/etc/traefik/routes`, and that the `fik` subdirectory is reserved for use by `fik`-generated files.

If you need to change this default directory, you can do so by setting `FIK_ROUTES` in any shell code read by `fik`.  That is, either the global `/etc/traefik/fikrc` and `$HOME/.config/fikrc`, or a project-specific `.fik` or `fik.md`.  (Note: exporting `FIK_ROUTES` in the shell environment *has no effect*; `FIK_ROUTES` can **only** be meaningfully set from within a `fik` configuration file.)

Whatever you set `FIK_ROUTES` to (or even if you don't set it at all), you will need to make sure Traefik is configured to read routes from there, or from one of its parent directories.  The default `FIK_ROUTES` setting assumes you've configured Traefik's `file` provider like this:

```toml
[file]
  directory = "/etc/traefik/routes"
  watch = true
```

If you are running Traefik in a docker container and `fik` on the host, `FIK_ROUTES` should be the host's path to the directory, while the Traefik setting should reflect the container's path.

If you are only using `fik` to manage .toml route files, you can have Traefik watch `FIK_ROUTES`, instead of a parent directory.

### Command-Line Interface

`fik` offers the following subcommands:

* `fik up` -- publish the current calculated routes to a .toml file for Traefik to read
* `fik down` -- remove the .toml file generated by `fik up`
* `fik toml` -- output the current calculated routes in TOML format to the console, paging if longer than one screenful, and with colorization if `pygmentize` is installed
* `fik json` -- output the current calculated routes in jq-colorized JSON to the console, paging if longer than one screenful
* `fik diff` -- compare the current .toml file contents (in sorted JSON form) with the current generated routes (i.e. what `fik json` would output), as a paged, unified diff, with colorizing if `colordiff`, `diff-highlight`, and/or `pygmentize` are available.
* `fik watch` -- output the first screenful of `fik diff` every 2 seconds
* `fik key` -- output the current "project key" (see [TOML Filenames](#toml-filenames), below, for more info)
* `fik filename` -- output the absolute path of the .toml file that `fik up` will create and `fik down` will remove.

Note: commands' output is only paged or colorized if sent to a tty, or if `FIK_ISATTY=1` is in the environment.  (`FIK_ISATTY=0` can be set to disable coloring and paging even if the output is being sent to a tty.)

The pager can be set using `FIK_PAGER`, which defaults to `less -FRX`.  Colorizing can also be controlled with these environment variables:

* `FIK_TOML_COLOR` -- colorizer for TOML output, defaults to `pygmentize -f 256 -O style=igor -l toml` if the `pygmentize` command is available
* `FIK_COLORDIFF` -- main colorizer for `diff` output, defaults to `colordiff` if the `colordiff` command is available
* `FIK_DIFF_HIGHLIGHT` -- line-diff highlighter for `diff` output, defaults to `diff-highlight` if the `diff-highlight` command is available

### Routing Directives

The following directives can be used to define routes from shell code in `.fik` or from `shell` blocks in `fik.md`:

* `backend` *name [urls...]* -- set the current backend to *name*, optionally adding any given *urls* to the backend's `servers` list (equivalent to calling `server` on each url).
* `server` *url* -- add *url* to the `servers` for the current backend.  No duplicate-checking is done: keys are assigned sequentially as `url001`, `url002`, etc.
* `match` *name [rules...]* -- set the current frontend to *name*, optionally adding any given *rules* to the frontend's `routes` list (equivalent to calling `must-have` on each rule).  The frontend's `backend` is set to the current backend, and its `passHostHeader` is set to `true`.
* `must-have` *rule* -- add *rule* to the `routes` for the current frontend.  No duplicate-checking is done: keys are assigned sequentially as `rule001`, `rule002`, etc.
* `priority` *priority* -- set the current frontend's `priority` to *priority*
* `tls` *cert key [entrypoints...]* -- add a certificate/private-key pair to zero or more *entrypoints*.  If no *entrypoints* are given, the certificate is added to all entry points.  *cert* and *key* can be either filenames or the actual PEM data; if they're filenames, they must be readable by Traefik.

#### Setting Arbitrary Proprties

These directives do not cover all possible configuration settings, such as rate limits, healthchecks, etc.  You can configure these directly using `yaml` or `json` blocks in `fik.md` (or in `fik.yml` or `fik.json`), or by using jq filter expressions (via jqmd's API functions).

`fik` provides two convenience functions for applying `jq` filter expressions to the current backend or frontend:

* `backend-set` *expr [`@`]name[`=`value]...* -- apply the jq filter string *expr* to the current backend, optionally setting the named jq variables from shell variables or values
* `frontend-set` *expr [`@`]name[`=`value]...* -- apply the jq filter string *expr* to the current frontend, optionally setting the named jq variables from shell variables or values

For example, the directive `priority 100` is exactly equivalent to `frontend-set '.priority=100'` .  This means you can do things like `backend-set '.healthcheck.interval="10s"'` to set a 10 second healthcheck interval on the current back end.

(It's important to note that filter expression strings should be *single-quoted*, with any string values inside them *double-quoted*.  This is because jq string values are JSON strings (which must be double-quoted), and shell strings must be single-quoted to avoid interpolation.)

Both `backend-set` and `frontend-set` are wrappers around jqmd's `APPLY` function, which lets you convert shell variables and values into JSON strings or values and pass them into a jq expression.  So you could write something like: `backend-set '.healthcheck.interval=$i' i="$INTERVAL"` to automatically JSON-encode the contents of the `$INTERVAL` shell variable and make it available as the jq variable `$i` for the duration of the filter's execution.

(For more on how `APPLY` works, see the [jqmd functions documentation](https://github.com/bashup/jqmd/#adding-jq-code-and-data).)

#### Custom Directives Using YAML Blocks

If you are using a `fik.md` file, you can define your own directives using YAML blocks and interpolation.  For example, if your .fik md contains this:

~~~markdown
```yaml @func health-interval i="$1"
backends:
  \($fik_backend):
    healthcheck:
      interval: \($i)
```
~~~

Then at any point below that block, you will be able to call e.g. `health-interval "10s"` to set the current backend's healthcheck interval.  (Your blocks can interpolate `$fik_backend` and `$fik_frontend` to get the name of the current back or front end.)

Custom directives can be placed in a separate `.md` file, which can then be loaded using [`mdsh-run`](https://github.com/bashup/mdsh/#available-functions) from any script block or file.  (So if you have global common directives, you can put them in a global `.md` file and then `mdsh-run` that file from your `$HOME/.config/fikrc` or `/etc/traefik/fikrc`, to make them available to all projects.)

For more on the process of creating shell functions like these from markdown blocks, see the jqmd documentation on [reusable code blocks](https://github.com/bashup/jqmd/#reusable-blocks).

### TOML Filenames

Each `fik` project has its own, automatically-generated `.toml` routing file under `$FIK_ROUTES`.  This file's name **must** be unique to each project, to prevent one project's routes from overwriting another's.

To avoid the need for manually assigning unique keys, `fik` automatically generates a random UUID and saves it in a `.fik-project` file in the project directory.  You must make sure this file is moved when the project is moved, or is moved with the configuration files if you move them to a new directory.  Otherwise, you will end up with a new key and a duplicate route file.

Similarly, you must **not** copy the `.fik-project` to other directories, or they will share the same route file and overwrite each other!  (In most cases, you will also want to avoid checking this file into revision control, so that different checkouts of the same repository will have their own unique project IDs.)

If you are changing a project's key, you should remove its existing route file with `fik down` before the change, then create the new route file with `fik up` after the change.  Alternately, if you need to leave the routes in effect during the change, you can:

1. Run `fik filename` to get the project's *old* TOML filename
2. Remove or edit the `.fik-project` file
3. Run `fik filename` again, to get the project's *new* TOML filename
4. Rename the file given by step 1, to the filename given by step 3

You can then resume using `fik up`, `fik diff`, etc. with the new project key.
