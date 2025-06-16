# Upload/Download to/from Nexus Private Registry

## User / Password

- **User**:  `UserName`
- **Password**: `Password`

## Nexus Private / Proxy and group URLs

- Private Repository : `https://nexus.example.com/repository/pypi/simple`
- Proxy Repository : `https://nexus.example.com/repository/pypi-proxy/simple`
- group Repository of two above: `https://nexus.example.com/repository/pypi-group/simple`

### How To install

For installing python packages from nexus repository you need to pass `pip` the `--index-url` flag Or set a `PIP_INDEX_URL` environment.

```bash
pip install --index-url=https://USER:PASSWORD@nexus.example.com/repository/pypi-group/simple PACKAGE-NAME # pragma: allowlist secret
```

Or install packages from `requirements.txt` file:

```bash
pip install --index-url=https://USER:PASSWORD@nexus.example.com/repository/pypi-group/simple -r requirements.txt # pragma: allowlist secret
```

### How To just Download (No Installation) From private repository

```bash
pip download --index-url=https://USER:PASSWORD@nexus.example.com/repository/pypi-group/simple pillow==8.4.0 # pragma: allowlist secret
```

```text
Looking in indexes: https://UserName:****@nexus.example.com/repository/pypi-group/simple
Collecting pillow==8.4.0
  Downloading https://nexus.example.com/repository/pypi-group/packages/pillow/8.4.0/Pillow-8.4.0.tar.gz (49.4 MB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 49.4/49.4 MB 5.9 MB/s eta 0:00:00
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
  Preparing metadata (pyproject.toml) ... done
Saved ./Pillow-8.4.0.tar.gz
Successfully downloaded pillow
```

As You See above stdout `pip` prints Out where it is downloaded and saved:

> Saved ./Pillow-8.4.0.tar.gz

## Upload packages to Nexus

- First Install `twine`

    ```bash
    pip install twine
    ```

- Download The package or Your Custome package in gzip format `PACKAGE-NAME.tar.gz` I
   > f its your custom package just archive it and compress it , No need to download anything.

    ```bash
    pip download PACKAGE-NAME
    ```

- Upload Package

    ```bash
    twine upload --repository-url https://nexus.example.com/repository/pypi/ -u USER -p PASSWORD PACKAGE-NAME-VERSION.tar.gz
    ```

### Examples

- Installing `ipython`

   ```bash
   pip install --index-url=https://USER:PASSWORD@nexus.example.com/repository/pypi-group/simple ipython # pragma: allowlist secret
   ```

- Uploading `Pillow==8.4.0`

  - Download Package

  ```bash
  pip download Pillow==8.4.0
  ```

  - Find where it is (`pip` prints out where it's been downloaded.)
  - Upload with `twine`
   > When You're Uploading you have to upload it to Private Repository.

   ```bash
   twine upload --repository-url https://nexus.example.com/repository/pypi/ -u USER -p PASSWORD Pillow-8.4.0.tar.gz
   ```

## Sources

- [Sonatype PYPI Doc](https://help.sonatype.com/repomanager3/nexus-repository-administration/formats/pypi-repositories)
- [Sonatype Docs](https://help.sonatype.com/docs)
