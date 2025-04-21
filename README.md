# Getting started with Heroku .deb Packages: Nginx

Run Nginx with the .deb Packages Heroku Buildpack

>note
> The `.deb` is pronouced "dot Deh · b" like Debra, not "dot Deeb", like dweeb.

## Use

```
$ git clone <url>
$ cd nginx-deb-packages-getting-started
$ pack build nginx-image --path .
$ docker run -it --rm --env PORT=5006 -p 5006:5006 nginx-image
```

Then visit [localhost:5006](http://localhost:5006).

## Configure

You can change the settings in [nginx.template.conf](nginx.template.conf) to meet your needs. You can add additional files in the `public` directory to serve different contents.

## How does it work? Short

This buildpack uses the [heroku/deb-packages](https://github.com/heroku/buildpacks-deb-packages) buildpack, declared in the [project.toml](project.toml).

In that file it declares that it needs two buildpacks:

```toml
[[io.buildpacks.group]]
uri = "heroku/deb-packages"

[[io.buildpacks.group]]
uri = "heroku/procfile"
```

And we want to install the `nginx` Debian package:

```toml
[com.heroku.buildpacks.deb-packages]
install = [
    "nginx"
]
```

When you run `pack build` the `heroku/deb-packages` CNB installs nginx. The `heroku/procfile` CNB reads in the `Procfile` that you want to boot a server by calling `web: bin/server`.

This is a short script that converts the `nginx.template.conf` into an nginx config file (after applying config vars such as `${PORT}`) and then uses it to boot nginx.

The result is a containerized Nginx server built with Cloud Native Buildpacks.

## How does it work? Full

The `heroku/deb-packages` is a Cloud Native Buildpack (CNB). It installs packages on a Debian linux distribution such as Ubuntu. This is the distribution used by Heroku. You should reach for it when you might otherwise use `apt get` or `dpkg` in a `Dockerfile`.

You can use this buildpack to install any package, not just `nginx`.

## Browsing packages

You can view available packages on [packages.ubuntu.com/](https://packages.ubuntu.com/). Different releases have different codenames. Heroku stack images (base images) are named `heroku-{Ubuntu LTS version}`. That means `heroku-24` is built on top of Ubuntu 24.04. To search for packages on your specific base image you'll need to convert:

- heroku-24 (Ubuntu 24.04)
  - noble
- heroku-22 (Ubuntu 22.04)
  - jammy

Once you know the distribution name you can select it in the dropdown and enter a search term such as "nginx". From there you will end up on a page like:

[Searching "nginx" against noble (heroku-24: Ubuntu 24.04)](https://packages.ubuntu.com/search?keywords=nginx&searchon=names&suite=noble&section=all)

You will see many packages. At the time of writing (2025/04/18) there are 53 matching packages.

## Package naming conventions

If you need to build software that compiles or "links" against a package, you often need to pull in the `-dev` variant. Many packages are prefixed with `lib` such as `libxml2`. You will see variants like `nginx-light` and `nginx-core` you can find a comparison for various packages by [searching for information about them](https://askubuntu.com/questions/553937/what-is-the-difference-between-the-core-full-extras-and-light-packages-for-ngi).

In our case the plain `nginx` will work fine:

```toml
[[io.buildpacks.group]]
uri = "heroku/deb-packages"

[[io.buildpacks.group]]
uri = "heroku/procfile"

[com.heroku.buildpacks.deb-packages]
install = [
    "nginx"
]
```

If we needed additional packages, we could list them in the install array. See the [heroku/deb-packages](https://github.com/heroku/buildpacks-deb-packages) readme for additional config options.

## What is a Cloud Native Buildpack (CNB)?

Cloud Native Buildpacks compete with `Dockerfile` as a way to describe and build OCI images. You will find many single-purpose CNBs such as [this Paketo Nginx buildpack](https://github.com/paketo-buildpacks/nginx). That are easy to use, but when what you need isn't provided in a ready-built CNB, you can use the `heroku/deb-packages` to install anything available as a Debian package.

One benefit of CNBs is that by default they are "rebaseable." Applications built with Dockerfile cannot be rebased, they must be "rebuilt."

Usually when you need to roll out security updates to an underlying operating system, you must take the new OS and completely re-build all layers by invoking the Dockerfile. This is timeconsuming (when done at scale), tedius, and prone to intermittent failures.

In contrast a "rebaseable" image can swap out the OS layer and replay the previously generated images on top of it without needing to exectute or "rebuild" the project. The process is extremely fast. The simplicity and speed mean it's easier to roll out security updates more often.

## Rebaseable layers

One way that CNBs guarantee rebaseable layers is they're not allowed to modify global directories that the operating system might conflict with. (If you didn't know layers roughly correspond do directories on disk). That also means things get a little tricky when using older software that wasn't designed to be relocated or assumes the entire machine and every dir are available to it's whims.

We need to work around this limitation to get Nginx to work seamlessly with our CNB.

## Understanding Nginx

By default NGINX was designed with the idea that there would be one installed globally on the whole server and that many applications would be delivered. The following section explains quirks of this design philosophy that need to be adapted to run (well) on a containerized environment.

### Understanding Nginx: Daemon off

If you run `$ nginx`, the default behavior is to spawn a "daemon" that lives in the background and handles all requests. We instead want to launch an individual nginx process and pass it local configuration. To accomplish this the config file needs `daemon off;` which will allow us to boot the server syncronously.

### Understanding Nginx: Stream logs to stdout

If you boot nginx with daemon mode off, you won't get much output, that's because NGINX sends logs to disk by default, assuming that a system administrator will SSH into the machine to read the logs when there's a problem. To fix this you can redirect the output to stdout instead:

```
# Print log to STDOUT
error_log /dev/stdout info;
```

This is also how Heroku prefers to collect logs. This guards against the failure mode of the hard drive getting full, and allows applications to horizontally scale to many machines. Usually logs are collected and made searchable via a logging addon.

### Understanding Nginx: Absolute paths

Nginx is not intended to be booted many times on a machine, however that's what we want. It also means that it's expecting to be configured using full paths. So, calling `$ nginx -c ./nginx.conf` won't work:

```
$ nginx -c bin/tmpenvsub ./my-nginx.conf
nginx: [emerg] open() "/opt/homebrew/Cellar/nginx-full/1.27.0/./nginx.template.conf" failed (2: No such file or directory)
```

To fix this problem, you need to run it with the full path like:

```
$ nginx -c "$PWD/./my-nginx.conf"
2025/04/21 14:31:16 [notice] 68534#0: using the "kqueue" event method
2025/04/21 14:31:16 [notice] 68534#0: nginx/1.27.0
```

### Understanding Nginx: Env vars

Nginx also wasn't intended to be configured with environment variables. Their documentation recommends using `envsubst` and a template to build your own substitution framework. You can see the code in [`bin/server`](bin/server) that reads in the contents of `nginx.template.conf`, applies env vars to it, then boots the resulting file.

We need this to be able to run on Heroku where the `PORT` of a web dyno is not static. This is used in the `listen` directive of our template:

```conf
# ...
http {
    server {
        listen ${PORT};
        listen [::]:${PORT};
        # ...
```

### Understanding Nginx: Temp files and `/var/lib/nginx`

Nginx defaults to writing logs and cache artifacts in the `/var/lib/nginx` directory, which is not writeable for us.

Buildpacks such as the dedicated [Paketo Nginx buildapack]() work around this limitation by using a [compiled from source](https://github.com/paketo-buildpacks/nginx/blob/1d7c46aef0cb885d3a5716dd0c0015fc81064aa9/dependency/actions/compile/entrypoint#L84-L85) version of Nginx with custom flags. But it's not 100% necessary. This project works around these disk access issues by setting configuration in the `nginx.template.conf` to point files to `/tmp` which is writeable to us.


### Understanding Nginx: Signals

For whatever reason, Nginx decided that SIGTERM means "stop immediately" rather than "cleanup and stop soon." They use SIGQUIT for "graceful shutdown." Even though [GNU docs](https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html) describe SIGTERM as:

> It is the normal way to politely ask a program to terminate.

On Heroku, when a dyno needs to shutdown [it will first send SIGTERM](https://devcenter.heroku.com/articles/dyno-shutdown-behavior) and if it doesn't exit within a timeout, it will send a SIGKILL. To fix this little misunderstanding, the `bin/server` script also traps SIGTERM and changes it to SIGQUIT for the benefit of our little server friend. It's not strictly necessarry, but it means that any in-flight requests will be given a chance to finish rahter than simply stopping and returning an error to all clients.

## Debugging Config modifications locally

If you run into problems while modifying the configuration you can debug it via the `nginx -t` command locally.

```term
$ bin/server
...
Running $ nginx -c /var/folders/yr/yytf3z3n3

nginx: [emerg] unknown directive "oops-i-am-not-a-command" in
```

Using the same file name above you can run:

```term
$ nginx -c /var/folders/yr/yytf3z3n3 -t
```

## Why "Deh" and not "Deeb"?

In [pronouncing Debian](https://www.debian.org/doc/manuals/project-history/intro.en.html#pronouncing-debian) they point out that Ian Murdock came up with the name "Debian" by combining his wife "Debra" (Deb) with his name "Ian."
