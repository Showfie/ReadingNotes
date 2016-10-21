Android源码摘要
===
### 摘取阅读Android源码中的所得，一开始可能会很基础；
#### 1 realloc和读取CPU名
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

#### 2 list.h
```
/*
 * Copyright (C) 2008-2013 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#ifndef _CUTILS_LIST_H_
#define _CUTILS_LIST_H_

#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif /* __cplusplus */

struct listnode
{
    struct listnode *next;
    struct listnode *prev;
};

#define node_to_item(node, container, member) \
    (container *) (((char*) (node)) - offsetof(container, member))

#define list_declare(name) \
    struct listnode name = { \
        .next = &name, \
        .prev = &name, \
    }

#define list_for_each(node, list) \
    for (node = (list)->next; node != (list); node = node->next)

#define list_for_each_reverse(node, list) \
    for (node = (list)->prev; node != (list); node = node->prev)

#define list_for_each_safe(node, n, list) \
    for (node = (list)->next, n = node->next; \
         node != (list); \
         node = n, n = node->next)

static inline void list_init(struct listnode *node)
{
    node->next = node;
    node->prev = node;
}

static inline void list_add_tail(struct listnode *head, struct listnode *item)
{
    item->next = head;
    item->prev = head->prev;
    head->prev->next = item;
    head->prev = item;
}

static inline void list_add_head(struct listnode *head, struct listnode *item)
{
    item->next = head->next;
    item->prev = head;
    head->next->prev = item;
    head->next = item;
}

static inline void list_remove(struct listnode *item)
{
    item->next->prev = item->prev;
    item->prev->next = item->next;
}

#define list_empty(list) ((list) == (list)->next)
#define list_head(list) ((list)->next)
#define list_tail(list) ((list)->prev)

#ifdef __cplusplus
};
#endif /* __cplusplus */

#endif
```
#### 3 等待文件生成
```
int wait_for_file(const char *filename, int timeout)
{
    struct stat info;
    time_t timeout_time = gettime() + timeout;
    int ret = -1;

    while (gettime() < timeout_time && ((ret = stat(filename, &info)) < 0))
        usleep(10000);

    return ret;
}
```
#### 4 修改进程名
```
#define PROCESS_NAME_DEVICE "/sys/qemu_trace/process_name"
void set_process_name(const char* new_name) {
#ifdef HAVE_ANDROID_OS
    char  propBuf[PROPERTY_VALUE_MAX];
#endif

    if (new_name == NULL) {
        return;
    }

    // We never free the old name. Someone else could be using it.
    int len = strlen(new_name);
    char* copy = (char*) malloc(len + 1);
    strcpy(copy, new_name);
    process_name = (const char*) copy;

#if defined(HAVE_PRCTL)
    if (len < 16) {
        prctl(PR_SET_NAME, (unsigned long) new_name, 0, 0, 0);
    } else {
        prctl(PR_SET_NAME, (unsigned long) new_name + len - 15, 0, 0, 0);
    }
#endif

#ifdef HAVE_ANDROID_OS
    // If we know we are not running in the emulator, then return.
    if (running_in_emulator == 0) {
        return;
    }

    // If the "running_in_emulator" variable has not been initialized,
    // then do it now.
    if (running_in_emulator == -1) {
        property_get("ro.kernel.qemu", propBuf, "");
        if (propBuf[0] == '1') {
            running_in_emulator = 1;
        } else {
            running_in_emulator = 0;
            return;
        }
    }

    // If the emulator was started with the "-trace file" command line option
    // then we want to record the process name in the trace even if we are
    // not currently tracing instructions (so that we will know the process
    // name when we do start tracing instructions).  We do not need to execute
    // this code if we are just running in the emulator without the "-trace"
    // command line option, but we don't know that here and this function
    // isn't called frequently enough to bother optimizing that case.
    int fd = open(PROCESS_NAME_DEVICE, O_RDWR);
    if (fd < 0)
        return;
    write(fd, process_name, strlen(process_name) + 1);
    close(fd);
#endif
}
```
