```bash
for lib in $(ldd /bin/ls | awk '{print $3}' | grep '^/'); do
    cp --parents "$lib" /tmp/myroot
done

```
