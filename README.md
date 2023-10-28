# php &ndash; Ansible Role

<!-- [![CI](https://github.com/xolyu/ansible-role-ROLE_NAME/actions/workflows/ci.yml/badge.svg)](https://github.com/xolyu/ansible-role-ROLE_NAME/actions/workflows/ci.yml) -->

Installs and configures PHP. The installation is done from the system package sources or by including [Ondřej's package sources](https://packages.sury.org/php/) or the [PPA on Ubuntu](https://launchpad.net/~ondrej/+archive/ubuntu/php/).

The role allows managing present PHP versions and automatically installing the latest PHP version. Furthermore, a differentiated configuration of the different PHP versions is possible.


## Requirements

* Systempackage `python3-debian` &ndash; for Ansible's module [`deb822_repository`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/deb822_repository_module.html), only on Debian when using Ondřej's package source.

*For automatic ensuring of the packages, see variable `php_ensure_requirements`.*


## Dependencies

* Ansible-Core Version >= 2.15, because of [`deb822_repository`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/deb822_repository_module.html) module

### Suggestions

* [`Community.General`](https://docs.ansible.com/ansible/latest/collections/community/general/index.html) collection ([Galaxy](https://galaxy.ansible.com/ui/repo/published/community/general/)), Version >= 2.2.0


## Role Variables

* **`php_package_source`**  
  Package source to be used.  
  Choices: `system_maintainer`, `sury_ondrej`  
  Default: `system_maintainer`

* **`php_ensure_requirements`**  
  Provides for the installation of the packages listed in section requirements.  
  Type: bool  
  Default: `no`

* **`php_stable_versions_only`**  
  Filters out release candidates when `yes` is selected, so `latest` points to the latest stable version. If `no` is selected, `latest` may point to an RC version if one is available.  
  _For `yes` applies: You can still install an RC version by specifying the corresponding version number. If at least one stable version is installed next to an RC version, `latest` points to the latest stable version for the configuration (ini, FPM pool, etc.). If no stable version is installed, `latest` points to the latest installed RC version for the configuration._  
  This only affects Ondrej's repository.  
  Type: bool  
  Default: `yes`

* **`php_versions_present`**  
  Defines the PHP versions that should be present.  
  Type: str *or* List of Strings  
  Choices: `latest`, *Version e.g. `8.1`*  
  Default: `latest`
  * `latest` &ndash; means the latest version available in the package sources.

* **`php_versions_absent`**  
  Defines the PHP versions that should be absent.  
  Type: str *or* List of Strings  
  Choices: `[]`, `all_except_present`, *Version e.g. `8.1`*  
  Default: `[]`
  * `all_except_present` &ndash; means all installed versions which are not mentioned under `php_versions_present`.

* **`php_fpm_enabled`**  
  Determines whether FPM packages are installed and configured. The configuration affects php.ini, as well as the FPM pools.  
  Type: bool  
  Default: `yes`

* **`php_apache_mod_enabled`**  
  Determines whether the Apache2 PHP module is installed and configured.  
  Type: bool  
  Default: `no`

* **`php_extensions`**  
  List of PHP extensions to be installed for all desired PHP versions. Extensions which are not available in the package sources for a PHP version will be skipped, the installation error will be ignored.  
  Type: List of Strings  
  Default: *see defaults/main.yml*

* **`php_extensions_extra`**  
  The same as `php_extensions`, additional packages can be defined without having to redefine all packages from `php_extensions`.  
  Type: List of Strings  
  Default: `[]`

* **`php_ini_file_backup`**  
  Specifies whether a backup file should be created so that you can get the original file or previous state back. With keep-original a copy is created only once, with the extension .original. With yes a copy with timestamp information is created for each change.  
  Choices: `keep-original`, `yes`, `no`  
  Default: `keep-original`

* **`php_ini_settings`**  
  Defines individual values that should be defined for the php.ini file. Other parameters of the existing php.ini file remain unaffected by changes. These settings apply to all php.ini files (all versions and variants).  
  Type: Dict  
  Default: *see defaults/main.yml*
  * **The ini settings dict**  
    The configuration is defined as a dict. The Key is the section of the INI file and the identifier of the option. The Value is the respective value of the option.  
    The Key is specified as `Section/Option`, if the key is only defined as `Option` (without `/`) the section `PHP` is implicitly assumed. Section and option are case-sensitive.
  * Example with implicit PHP section
    ```yml
    php_ini_settings:
      max_execution_time: 30
    ```
    results in
    ```ini
    [PHP]
    max_execution_time = 30
    ```
  * Example with explicit specification of the section:
    ```yml
    php_ini_settings:
      Date/date.timezone: Europe/Berlin
    ```
    results in
    ```ini
    [Date]
    date.timezone = Europe/Berlin
    ```

* **`php_ini_settings_specific`**  
  The same as `php_ini_settings`, but with the difference that the settings are not valid for all php.ini files, but only for the respective application criteria. The dict of the first level defines with the key the application criteria of the settings. The value corresponds to the settings according to *The ini settings dict* described in `php_ini_settings`.  
  Type: Dict of Dicts  
  Default: *see defaults/main.yml*
  * **The application criteria** - according to the priority  
    *(Higher priority overwrites the setting of lower priority.)*
    1. `latest/<VARIANT>` or `not-latest/<VARIANT>`, e.g. `latest/fpm`
    2. `<VERSION>/<VARIANT>`, e.g. `8.1/fpm`, `8.3/apache2`
    3. `latest` or `not-latest`
    4. `<VERSION>`, e.g. `8.1`, `8.3`
    5. `<VARIANT>`, choices: `cli`, `fpm`, `apache2`
    6. settings from variable `php_ini_settings`
  * Notes
    * The keyword latest/not-latest is ranked higher than the version number that describes latest.
    * `latest` &ndash; means the latest version available in the package sources.
    * `not-latest` &ndash; means all versions that do not correspond to the `latest` version.

* **`php_confd_settings`**  
  Configurations saved as `.ini` files in the `conf.d` directory.  
  Type: Dict  
  Default: `{}`
  * **The confd settings dict**  
    The key corresponds to the file name, the file extension `.ini` is added automatically. The value can be a string with the content of the file or a dict as `{'state': 'absent'}` to delete a configuration.

* **`php_confd_settings_specific`**  
  Analogous to `php_ini_settings_specific` conf.d settings are defined, but only for the respective application criteria. The dict of the first level defines with the key the application criteria of the settings. The application criteria are the same as in `php_ini_settings_specific`. The value corresponds to the settings according to *The confd settings dict* described in `php_confd_settings`.  
  Type: Dict of Dicts  
  Default: `{}`

* **`php_fpm_pool_defaults`**  
  Defines the defaults that are used for all FPM pools for configuration.  
  Type: Dict  
  Default: *see defaults/main.yml*
  * **The fpm pool config dict**  
    The Key corresponds to the name of the option.  
    The Value is correspondingly the value of the option. The value can contain the placeholders `##NAME##` (name of the FPM pool) and `##VERSION##` (PHP version, e.g. `8.1`).  
    If the Value itself is a dict, the key is interpreted as a suboption.
  * Example with placeholder:
    ```yml
    mypool:
      listen: /run/php/php##VERSION##-fpm.##NAME##.sock
    ```
    results in
    ```ini
    [mypool]
    listen = /run/php/php8.1-fpm.mypool.sock
    ```
  * Example with Dict as value:
    ```yml
    mypool:
      env:
        HOSTNAME: $HOSTNAME
        TEMP: /tmp
    ```
    results in
    ```ini
    [mypool]
    env[HOSTNAME] = $HOSTNAME
    env[TEMP] = /tmp
    ```

* **`php_fpm_pools`**  
  Definition of the FPM pools. The Key corresponds to the pool name. The Value is the pool configuration, according to *The fpm pool config dict* described in `php_fpm_pool_defaults`.  
  Type: Dict of Dicts  
  Default: `{}`
  * **Options of _The fpm pool config dict_**
    * `_php_versions`: String or list of strings of all PHP versions for which the FPM pool should be configured.  
      Choices: `latest`, `all`, *Version e.g. `8.1`*, `[]`  
      Default: `latest`
    * `_state`: Defines whether the FPM pool is made available or removed.  
      Choices: `present`, `absent`  
      Default: `present`
  * Example for removing a pool:
    ```yml
    php_fpm_pools:
      mypool:
        _php_versions: all
        _state: absent
    ```
    is equivalent to
    ```yml
    php_fpm_pools:
      mypool:
        _php_versions: []
    php_fpm_pool_absent_unconfigured_versions: yes
    ```

* **`php_fpm_pool_absent_unconfigured_versions`**  
  Defines whether an FPM pool is deleted by its name for all installed PHP versions except the explicitly configured ones.  
  Type: bool  
  Default: `yes`

* **`php_fpm_pool_www_state`**  
  Describes what to do with the default FPM pool `www`. This option affects all installed PHP versions.  
  When disabled, the pool is renamed from `*.conf` to `*.disabled`, but is not deleted.  
  Choices: `noop` (do nothing), `enabled`, `disabled`  
  Default: `noop`


<!--
* **`VAR`**  
  DESC  
  Choices: `VAL`, `ANOTHER`  
  Default: `VAL`

* **`VAR`**  
  DESC  
  Type: List of Dicts  
  Default: `VAL`

* **`VAR`**  
  DESC  
  Type: bool  
  Default: `VAL`
-->


## Example Playbook

Install PHP in the latest version from the package sources of the system maintainer.

```yml
---
- hosts: webservers
  roles:
    - xolyu.php
```

Install PHP in the latest version from the package sources of Ondřej Surý.

```yml
- hosts: webservers
  roles:
    - role: xolyu.php
      php_package_source: sury_ondrej
```

Install PHP in the latest version for use with Apache's PHP module.

```yml
- hosts: webservers
  roles:
    - role: xolyu.php
      php_package_source: sury_ondrej
      php_fpm_enabled: no
      php_apache_mod_enabled: yes
```

Installation of the latest version and special extensions, as well as automatic uninstallation of not explicitly mentioned PHP versions.

```yml
---
- hosts: webservers

  vars:
    php_package_source: sury_ondrej

    php_versions_present:
      - '7.4'
      - latest

    php_versions_absent: all_except_present

    php_extensions_extra:
      - imap
      - pgsql
      - redis

  roles:
    - xolyu.php
```

More detailed configuration of FPM and FPM pools.

```yml
---
- hosts: webservers

  vars:
    php_package_source: sury_ondrej

    php_versions_present:
      - '7.4'
      - '8.2'

    php_ini_settings_specific:
      fpm:
        memory_limit: 256M
        post_max_size: 32M
        upload_max_filesize: 32M
      '7.4/fpm':
        memory_limit: 128M
      'latest/fpm':
        memory_limit: 512M

    php_confd_settings_specific:
      fpm:
        90-opcache-config: |
          opcache.enable = 1
          opcache.interned_strings_buffer = 8
          opcache.max_accelerated_files = 10000
          opcache.memory_consumption = 128
          opcache.save_comments = 1
          opcache.revalidate_freq = 1

    php_fpm_pools:
      mypool:
        _php_versions: latest
        listen: /run/php/fpm-pool.mypool.sock
        user: myapp
        group: myapp
      the-old:
        _php_versions: '7.4'
        listen: /run/php/fpm-pool.##NAME##.sock
        user: mylegacyapp
        group: mylegacyapp
      super:
        _php_versions: all
        listen: /run/php/php##VERSION##.fpm.##NAME##.sock


  roles:
    - xolyu.php
```


## License

GNU General Public License v3.0


## Author Information

Xolyu.
