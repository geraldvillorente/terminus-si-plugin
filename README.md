# Terminus Secure Integration Test Plugin

[![Unsupported](https://img.shields.io/badge/Pantheon-Unsupported-yellow?logo=pantheon&color=FFDC28)](https://pantheon.io/docs/oss-support-levels#unsupported)

Terminus plugin that allows for testing [Pantheon Secure Integration (SI)](https://docs.pantheon.io/guides/secure-development/secure-integration/) configurations.

Adds the following commands to Terminus:

* `si:constants:list` (alias: `silist`) - outputs a list of all SI constants configured for the site, along with the IP and port to which the SI resolves.
* `si:showcerts` (alias: `sicerts`) - runs an OpenSSL handshake to inspect the remote server certificates (can be used as a proxy for any kind of SI test).
* `si:test:curl` (alias: `sitcurl`) - runs a test that determines whether cURL functionality is working as expected.
* `si:test:ldap` (alias: `sildap`) - runs a test that determines whether LDAP functionality is working as expected.
* `si:test:smtp` (alias: `sismtp`) - runs a test that determines whether SMTP functionality is working as expected.
* `si:test:ssh` (alias: `sissh`) - runs a test that determines whether SSH functionality is working as expected.

## Details

**What does this plugin do?** This plugin was written to provide a baseline function check of some common SI setups, verifying that requests are sent to `localhost:<SI port>` and forwarded to the client's remote service. If the client's service is configured properly and is capable of replying, that reply will be displayed. If the client reports that their SI is not working correctly, this plugin provides a way to troubleshoot whether there is a problem with the SI setup itself or if the module configuration is incorrect (for example, if the client reports that they cannot send mail via the stunnel and yet the `si:test:smtp` command indicates success, the problem is likely in the module configuration, not the SI setup).

**What doesn't this plugin do?** This plugin does not indicate whether a module or plugin is configured properly, nor does it guarantee that the client's server is set up properly.

**Why might this plugin say that the SI is not configured properly?**

There are a few reasons why this plugin may report that the SI test failed:

* The client's server has not yet been configured to allow Pantheon's F5 load balancer IPs.
* The service running on the client's server has not yet been set up or is not responding for some reason.
* The SI has not yet been activated.
* The SI is pointing to the wrong IP and port, which could be because the client sent the wrong information or because the engineer set up the SI incorrectly.

This plugin does not attempt to diagnose _why_ a connection failed; it only reports if it _did_.

## Usage

### `si:constants:list`

Example command: `terminus si:constants:list my-site.dev`

This command simply checks which SI constants are configured for a given environment. The output will indicate the PHP constant name to use, as well as the destination IP and port that the customer requested when setting up the SI. **NOTE**: the port listed is the DESTINATION port, not the SOURCE port.

### `si:showcerts`

Example command: `terminus si:showcerts my-site.dev --constant-name=PANTHEON_SOIP_CONSTANT_NAME`

Options:

| Option          | Description                                                         |
|-----------------|---------------------------------------------------------------------|
| --constant-name | The name of the constant to use. Must start with `PANTHEON_SOIP_`.  |

This command runs an OpenSSL command on the remote server which performs an SSL handshake and returns the certificates from the remote server. This command can be used as a very basic check on whether or not the appserver is communicating with the remote service. This command may still fail, even if the SI is set up properly, depending on the client's server configuration, but it is a good first pass to determine if the SI is functioning properly. The output will be the complete output from the `openssl` command.

### `si:test:curl`

Example command: `terminus si:test:curl my-site.dev --constant-name=PANTHEON_SOIP_CONSTANT_NAME --url=http://www.example.com/test.json`

Options:

| Option          | Description                                                         |
|-----------------|---------------------------------------------------------------------|
| --constant-name | The name of the constant to use. Must start with `PANTHEON_SOIP_`.  |
| --url           | The URL to retrieve via the SI.                                    |

This command uses cURL to assess whether basic cURL communication is working between the appserver and the client's remote service. Note that this test works for a variety of protocols, not just HTTP. Since PHP uses cURL as a backend for services like SOAP, and because many Drupal and WordPress modules/plugins use cURL for services like Apache Solr, this plugin can test a variety of scenarios where data is retrieved from a remote web server. The output from this command will indicate success or failure. For additional debugging information (which includes the content retrieved from the `--url` and the elapsed time of the test), use Terminus's `-vv` flag.

### `si:test:ldap`

Example command: `terminus si:test:ldap mysite.dev --constant-name=PANTHEON_SOIP_CONSTANT_NAME --use-tls=true --bind-dn="cn=admin,dc=example,dc=org" --bind-password`

Options:

| Option          | Description                                                                                                                                                                              |
|--------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| --constant-name    | The name of the constant to use. Must start with `PANTHEON_SOIP_`.                                                                                                                       |
| --use-tls          | If `TRUE`, opens a connection using `ldaps://`; if `FALSE`, opens a connection using `ldap://`. Default: `TRUE`                                                                          |
| --proto            | The [LDAP protocol](http://php.net/manual/en/function.ldap-set-option.php) to use with the connection. Can be `2` or `3` (default).                                                     |
| --bind-dn          | The bind DN to use when connecting to the LDAP server. Leave blank to perform an anonymous binding                                                                                       |
| --bind-password    | The bind password to use when connecting to the LDAP server. Leave blank to be prompted to enter the password on the command line, or specify the password directly on the command line. |
| --bypass-tls-check | Bypass the PHP TLS check of the remote certificate. This option has security considerations when running in a live environment but is useful for debugging intermittent connection problems. Default: `FALSE` |

This command uses PHP's built-in [LDAP functions](http://php.net/manual/en/book.ldap.php) to communicate with a client's LDAP server and perform a basic binding. The output from this command will indicate success or failure. For additional debugging information (which includes a status message indicating the test parameters and results, and the elapsed time of the test), use Terminus's `-vv` flag.

### `si:test:smtp`

Example command: `terminus si:test:smtp my-site.dev --constant-name=PANTHEON_SOIP_CONSTANT_NAME --relay-address=client.smtp.server.com`

Options:

| Option          | Description                                                                          |
|-----------------|--------------------------------------------------------------------------------------|
| --constant-name | The name of the constant to use. Must start with `PANTHEON_SOIP_`.                   |
| --relay-address | The address of the client's SMTP server (or optionally any other SMTP relay server). |

This command uses web sockets to connect to the client's SMTP server and issue a `HELO` command to determine whether or not communication is flowing to the client's SMTP server. The output from this command will indicate success or failure. For additional debugging information (which includes the output from the `HELO` command and the elapsed time of the test), use Terminus's `-vv` flag.

### `si:test:ssh`

Example command: `terminus SI:test:ssh my-site.dev --constant-name=PANTHEON_SOIP_CONSTANT_NAME

Options:

| Option          | Description                                                        |
|-----------------|--------------------------------------------------------------------|
| --constant-name | The name of the constant to use. Must start with `PANTHEON_SOIP_`. |

This command uses web sockets to connect to a remote server and checks to see if it is an SSH server. (Currently the mechanism is that it looks for `ssh` in the first 2048 bytes of the server response when connecting, as OpenSSH servers identify themselves as such upon a successful connection. PRs for other cases are welcome.) The output from this command will indicate success or failure. For additional debugging information (which includes a status message consisting of the header information returned), use Terminus's `-vv` flag.

## Installation
For help installing, see [Manage Plugins](https://pantheon.io/docs/terminus/plugins/)

```
mkdir -p ~/.terminus/plugins
composer create-project -d ~/.terminus/plugins geraldvillorente/terminus-si-plugin:dev-master
```

For Terminus 3:

```
terminus self:plugin:install geraldvillorente/terminus-si-plugin
```

## Help

Run `terminus list si` for a complete list of available commands. Use `terminus help <command>` to get help on any individual command.
