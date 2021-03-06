# Nginx Configuration

Dokku uses nginx as its server for routing requests to specific applications. By default, access and error logs are written for each app to `/var/log/nginx/${APP}-access.log` and `/var/log/nginx/${APP}-error.log` respectively

```
nginx:access-logs <app> [-t]             # Show the nginx access logs for an application (-t follows)
nginx:build-config <app>                 # (Re)builds nginx config for given app
nginx:error-logs <app> [-t]              # Show the nginx error logs for an application (-t follows)
nginx:show-conf <app>                    # Display app nginx config
nginx:validate [<app>] [--clean]         # Validates and optionally cleans up invalid nginx configurations
```

## Checking access logs

You may check nginx access logs via the `nginx:access-logs` command. This assumes that app access logs are being stored in `/var/log/nginx/$APP-access.log`, as is the default in the generated `nginx.conf`.

```shell
dokku nginx:access-logs node-js-app
```

You may also follow the logs by specifying the `-t` flag.

```shell
dokku nginx:access-logs node-js-app -t
```

## Checking error logs

You may check nginx error logs via the `nginx:access-logs` command. This assumes that app error logs are being stored in `/var/log/nginx/$APP-error.log`, as is the default in the generated `nginx.conf`.

```shell
dokku nginx:error-logs node-js-app
```

You may also follow the logs by specifying the `-t` flag.

```shell
dokku nginx:error-logs node-js-app -t
```

## Regenerating nginx config

In certain cases, your app nginx configs may drift from the correct config for your app. You may regenerate the config at any point via the `nginx:build-config` command. This may fail if there are no current web listeners for your app.

```shell
dokku nginx:build-config node-js-app
```

## Showing the nginx config

For debugging purposes, it may be useful to show the nginx config. This can be achieved via the `nginx:show-conf` command.

```shell
dokku nginx:show-conf node-js-app
```

## Validating nginx configs

It may be desired to validate an nginx config outside of the deployment process. To do so, run the `nginx:validate` command. With no arguments, this will validate all app nginx configs, one at a time. A minimal wrapper nginx config is generated for each app's nginx config, upon which `nginx -t` will be run.

```shell
dokku nginx:validate
```

As app nginx configs are actually executed within a shared context, it is possible for an individual config to be invalid when being validated standalone but _also_ be valid within the global server context. As such, the exit code for the `nginx:validate` command is the exit code of `nginx -t` against the server's real nginx config.

The `nginx:validate` command also takes an optional `--clean` flag. If specified, invalid nginx configs will be removed.

> Warning: Invalid app nginx config's will be removed _even if_ the config is valid in the global server context.

```shell
dokku nginx:validate --clean
```

The `--clean` flag may also be specified for a given app:

```shell
dokku nginx:validate node-js-app --clean
```

## Customizing the nginx configuration

> New as of 0.5.0

Dokku uses a templating library by the name of [sigil](https://github.com/gliderlabs/sigil) to generate nginx configuration for each app. You may also provide a custom template for your application as follows:

- Copy the following example template to a file named `nginx.conf.sigil` and either:
  - If using a buildpack application, you __must__ check it into the root of your app repo.
  - `ADD` it to your dockerfile `WORKDIR`
  - if your dockerfile has no `WORKDIR`, `ADD` it to the `/app` folder

> When using a custom `nginx.conf.sigil` file, depending upon your application configuration, you *may* be exposing the file externally. As this file is extracted before the container is run, you can, safely delete it in a custom `entrypoint.sh` configured in a Dockerfile `ENTRYPOINT`.

> The default template is available [here](https://github.com/dokku/dokku/blob/master/plugins/nginx-vhosts/templates/nginx.conf.sigil), and can be used as a guide for your own, custom `nginx.conf.sigil` file. Please refer to the appropriate template file version for your Dokku version.

### Available template variables

```
{{ .APP }}                          Application name
{{ .APP_SSL_PATH }}                 Path to SSL certificate and key
{{ .DOKKU_ROOT }}                   Global Dokku root directory (ex: app dir would be `{{ .DOKKU_ROOT }}/{{ .APP }}`)
{{ .DOKKU_APP_LISTENERS }}          List of IP:PORT pairs of app containers
{{ .PROXY_PORT }}                   Non-SSL nginx listener port (same as `DOKKU_PROXY_PORT` config var)
{{ .PROXY_SSL_PORT }}               SSL nginx listener port (same as `DOKKU_PROXY_SSL_PORT` config var)
{{ .NOSSL_SERVER_NAME }}            List of non-SSL VHOSTS
{{ .PROXY_PORT_MAP }}               List of port mappings (same as `DOKKU_PROXY_PORT_MAP` config var)
{{ .PROXY_UPSTREAM_PORTS }}         List of configured upstream ports (derived from `DOKKU_PROXY_PORT_MAP` config var)
{{ .RAW_TCP_PORTS }}                List of exposed tcp ports as defined by Dockerfile `EXPOSE` directive (**Dockerfile apps only**)
{{ .SSL_INUSE }}                    Boolean set when an app is SSL-enabled
{{ .SSL_SERVER_NAME }}              List of SSL VHOSTS
```

> Note: Application config variables are available for use in custom templates. To do so, use the form of `{{ var "FOO" }}` to access a variable named `FOO`.

### Customizing via configuration files included by the default templates

The default nginx.conf template will include everything from your apps `nginx.conf.d/` subdirectory in the main `server {}` block (see above):

```
include {{ .DOKKU_ROOT }}/{{ .APP }}/nginx.conf.d/*.conf;
```

That means you can put additional configuration in separate files, for example to limit the uploaded body size to 50 megabytes, do

```shell
mkdir /home/dokku/node-js-app/nginx.conf.d/
echo 'client_max_body_size 50m;' > /home/dokku/node-js-app/nginx.conf.d/upload.conf
chown dokku:dokku /home/dokku/node-js-app/nginx.conf.d/upload.conf
service nginx reload
```

The example above uses additional configuration files directly on the Dokku host. Unlike the `nginx.conf.sigil` file, these additional files will not be copied over from your application repo, and thus need to be placed in the `/home/dokku/node-js-app/nginx.conf.d/` directory manually.

For PHP Buildpack users, you will also need to provide a `Procfile` and an accompanying `nginx.conf` file to customize the nginx config *within* the container. The following are example contents for your `Procfile`

    web: vendor/bin/heroku-php-nginx -C nginx.conf -i php.ini php/
    
Your `nginx.conf` file - not to be confused with Dokku's `nginx.conf.sigil` - would also need to be configured as shown in this example:

    client_max_body_size 50m;
    location / {
        index index.php;
        try_files $uri $uri/ /index.php$is_args$args;
    }

Please adjust the `Procfile` and `nginx.conf` file as appropriate.

## Custom Error Pages

By default, Dokku provides custom error pages for the following three categories of errors:

- 4xx: For all non-404 errors with a 4xx response code.
- 404: For "404 Not Found" errors.
- 5xx: For all 5xx error responses

These are provided as an alternative to the generic Nginx error page, are shared for _all_ applications, and their contents are located on disk at `/var/lib/dokku/data/nginx-vhosts/dokku-errors`. To customize them for a specific app, create a custom `nginx.conf.sigil` as described above and change the paths to point elsewhere.

## Domains plugin

See the [domain configuration documentation](/docs/configuration/domains.md).

## Customizing hostnames

See the [customizing hostnames documentation](/docs/configuration/domains.md#customizing-hostnames).

## Disabling VHOSTS

See the [disabling vhosts documentation](/docs/configuration/domains.md#disabling-vhosts).

## Default site

See the [default site documentation](/docs/configuration/domains.md#default-site).

## Running behind a load balancer

See the [load balancer documentation](/docs/configuration/ssl.md#running-behind-a-load-balancer).

## HSTS Header

See the [HSTS documentation](/docs/configuration/ssl.md#hsts-header).

## SSL Configuration

See the [ssl documentation](/docs/configuration/ssl.md).

## Disabling Nginx

See the [proxy documentation](/docs/advanced-usage/proxy-management.md).

## Managing Proxy Port mappings

See the [proxy documentation](/docs/advanced-usage/proxy-management.md#proxy-port-mapping).
