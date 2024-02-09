# Vault Artifactory Secrets Plugin

This plugin is actively maintained by JFrog Inc. Please refer to [CONTRIBUTING.md](CONTRIBUTING.md) for contributions and create GitHub issues to ask for feature requests and support.

Contact [JFrog Support](https://jfrog.com/support/) for urgent, time sensitive issues.

----------------------------------------------------------------

This is a [HashiCorp Vault](https://www.vaultproject.io/) plugin which talks to JFrog Artifactory server and will
dynamically provision access tokens with specified scopes. This backend can be mounted multiple times
to provide access to multiple Artifactory servers.

Using this plugin, you can limit the accidental exposure window of Artifactory tokens; useful for continuous integration servers.

## Access Token Creation and Revoking

This backend creates access tokens in Artifactory using the admin credentials provided. Note that if you provide non-administrative credentials, then the "username" must match the username of the credential owner.

### Admin Token Expiration Notice

> Prior to Artifactory 7.42.1, admin access token was created with the system token expiration (default to 1 year) even when `expires_in` API field is set to `0`. In 7.42.1, admin token expiration no longer constrained by system configuration and therefore can be set to non-expiring.
> See section ["Generate a Non-expiry Admin Token without Changing the Configuration"](https://www.jfrog.com/confluence/display/JFROG/Artifactory+Release+Notes#ArtifactoryReleaseNotes-Artifactory7.42.1Cloud) in the release note.
>
> Therefore if you created access token(s) with Artifactory prior to 7.42.1, the tokens will have a 1 year expiration time (or whatever value is set in the Artifactory configuration) and will become unusable silently when it expires.
>
> We suggest upgrading your Artifactory to 7.42.1 or later (if possible) and rotate your tokens to get new, non-expiring tokens. Or set reminders to ensure you rotate your tokens before expiration.
>
> It should also be noted that some "scripts" used to create an admin token may default to expiration in `1h`, so it is best to **rotate** the admin token immediately, to ensure it doesn't expire unexpectedly.
>
> If you are using v0.2.9 or later, you can check if your admin token has an expiration using `vault read artifactory/config/admin`. If the exp/expires fields are not present, your token has no expiration set.

### Dynamic Usernames

Previous versions of this plugin required a static `username` associated to the roles. This is still supported for backwards compatibility, but you can now use a dynamically generated username, based on [Vault Username Templates][vault-username-templating]. The generated tokens will be associated to a username generated from the template `v-{{.RoleName}}-{{Random 8}})` (`v-jenkins-x4mohTA8`), by default. You can change this template by specifying a `username_template=` option to the `/artifactory/config/admin` endpoint. The "scope" in the role should be `applied-permissions/groups:(list-of-groups)`, since `applied-permissions/user` would require the username to exist ahead of time. The user will not show in the Users list, but will be dynamically created during the scope of the token. The username still needs to be compliant with [artifactory requirements][artifactory-create-token] (less than 255 characters). It will be converted to lowercase by the API.

Example:

```sh
vault write artifactory/config/admin username_template="v_{{.DisplayName}}_{{.RoleName}}_{{random 10}}_{{unix_time}}"
```

### Expiring Tokens

By default, the Vault generated Artifactory tokens will not show an expiration date, which means that Artifactory will not
automatically revoke them. Vault will revoke the token when its lease expires due to logout or timeout (ttl/max_ttl). The reason
for this is because of the [default Revocable/Persistency Thresholds][artifactory-token-thresholds] in Artifactory. If you would
like the artifactory token itself to show an expiration, and you are using Artifactory v7.50.3 or higher, you can write
`use_expiring_tokens=true` to the `/artifactory/config/admin` endpoint. This will set the `force_revocable=true` parameter and
set `expires_in` to either max lease TTL or role max_ttl, whichever is lower, when a token is created, overriding the default
thresholds mentioned above.

Example:

```sh
vault write artifactory/config/admin use_expiring_tokens=true
```

Example Token Output:

```console
$ ACCESS_TOKEN=$(vault read -field access_token artifactory/token/test)
$ jwt decode $ACCESS_TOKEN

Token header
------------
{
"typ": "JWT",
"alg": "RS256",
"kid": "nxB2_1jNkYS5oYsl6nbUaaeALfKpfBZUyP0SW3txYUM"
}

Token claims
------------
{
"aud": "*@*",
"exp": 1678913614,
"ext": "{\"revocable\":\"true\"}",
"iat": 1678902814,
"iss": "jfac@01gvgpzpv8jytn0fvq41wb1srj",
"jti": "e39cec86-069c-4b75-8897-c2bf05dc8354",
"scp": "applied-permissions/groups:readers",
"sub": "jfac@01gvgpzpv8jytn0fvq41wb1srj/userv-test-p9nprfwr"
}
```

### Artifactory Version Detection

Some of the functionality of this plugin requires certain versions of Artifactory. For example, as of Artifactory 7.50.3, we can optionally set the `force_revocable` flag and set the expiration of the token to `max_ttl`.
If you have upgraded Artifactory after installing this plugin, and would like to take advantage of newer features, you can issue an empty write to the `artifactory/config/admin` endpoint to re-detect the version, or it will re-detect upon reload.

Example:

```sh
vault write -f artifactory/config/admin
```

## Installation

### Using pre-built releases

You can find pre-built releases of the plugin [here][artreleases] and download the latest binary file corresponding to your target OS.

### From Sources

If you prefer to build the plugin from sources, clone the GitHub repository locally and run the command `make build` from the root of the sources directory.

See [Local Development Prerequisites](#local-development-prerequisites) section for pre-requisites.

Upon successful compilation, the resulting `artifactory-secrets-plugin` binary is stored in the `dist/vault-plugin-secrets-artifactory_<OS architecture>` directory.

## Configuration

Copy the plugin binary into a location of your choice; this directory must be specified as the [`plugin_directory`][vaultdocplugindir] in the Vault configuration file:

```hcl
plugin_directory = "path/to/plugin/directory"
```

Start a Vault server with this configuration file:

```sh
vault server -config=path/to/vault/config.hcl
```

Once the server is started, register the plugin in the Vault server's [plugin catalog][vaultdocplugincatalog]:

```sh
vault plugin register \
  -sha256=$(sha256sum path/to/plugin/directory/artifactory | cut -d " " -f 1) \
  -command=artifactory-secrets-plugin \
  secret artifactory
```

> **Note**
> you may need to also add arguments to the registration like `-args="-ca-cert ca.pem` or something insecure like: `-args="-tls-skip-verify"` depending on your environment. (see `./path/to/plugins/artifactory -help` for all the options)

> **Note**
> This inline checksum calculation above is provided for illustration purpose and does not validate your binary. It should **not** be used for production environment. Instead you should use the checksum provided as [part of the release](https://github.com/jfrog/vault-plugin-secrets-artifactory/releases). See [How to verify binary checksums](#how-to-verify-binary-checksums) section.

You can now enable the Artifactory secrets plugin:

```sh
vault secrets enable artifactory
```

### How to verify binary checksums

Checksums for each binary are provided in the `artifactory-secrets-plugin_<version>_checksums.txt` file. It is signed with the public key `vault-plugin-secrets-artifactory-public-key.asc` which creates the signature file `artifactory-secrets-plugin_<version>_checksums.txt.sig`.

If the public key is not in your GPG keychain, import it:
```sh
gpg --import artifactory-secrets-plugin-public-key.asc
```

Then verify the checksums file signature:

```sh
gpg --verify artifactory-secrets-plugin_<version>_checksums.txt.sig
```

You should see something like the following:
```sh
gpg: assuming signed data in 'artifactory-secrets-plugin_0.2.17_checksums.txt'
gpg: Signature made Mon May  8 14:22:12 2023 PDT
gpg:                using RSA key ED4FF1CD6C2318B470A33A1659FE1520A4A355CD
gpg: Good signature from "Alex Hung <alexh@jfrog.com>" [ultimate]
```

With the checksums file verified, you can now safely use the SHA256 checkum inside as part of the Vault plugin registration (vs calling `sha256sum`).

### Artifactory

1. Log into the Artifactory UI as an "admin".
1. Create the Access Token that Vault will use to interact with Artifactory. In Artifactory 7.x this can be done in the UI Administration (gear) -> User Management -> Access Tokens -> Generate Token.
    * Token Type: `Scoped Token`
    * Description: (optional) `vault-plugin-secrets-artifactory` (NOTE: This will be lost on admin token rotation, because it is not part of the token)
    * Token Scope: `Admin` **(IMPORTANT)**
    * User name: `vault-admin` (for example)
    * Service: `Artifactory` (or you can leave it on "All")
    * Expiration time: `Never` (do not set the expiration time less than `7h`, since by default, it will not be revocable once the expiration is less than 6h)
1. Save the generated token as the environment variable `TOKEN`

Alternatives:

* Use the [CreateToken REST API][artifactory-create-token], and save the `access_token` from the JSON response as the environment variable `TOKEN`.
* Use [`getArtifactoryAdminToken.sh`](./scripts/getArtifactoryAdminToken.sh).

    ```sh
    export JFROG_URL=https://artifactory.example.org
    export ARTIFACTORY_USERNAME=admin
    export ARTIFACTORY_PASSWORD=password
    TOKEN=$(scripts/getArtifactoryAdminToken.sh)
    ```

### Vault

```sh
vault write artifactory/config/admin \
    url=https://artifactory.example.org \
    access_token=$TOKEN
```

**OPTIONAL**, but recommended: Rotate the admin token, so that only Vault knows it.

```sh
vault write -f artifactory/config/rotate
```

**NOTE** some versions of artifactory (notably `7.39.10`) fail to rotate correctly. As noted above, we recommend being on `7.42.1` or higher. The token was indeed rotated, but as the error indicates, the old token could not be revoked.

**ALSO** If you want to change the username for the admin token (tired of it just being "admin"?) or set a "Description" on the token, those parameters are optionally
available on the `artifactory/config/rotate` endpoint.

```sh
vault write artifactory/config/rotate username="new-username" description="A token used by vault-secrets-engine on our vault server"`
```

#### Bypass TLS connection verification with Artifactory

To bypass TLS connection verification with Artifactory, set `bypass_artifactory_tls_verification` to `true`, e.g.

```sh
vault write artifactory/config/admin \
    url=https://artifactory.example.org \
    access_token=$TOKEN \
    bypass_artifactory_tls_verification=true
```

OPTIONAL: Check the results:

```sh
vault read artifactory/config/admin
```

Example output:

```console
Key                                 Value
---                                 -----
access_token_sha256                 74834a86b2082750201e2a1e520f21f7bfc7d4026e5bd2b075ca2d0699b7c4e3
bypass_artifactory_tls_verification false
scope                               applied-permissions/admin
token_id                            db0002b0-af08-486c-bbad-b255a3cc7b31
url                                 http://localhost:8082
use_expiring_tokens                 false
username                            vault-admin
version                             7.55.6
```

## Usage

Create a role (scope for artifactory >= 7.21.1)

```sh
vault write artifactory/roles/jenkins \
    scope="applied-permissions/groups:automation " \
    default_ttl=1h max_ttl=3h
```

Also supports `grant_type=[Optional, default: "client_credentials"]`, and `audience=[Optional, default: *@*]` see [JFrog documentation][artifactory-create-token].

NOTE: By default, the username will be generated automatically using the template `v-(RoleName)-(random 8)` (i.e. `v-jenkins-x4mohTA8`). If you would prefer to have a static username (the same for every token), you can set `username=whatever-you-want`, but keep in mind that in a dynamic environment, someone or something using an old, expired token might cause a denial of service (too many failed logins) against users with the correct token.

<details>
<summary>CLICK for: Create a Role (scope for artifactory < 7.21.1)</summary>

```sh
vault write artifactory/roles/jenkins \
    username="example-service-jenkins" \
    scope="api:* member-of-groups:ci-server" \
    default_ttl=1h max_ttl=3h
```

</details>

> **Note**
> There are some changes in the **scopes** supported in artifactory request >7.21. Please refer to the JFrog documentation for the same according to the artifactory version.

```sh
vault list artifactory/roles
```

Example Output:

```console

Keys
----
jenkins
```

```sh
vault read artifactory/token/jenkins
```

Example output (token truncated):

```console
Key                Value
---                -----
lease_id           artifactory/token/jenkins/9hHxV1NlyLzPgmNIzjssRCa9
lease_duration     1h
lease_renewable    true
access_token       eyJ2ZXIiOiIyIiw....
role               jenkins
scope              applied-permissions/groups:automation
token_id           06d962b2-63e2-4279-a25d-d2a9cab6507f
username           v-jenkins-x4mohTA8
```

### Scoped Access Tokens

In order to decouple Artifactory Group maintenance from Vault plugin configuration, you can configure a single role to request Access Tokens for specific groups. This option should be used with extreme care to ensure that your Vault policies are restricting which groups it can request tokens on behalf of.

Create a role (scope for artifactory >= 7.21.1)

```sh
vault write artifactory/roles/jenkins \
    username="jenkins-vault"
    scope="applied-permissions/groups:admin" \
    default_ttl=1h max_ttl=3h
```

Request Access Token for test-group

```sh
vault read artifactory/token/jenkins scope=applied-permissions/groups:test-group
```

Example output (token truncated):

```console
Key                Value
---                -----
lease_id           artifactory/token/jenkins/9hHxV1NlyLzPgmNIzjssRCa9
lease_duration     1h
lease_renewable    true
access_token       eyJ2ZXIiOiIyIiw....
role               jenkins
scope              applied-permissions/groups:test-group
token_id           06d962b2-63e2-4279-a25d-d2a9cab6507f
username           v-jenkins-b0ftbTAG
```

Example Vault Policy

```console
path "artifactory/token/jenkins" {
  capabilities = ["read"],
  required_parameters = ["scope"],
  allowed_parameters = {
    "scope" = ["applied-permissions/groups:test-group"]
  }
  denied_parameters = {
    "scope" = ["applied-permissions/groups:admin"]
  }
}
```

### User Token Path

User tokens may be obtained from the `/artifactory/user_token/<user-name>` endpoint. This is useful in conjunction with [ACL Policy Path Templating](https://developer.hashicorp.com/vault/tutorials/policies/policy-templating) to allow users authenticated to Vault to obtain API tokens in Artfactory for their own account. Be careful to ensure that Vault authentication methods & policies align with user account names in Artifactory. For example the following policy allows users authenticated to the `azure-ad-oidc` authentication mount to obtain a token for Artifactory for themselves, assuming the `upn` metadata is populated in Vault during authentication.

```
path "artifactory/user_token/{{identity.entity.aliases.azure-ad-oidc.metadata.upn}}" {
  capabilities = [ "read" ]
}
```

Default values for the token's description, ttl, max_ttl and audience may be configured at the `/artifactory/config/admin` endpoint. TTL rules follow Vault's [general cases](https://developer.hashicorp.com/vault/docs/concepts/tokens#the-general-case) and [token hierarchy](https://developer.hashicorp.com/vault/docs/concepts/tokens#token-hierarchies-and-orphan-tokens). The desired lease TTL will be determined by the most specific TTL value specified with the request ttl parameter being highest precedence, followed by the plugin configuration, secret mount tuning, or system default ttl. The maximum TTL value allowed is limited to the lowest value of the max_ttl setting set on the system, secret mount tuning, plugin configuration, or the specific request.

Example Token Configuration:

```sh
vault write artifactory/config/user_token default_description="Generated by Vault" max_ttl=604800 default_ttl=86400
```

```console
$ vault read artifactory/config/user_token
Key                    Value
---                    -----
audience               n/a
default_description    Generated by Vault
default_ttl            24h
max_ttl                168h
scope                  applied-permissions/admin
token_id               8df5dd21-31ae-4062-bbe5-580a607f5645
username               vault-admin
```

Example Usage:
```console
$ vault read artifactory/user_token/admin description="Dev Desktop"
Key                Value
---                -----
lease_id           artifactory/user_token/admin/4UhTThCwctPGX0TYXeoyoVEt
lease_duration     24h
lease_renewable    true
access_token       eyJ2Z424242424.....
description        Dev Desktop
scope              applied-permissions/user
token_id           3c6b2e63-87dc-4d26-9698-ffdfb282a6ee
username           admin
```

## Development

### Local Development Prerequisites

* Vault
  * <https://developer.hashicorp.com/vault/docs/install>
  * `brew install vault`
* Golang
  * <https://go.dev/doc/install>
  * `brew install golang`
* GoReleaser - Used during the build process
  *  <https://goreleaser.com/install/>
  * `brew install goreleaser`
* Docker - To run a test Artifactory instance (very useful for testing)
  * <https://docs.docker.com/get-docker/>
  * `brew install docker --cask`

### Testing Locally

If you're compiling this yourself and want to test locally, you will need a working docker environment. You will also need vault and golang installed, then you can follow the steps below.

* In first terminal, build the plugin and start the local dev server:

```sh
make
```

* In another terminal, setup a test artifactory instance.

```sh
make artifactory
```

* In the same terminal, setup artifactory-secrets-engine in vault with values:

```sh
export VAULT_ADDR=http://localhost:8200
export VAULT_TOKEN=root
make setup
```

NOTE: Each time you rebuild (`make`), vault will restart, so you will need to run `make setup` again, since vault is in dev mode.

* Once you are done testing, you can destroy the local artifactory instance:

```sh
make stop_artifactory
```

### Other Local Development Details

This section is informational, and is not intended as a step-by-step. If you really want the gory details, checkout [the `Makefile`](./Makefile)

#### Install Vault binary

* You can follow the [Installing Vault](https://developer.hashicorp.com/vault/docs/install) instructions.
* Alternatively, if you are on MacOS, and have HomeBrew, you can use that:

```sh
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

----------------------------------------------------------------

#### Start Vault dev server

```sh
make start
```

----------------------------------------------------------------

#### Export Vault url and token

```sh
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN=root
```

----------------------------------------------------------------

#### Build plugin binary

```sh
make build
```

----------------------------------------------------------------

#### Upgrade plugin binary

To build and upgrade the plugin without having to reconfigure it...

```sh
make upgrade
```

----------------------------------------------------------------

#### Create Test Artifactory

```sh
make artifactory
```

Set `ARTIFACTORY_VERSION` to a [specific self hosted version][artifactory-release-notes] to override the default.

Example:

```sh
make artifactory ARTIFACTORY_VERSION=7.49.10
```

NOTE: If you get a message like:

```console
make: Nothing to be done for `artifactory'.
```

This simply means that "make" thinks artifactory is already running due to the existence of the `./vault/artifactory.env` file.
If you want to run a different version, first use `make stop_artifactory`. If you stopped artifactory using other means (docker), then `rm vault/artifactory.env` manually.

----------------------------------------------------------------

#### Register artifactory-secrets plugin with Vault server

If you didn't run `make upgrade` (i.e. just `make build`), then you need to register the newly built plugin with the Vault server.

```sh
make register
```

----------------------------------------------------------------

#### Enable artifactory-secrets plugin

```sh
make enable
```

----------------------------------------------------------------

#### Disable plugin (unmount from vault)

```sh
make disable
```

NOTE: This is a good idea before stopping artifactory, especially if you plan to change versions of artifactory. Alternatively, just exit vault (Ctrl+c), and it will go back to default state.

----------------------------------------------------------------

#### Get ADMIN Artifactory token and write it to vault

```sh
make admin
```

NOTE: This following might be some useful environment variables:

> * `JFROG_URL`
> * `ARTIFACTORY_USERNAME`
> * `ARTIFACTORY_PASSWORD`

For example:

```sh
JFROG_URL=https://artifactory.example.org ARTIFACTORY_USERNAME=tommy ARTIFACTORY_PASSWORD='SuperSecret' make admin
```

If you already have a JFROG_ACCESS_TOKEN, you can skip straight to that too:

```sh
export JFROG_URL=https://artifactory.example.com
export JFROG_ACCESS_TOKEN=(PASTE YOUR JFROG ADMIN TOKEN)
make admin
```

----------------------------------------------------------------

* Setup a "test" role, bound to the "readers" group

```sh
make testrole
```

----------------------------------------------------------------

#### Run Acceptance Tests

```sh
make acceptance
```

This requires the following:
* A running Artifactory instance
* Env vars `JFROG_URL` and `JFROG_ACCESS_TOKEN` for the running Artifactory instance be set

## Issues

* RTFACT-22477 - proposing CIDR restrictions on the created access tokens.
* () - Artifactory 7.39.10 fails to revoke previous token during rotation. Recommend 7.42.1. or higher.

## Contributors

See the [contribution guide](./CONTRIBUTING.md).

## License

Copyright (c) 2023 JFrog.

Apache 2.0 licensed, see [LICENSE][LICENSE] file.

[LICENSE]: ./LICENSE
[artreleases]: https://github.com/jfrog/vault-plugin-secrets-artifactory/releases
[vaultdocplugindir]: https://www.vaultproject.io/docs/configuration/index.html#plugin_directory
[vaultdocplugincatalog]: https://www.vaultproject.io/docs/internals/plugins.html#plugin-catalog
[artifactory-create-token]: https://www.jfrog.com/confluence/display/JFROG/JFrog+Platform+REST+API#JFrogPlatformRESTAPI-CreateToken
[vault-username-templating]: https://developer.hashicorp.com/vault/docs/concepts/username-templating
[artifactory-release-notes]: https://www.jfrog.com/confluence/display/JFROG/Artifactory+Release+Notes
[artifactory-token-thresholds]: https://www.jfrog.com/confluence/display/JFROG/Access+Tokens#AccessTokens-UsingtheRevocableandPersistencyThresholds
