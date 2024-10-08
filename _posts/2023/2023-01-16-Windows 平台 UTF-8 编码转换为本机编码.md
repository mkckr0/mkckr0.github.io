---
title: Windows 平台 UTF-8 编码转换为本机编码
date: 2023-01-16 20:40
---

```cpp
std::string from_utf8(const std::string& src)
{
    int n = MultiByteToWideChar(CP_UTF8, 0, src.c_str(), (int)src.size(), 0, 0);
    std::vector<wchar_t> wbuf(n);
    MultiByteToWideChar(CP_UTF8, 0, src.c_str(), (int)src.size(), wbuf.data(), (int)wbuf.size());

    UINT cp = GetACP();
    n = WideCharToMultiByte(cp, 0, wbuf.data(), (int)wbuf.size(), 0, 0, 0, 0);
    std::vector<char> buf(n);
    WideCharToMultiByte(cp, 0, wbuf.data(), (int)wbuf.size(), buf.data(), (int)buf.size(), 0, 0);

    return std::string(buf.data(), buf.size());
}
```