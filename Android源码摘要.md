Android源码摘要
===
## 摘取阅读Android源码中的所得，一开始可能会很基础；
**1 realloc和读取CPU名
```
void get_hardware_name(char *hardware, unsigned int *revision)
{
    const char *cpuinfo = "/proc/cpuinfo";
    char *data = NULL;
    size_t len = 0, limit = 1024;
    int fd, n;
    char *x, *hw, *rev;

    /* Hardware string was provided on kernel command line */
    if (hardware[0])
        return;

    fd = open(cpuinfo, O_RDONLY);
    if (fd < 0) return;

    for (;;) {
        x = realloc(data, limit);
        if (!x) {
            ERROR("Failed to allocate memory to read %s\n", cpuinfo);
            goto done;
        }
        data = x;

        n = read(fd, data + len, limit - len);
        if (n < 0) {
            ERROR("Failed reading %s: %s (%d)\n", cpuinfo, strerror(errno), errno);
            goto done;
        }
        len += n;

        if (len < limit)
            break;

        /* We filled the buffer, so increase size and loop to read more */
        limit *= 2;
    }

    data[len] = 0;
    hw = strstr(data, "\nHardware");
    rev = strstr(data, "\nRevision");

    if (hw) {
        x = strstr(hw, ": ");
        if (x) {
            x += 2;
            n = 0;
            while (*x && *x != '\n') {
                if (!isspace(*x))
                    hardware[n++] = tolower(*x);
                x++;
                if (n == 31) break;
            }
            hardware[n] = 0;
        }
    }

    if (rev) {
        x = strstr(rev, ": ");
        if (x) {
            *revision = strtoul(x + 2, 0, 16);
        }
    }

done:
    close(fd);
    free(data);
}
```
