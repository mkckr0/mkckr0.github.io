---
title: 使用 FormatMessage 格式化 Windows 错误码
date: 2022-01-26 01:56
---

> https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-formatmessage

```cpp
#include <string>

#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
#endif // !WIN32_LEAN_AND_MEAN
#include <Windows.h>

std::string str_win_err(int err)
{
	LPSTR buf = nullptr;
	FormatMessageA(
		FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_ALLOCATE_BUFFER,
		nullptr,
		err,
		0,
		(LPSTR)&buf,
		0,
		nullptr
	);
	std::string msg;
	if (buf) {
		msg.assign(buf);
        LocalFree(buf);
	}
	return msg;
}
```
