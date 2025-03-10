![PyPI - License](https://img.shields.io/pypi/l/taiga-contrib-ldap-auth-ext.svg)
[![PyPI - Status](https://img.shields.io/pypi/status/taiga-contrib-ldap-auth-ext.svg)](https://pypi.org/project/taiga-contrib-ldap-auth-ext/)
[![PyPI](https://img.shields.io/pypi/v/taiga-contrib-ldap-auth-ext.svg)](https://pypi.org/project/taiga-contrib-ldap-auth-ext/)
[![PyPI - Downloads](https://img.shields.io/pypi/dm/taiga-contrib-ldap-auth-ext.svg)](https://pypistats.org/packages/taiga-contrib-ldap-auth-ext)
[![Managed with Taiga.io](https://img.shields.io/badge/managed%20with-TAIGA.io-709f14.svg)](https://tree.taiga.io/project/monogrammbot-monogrammtaiga-contrib-ldap-auth-ext/ "Managed with Taiga.io")

# Taiga contrib ldap auth ext

Extended [Taiga.io](https://taiga.io/) plugin for LDAP authentication.

This is a fork of [ensky/taiga-contrib-ldap-auth](https://github.com/ensky/taiga-contrib-ldap-auth), which also retrieves the various contributions and other forks into one.

## :whale: Installation with Docker

If you installed a dockerized Taiga using the 30 Minute Setup approach, you should be able to install this plugin using this guide.

The following will assume that you have a clone of the [kaleidos-ventures/taiga-docker](https://github.com/kaleidos-ventures/taiga-docker) repository on the computer you want to host Taiga on.

### `taiga-back`

1. Edit the `taiga-back` section in the `docker-compose.yml`: Replace `image: taigaio/taiga-back:latest` with `build: ./custom-back`
2. Create a folder `custom-back` next to the `docker-compose.yml` file
3. In this folder, create a file `config.append.py`. Copy the contents of the [`taiga-back` configuration](#taiga-back-configuration) section from this document into it.
4. In this folder, also create a `Dockerfile`. The contents of the file are collapsed below.

If you were to start Taiga now, it would not pull the `taiga-back` directly from Docker Hub but instead build the image from the specified `Dockerfile`. This is exactly what we want, however, do not start Taiga yet – there is still work to be done in `taiga-front`.

#### `custom-back/Dockerfile`

```Dockerfile
FROM taigaio/taiga-back:latest

# Insert custom configuration into the taiga configuration file
COPY config.append.py /taiga-back/settings
RUN cat /taiga-back/settings/config.append.py >> /taiga-back/settings/config.py && rm /taiga-back/settings/config.append.py

RUN pip install taiga-contrib-ldap-auth-ext
```

<details>
<summary>Click here to expand explanation</summary>

The statements in the Dockerfile have the following effect:

1. `FROM ...` bases the image we build on the official `taigaio/taiga-back` image.
2. `COPY ...` and `RUN ...` copy the `config.append.py` file into the container, append it to `/taiga-back/settings/config.py` and then delete it again.
3. `RUN pip install ...` installs this plugin.
</details>

### `taiga-front`

1. Edit the `taiga-front` section in the `docker-compose.yml`. Insert the following below `networks`:

    ```yml
    volumes:
    - ./custom-front/conf.override.json:/usr/share/nginx/html/conf.json
    ```

    There should already be a commented block hinting that you can do this (just with a different path). You can delete this block, or, alternatively, place the file at the path given there and just remove the `# `.

2. Create a folder `custom-front` next to the `docker-compose.yml` file
3. In this folder, create a file `conf.override.json`. The contents of the file are below.

#### `custom-front/conf.override.json`

This file will replace the `conf.json` file. As the `conf.json` is normally automatically generated at runtime from the configuration in your `docker-compose.yml`, this is a bit trickier. Basically, the process boils down to this:

1. Somehow get a valid `conf.json`
2. Create a modified version by adding the following entry somewhere in the JSON: 
    ```json
    "loginFormType": "ldap",
    ```

The question is: How do you get a valid `conf.json`?

* The [relevant section of the Taiga 30 min setup guide](https://community.taiga.io/t/taiga-30min-setup/170#map-a-confjson-file-23) recommends to use an example `config.json` which you then have to adjust.
* Alternatively, you could also start the container first without any adjustments, and then copy the file out like this:
    ```bash
    docker cp taiga_taiga-front_1:/usr/share/nginx/html/conf.json conf.json
    ```
    You then have a valid, production-ready `conf.json` you can just extend by the entry mentioned above. I'd recommend this method.

## :package: Installation without Docker

### Installation

Install the PIP package `taiga-contrib-ldap-auth-ext` in your `taiga-back` python virtualenv:

```bash
pip install taiga-contrib-ldap-auth-ext
```

If needed, change `pip` to `pip3` to use the Python 3 version.

For an even simpler installation, you can use our own Docker image: <https://github.com/Monogramm/docker-taiga>

### `taiga-back`

Append the contents of the [`taiga-back` configuration](#taiga-back-configuration) section from this document to the file `settings/common.py` (for Taiga >5.0) or `settings/local.py` (for Taiga ≤5.0).

### `taiga-front`

Change the `loginFormType` setting to `"ldap"` in `dist/conf.json`:

```json
"loginFormType": "ldap",
```

## :wrench: Configuration

### `taiga-back` configuration

If you use the installation with Docker, put something similar to the following in the file `custom-back/config.append.py`. 

If you use the installation without Docker, append something similar to the following to the file `settings/common.py` (for Taiga >5.0) or `settings/local.py` (for Taiga ≤5.0):

```python
INSTALLED_APPS += ["taiga_contrib_ldap_auth_ext"]

# Multiple LDAP servers are currently not supported, see
# https://github.com/Monogramm/taiga-contrib-ldap-auth-ext/issues/16
LDAP_SERVER = "ldaps://ldap.example.com"
LDAP_PORT = 636

LDAP_BIND_DN = "CN=SVC Account,OU=Service Accounts,OU=Servers,DC=example,DC=com"
LDAP_BIND_PASSWORD = "verysecurepassword"

LDAP_SEARCH_BASE = 'OU=DevTeam,DC=example,DC=net'

LDAP_USERNAME_ATTRIBUTE = "uid"
LDAP_EMAIL_ATTRIBUTE = "mail"
LDAP_FULL_NAME_ATTRIBUTE = "givenName"

LDAP_SAVE_LOGIN_PASSWORD = False

LDAP_MAP_USERNAME_TO_UID = None
```

_You need to change most of the values to match your setup._

<details>
<summary>Click here to expand configuration explanation</summary>

**`LDAP_SERVER` and `LDAP_PORT`:** You will definitely have to change the server URL. If possible, try to keep the `ldaps://` to use a secure connection. The port can likely stay as is, unless...

* ... you run the LDAP server on a different (non-standard) port.
* ... you want to use unencrypted, insecure LDAP: In this case, change `ldaps://` to `ldap://` and the port to 389.
* ... you want to use STARTTLS. In this case, you have to make the same changes as for unencrypted, insecure LDAP and set `LDAP_START_TLS = True`, making the section look like this:
    ```python
    LDAP_SERVER = "ldap://ldap.example.com"
    LDAP_PORT = 389
    LDAP_START_TLS = True
    ```
    What happens is that an unencrypted connection is established first, but then upgraded to a secure connection. This is [less secure](https://docs.redhat.com/de/documentation/red_hat_directory_server/12/html/securing_red_hat_directory_server/assembly_enabling-tls-encrypted-connections-to-directory-server_securing-rhds#assembly_enabling-tls-encrypted-connections-to-directory-server_securing-rhds) than `ldaps://` (see also [the related discussion for STARTTLS for emails](https://serverfault.com/questions/523804/is-starttls-less-safe-than-tls-ssl) or [this blog post](https://blog.apnic.net/2021/11/18/vulnerabilities-show-why-starttls-should-be-avoided-if-possible/)), because an attacker could strip the “upgrade to secure connection” request causing the connection to remain insecure. It is still safer than an unecrypted connection, of course.

**`LDAP_BIND_DN`, `LDAP_BIND_PASSWORD`**: You will need to change them. 

The bind user is a dedicated service account. The plugin will connect to the LDAP server using this service account and search for an LDAP entry that has a `LDAP_USERNAME_ATTRIBUTE` or `LDAP_EMAIL_ATTRIBUTE` matching the user-provided login.

If the search is successful, the found LDAP entry and the user-provided password are used to attempt a bind to LDAP. If the bind is successful, then we can say that the user is authorised to log in to Taiga.

If `LDAP_BIND_DN` is not specified or blank, an anonymous bind is attempted.

It is recommended to limit the service account and only allow it to read and search the LDAP structure (no write or other LDAP access). The credentials should also not be used for any other account on the network. This minimizes the damage in cases of a successful LDAP injection or if you ever accidentially give someone access to the configuration file (e.g. by committing it into version control or having misconfigured permissions). Use a suitably strong, ideally randomly generated password.

You can also use the credentials provided by the user to bind to LDAP (eliminating the need for a dedicated LDAP service account). To do so, do the following three things:

1. Set `LDAP_BIND_WITH_USER_PROVIDED_CREDENTIALS = True`
2. Insert the placeholder `<username>` inside `LDAP_BIND_DN`, e.g. like this: `"CN=<username>,OU=DevTeam,DC=example,DC=com"`.
3. Remove `LDAP_BIND_PASSWORD` (it will not be used)

Taiga will then determine the LDAP bind user by replacing `<username>` with the user-provided username, and bind using the user-provided password.

**`LDAP_SEARCH_BASE`**: The subtree where the users are located.

**`LDAP_USERNAME_ATTRIBUTE`, `LDAP_EMAIL_ATTRIBUTE`, `LDAP_FULL_NAME_ATTRIBUTE`**: These are the LDAP attributes used to get the username, email and full name shown in the Taiga application. They need to have a value in LDAP. Depending on your LDAP setup, you might need to change them.

**`LDAP_SAVE_LOGIN_PASSWORD`**: Set this to `True` or remove the line if you want to store the passwords in the local database as well.

**`LDAP_MAP_USERNAME_TO_UID`**: This line fixes a bug. If omitted, the plugin will likely crash and no authentication is possible.

<!-- TODO: Explain this -->
</details>

#### Additional configuration options

<details>
<summary>Click here to expand additional configuration options</summary>

By default, Taiga will fall back to `normal` authentication if LDAP authentication fails. Add the following line to disable this and only allow LDAP login:

```python
LDAP_FALLBACK = ""
```

You can specify additional search criteria that will be ANDed using the following line:

```python
LDAP_SEARCH_FILTER_ADDITIONAL = '(mail=*)'
```

If you want to change how the LDAP username, e-mail or name are mapped to the local database, you can use the following lines to do so:

```python
def _ldap_slugify(uid: str) -> str:
    """Map an LDAP username to a local DB user unique identifier.

    Upon successful LDAP bind, will override returned username attribute
    value. May result in unexpected failures if changed after the database
    has been populated. 
    """

    # example: force lower-case
    return uid.lower()
    
LDAP_MAP_USERNAME_TO_UID = _ldap_slugify


def _ldap_map_email(email: str) -> str:
    ...

def _ldap_map_name(name: str) -> str:
    ...

LDAP_MAP_EMAIL = _ldap_map_email
LDAP_MAP_NAME = _ldap_map_name
```

To support alternative TLS ciphersuites, protocol versions or disable certificate validation (note that all of these options have the power to harm your security, so apply them with caution), use the following lines:

```python
from ldap3 import Tls
import ssl

# Add or remove options or change values as necessary.
LDAP_TLS_CERTS = Tls(validate=ssl.CERT_NONE, version=ssl.PROTOCOL_TLSv1, ciphers='RSA+3DES')
```

To not store the passwords in the local database, use the following line:

```python
LDAP_SAVE_LOGIN_PASSWORD = False
```

Group management via LDAP does not yet exist, see issues #15 and #17. However, the configuration would look a bit like this:

```python
# Group search filter where $1 is the project slug and $2 is the role slug
#LDAP_GROUP_SEARCH_FILTER = 'CN=$2,OU=$1,OU=Groups,DC=example,DC=net'
# Use an attribute in the user entry for membership
#LDAP_USER_MEMBER_ATTRIBUTE = 'memberof,primaryGroupID'
# Starting point within LDAP structure to search for login group
#LDAP_GROUP_SEARCH_BASE = 'OU=Groups,DC=example,DC=net'
# Group classes filter
#LDAP_GROUP_FILTER = '(|(objectclass=group)(objectclass=groupofnames)(objectclass=groupofuniquenames))'
# Group member attribute
#LDAP_GROUP_MEMBER_ATTRIBUTE = 'memberof,primaryGroupID'

# Taiga super users group id
#LDAP_GROUP_ADMIN = 'OU=TaigaAdmin,DC=example,DC=net'
```

</details>

## :bulb: Further notes

* **Security recommendation**: The service account to perform the LDAP search should be configured to only allow reading/searching the LDAP structure. No other LDAP (or wider network) permissions should be granted for this user because you need to specify the service account password in the configuration file. A suitably strong password should be chosen, eg. `VmLYBbvJaf2kAqcrt5HjHdG6`.
* If you are using the Taiga's built-in `USER_EMAIL_ALLOWED_DOMAINS` config option, all LDAP email addresses will still be filtered through this list. Ensure that if `USER_EMAIL_ALLOWED_DOMAINS` != `None`, that your corporate LDAP email domain is also listed there. This is due to the fact that LDAP users are automatically "registered" behind the scenes on their first login.
* If you plan to only allow your LDAP users to access Taiga, set the `PUBLIC_REGISTER_ENABLED` config option to `False`. This will prevent any external user to register while still automatically register LDAP users on their first login.
* Instead of appending to the `common.py` file in `taiga-back`, you can also insert the configuration into `config.py`. In our tests, both ways worked.
