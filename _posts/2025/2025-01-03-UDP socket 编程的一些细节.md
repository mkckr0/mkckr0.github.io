---
title: UDP socket 编程的一些细节
date: 2025-01-03 10:21
---

UDP socket 编程在各种操作系统和编程语言都有不同的实现，但是接口都类似，一些细节也类似。下面以 Linux 为例，说明 UDP socket 编程的一些细节。

- UDP socket 被创建后，初始的 local_addr 是 `0.0.0.0:0`，remote_addr 为 NULL。

- bind 会设置 local_addr。

- connect 除了会设置 remote_addr，还会自动`bind <remote_host>:<random_port>`。例如：`connect 192.168.1.2:65533` 会自动 `bind 192.168.1.2:<random_port>`。

- sendto 会自动 `bind 0.0.0.0:<random_port>` 不会设置 remote_addr。

- recvfrom 无法自动 bind，因为 local_addr 是 `0.0.0.0:0` 无法收到数据包，会直接block。

- 先 bind 再 connect 不会有问题，因为 connect 或者 sendto 会判断是不是已经 bound。但是先 connect 再 bind 会重复 bind。

案例：如何设置 local_addr 为 `0.0.0.0:65530`，remote_addr 为 `192.168.1.2:65531`？  
先 `bind 0.0.0.0:65530`，再 `connect 192.168.1.2:65531`。

测试代码：

<details>
<summary markdown="0">core.hpp</summary>

```cpp
#include <iostream>
#include <string.h>

#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

using namespace std;

void exit_with_error()
{
    auto err = errno;
    cout << "error: " << strerror(err) << endl;
    exit(-1);
}

sockaddr_in get_addr(const char* host, uint16_t port)
{
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    inet_pton(AF_INET, host, &addr.sin_addr.s_addr);
    addr.sin_port = htons(port);
    return addr;
}

void bind(int fd, const char* host, uint16_t port)
{
    sockaddr_in bind_addr = get_addr(host, port);
    if (bind(fd, (sockaddr*)&bind_addr, sizeof(sockaddr_in)) == -1) {
        exit_with_error();
    }
}

string local_addr(int fd)
{
    sockaddr_in local_addr{};
    socklen_t addr_len = sizeof(local_addr);
    if (getsockname(fd, (sockaddr*)&local_addr, &addr_len) == -1) {
        exit_with_error();
    }

    char host[50]{};
    if (inet_ntop(local_addr.sin_family, &local_addr.sin_addr, host, sizeof(host)) == nullptr) {
        exit_with_error();
    }

    return string(host) + ":" + to_string(htons(local_addr.sin_port));
}

string remote_addr(int fd)
{
    sockaddr_in local_addr{};
    socklen_t addr_len = sizeof(local_addr);
    if (getpeername(fd, (sockaddr*)&local_addr, &addr_len) == -1) {
        return {};
    }

    char host[50]{};
    if (inet_ntop(local_addr.sin_family, &local_addr.sin_addr, host, sizeof(host)) == nullptr) {
        return {};
    }

    return string(host) + ":" + to_string(htons(local_addr.sin_port));
}

void print_addr(int fd)
{
    cout << "local_addr: " << local_addr(fd) << " remote_addr: " << remote_addr(fd) << endl;
}
```

</details>

<details>
<summary markdown="0">server.cpp</summary>

```cpp
#include "core.hpp"

int main(int argc, char* argv[])
{
    auto fd = socket(AF_INET, SOCK_DGRAM, 0);
    print_addr(fd);

    bind(fd, "127.0.0.1", 65530);
    cout << "bound" << endl;
    print_addr(fd);

    sockaddr_in sock_addr = get_addr("127.0.0.1", atoi(argv[1]));
    char buf[1024] = "abcd";
    auto len = sendto(fd, buf, strlen(buf), 0, (sockaddr*)&sock_addr, sizeof(sock_addr));
    if (len == -1) {
        exit_with_error();
    }
    cout << "len: " << len << endl;

    cin.get();
}
```

</details>

<details>
<summary  markdown="0">client.cpp</summary>

```cpp
#include "core.hpp"

int main()
{
    cout << "client\n";

    auto fd = socket(AF_INET, SOCK_DGRAM, 0);
    print_addr(fd);

    // auto addr = get_addr("192.168.56.111", 65530);
    // if (connect(fd, (const sockaddr*)&addr, sizeof(addr)) == -1) {
    //     exit_with_error();
    // }
    // print_addr(fd);

    // bind(fd, "127.0.0.1", 65532);
    // print_addr(fd);

    char buf[1024] = "abcd";

    auto sock_addr = get_addr("127.0.0.1", 65530);
    if (sendto(fd, buf, 0, 0, (sockaddr*)&sock_addr, sizeof(sock_addr)) == -1) {
        exit_with_error();
    }
    print_addr(fd);

    // sockaddr_in addr2{};
    // socklen_t addr_len{};
    // auto len = recvfrom(fd, buf, sizeof(buf), 0, (sockaddr*)&addr2, &addr_len);
    // if (len == -1) {
    //     exit_with_error();
    // }
    // print_addr(fd);

    cin.get();
}
```

</details>