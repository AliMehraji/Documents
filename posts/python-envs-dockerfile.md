---
title: Python Env Variables in Dockerfile
published: true
description: Python environment variables to set globally in Dockerfile
tags: 'docker,python,devops'
id: 2722999
date: '2025-07-25T17:45:36Z'
---

It is necessary to set Python-specific environment variables in Dockerfile. Some of these variables are optional, while others are essential.

```dockerfile
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=on \
    PIP_TIMEOUT=60 \
    PIP_INDEX_URL=https://${JFROG}/artifactory/api/pypi/python/simple/
```

- [PYTHONUNBUFFERED][envvar-PYTHONUNBUFFERED]
  > Force the `stdout` and stderr streams to be unbuffered. This option has no effect on the stdin stream.
  - Ensure **logs appear immediately** in the container logs, Disables output buffering for `stdout` and `stderr`.
  - Crucial for:
    - **Debugging** or **monitoring logs** (e.g., in `docker logs` or CI pipelines).
    - **Streaming logs** to systems like `ELK`, `Loki`, etc.

- [PYTHONDONTWRITEBYTECODE][envvar-PYTHONDONTWRITEBYTECODE]
  > If this is set to a non-empty string, Python won’t try to write .pyc files on the import of source modules. This is equivalent to specifying the -B option.
  - Disables writing `.pyc` files (compiled bytecode) to disk.
  - Prevents Python from creating:
    - `__pycache__/`
    - `.pyc` files
  - Keeps the container **clean and lightweight**.
  - Avoids unnecessary file writes (especially important on container read-only filesystems or volume mounts).

- `PIP_INDEX_URL` is for on-premises/private PYPI artifactory

    > pip’s command line options can be set with environment variables using the format `PIP_<UPPER_LONG_NAME>`.
    >
    > Dashes (-) have to be replaced with underscores (_).

    Example, invoke `pip --help` to get the cli options (flags)

      * `--disable-pip-version-check` --> `PIP_DISABLE_PIP_VERSION_CHECK`
      * `--timeout` --> `PIP_TIMEOUT`
      * `--no-color` --> `PIP_NO_COLOR` </br></br></br>

    ```bash
      General Options:
        --proxy <proxy>             Specify a proxy in the form scheme://[user:passwd@]proxy.server:port.
        --retries <retries>         Maximum attempts to establish a new HTTP connection. (default: 5)
        --timeout <sec>             Set the socket timeout (default 15 seconds).
        --exists-action <action>    Default action when a path already exists: (s)witch, (i)gnore, (w)ipe, (b)ackup, (a)bort.
        --trusted-host <hostname>   Mark this host or host:port pair as trusted, even though it does not have valid or any HTTPS.
        --cert <path>               Path to PEM-encoded CA certificate bundle. If provided, overrides the default. See 'SSL Certificate
                                    Verification' in pip documentation for more information.
        --client-cert <path>        Path to SSL client certificate, a single file containing the private key and the certificate in PEM
                                    format.
        --cache-dir <dir>           Store the cache data in <dir>.
        --no-cache-dir              Disable the cache.
        --disable-pip-version-check Dont periodically check PyPI to determine whether a new version of pip is available for download. Implied with --no-index.
        --no-color                  Suppress colored output.
        --use-feature <feature>     Enable new functionality, that may be backward incompatible.
        --use-deprecated <feature>  Enable deprecated functionality, that will be removed in the future.
        --resume-retries <resume_retries>
                                    Maximum attempts to resume or restart an incomplete download. (default: 0)
    ```

Notice

- **Security best practices in K8S**

  > running processes with `nonroot` users with `USER nonroot:nonroot` in Dockerfile Or `SecurityContexts` Like `fsGroup` , `runAsGroup`, and `runAsUser`, in pod template, the filesystem is read-only so the `PYTHONDONTWRITEBYTECODE=1` has to be set.

## Resources

- [Python CommandLine and Environment](https://docs.python.org/3/using/cmdline.html#command-line-and-environment)
- [PYTHONUNBUFFERED][envvar-PYTHONUNBUFFERED]
- [PYTHONDONTWRITEBYTECODE][envvar-PYTHONDONTWRITEBYTECODE]
- [PIP Environment Variables][pip-env-variable]
- [Install packages using pip][install-packages-using-pip]

[envvar-PYTHONUNBUFFERED]: https://docs.python.org/3/using/cmdline.html#envvar-PYTHONUNBUFFERED
[envvar-PYTHONDONTWRITEBYTECODE]: https://docs.python.org/3/using/cmdline.html#envvar-PYTHONDONTWRITEBYTECODE
[pip-env-variable]: https://pip.pypa.io/en/stable/topics/configuration/#environment-variables
[install-packages-using-pip]: https://packaging.python.org/en/latest/guides/installing-using-pip-and-virtual-environments/
