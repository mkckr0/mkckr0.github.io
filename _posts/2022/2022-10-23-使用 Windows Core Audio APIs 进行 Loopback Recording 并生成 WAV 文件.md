---
title: 使用 Windows Core Audio APIs 进行 Loopback Recording 并生成 WAV 文件
date: 2022-10-23 15:28
---

> ## 参考文档
> [COM Coding Practices](https://learn.microsoft.com/en-us/windows/win32/learnwin32/com-coding-practices)
> [Audio File Format Specifications](https://www-mmsp.ece.mcgill.ca/Documents/AudioFormats/WAVE/WAVE.html)
> [Core Audio APIs](https://learn.microsoft.com/en-us/windows/win32/coreaudio/core-audio-apis-in-windows-vista)
> [Loopback Recording](https://learn.microsoft.com/en-us/windows/win32/coreaudio/loopback-recording)

```cpp
#include <iostream>
#include <fstream>
#include <vector>

#include <mmdeviceapi.h>
#include <combaseapi.h>
#include <atlbase.h>
#include <Functiondiscoverykeys_devpkey.h>
#include <Audioclient.h>
#include <Audiopolicy.h>

// 利用RAII手法，自动调用 CoUninitialize
class CoInitializeGuard {
public:
    CoInitializeGuard()
    {
        _hr = CoInitializeEx(nullptr, COINIT::COINIT_MULTITHREADED);
    }

    ~CoInitializeGuard()
    {
        if (_hr == S_OK || _hr == S_FALSE) {
            CoUninitialize();
        }
    }

    HRESULT result() const { return _hr; }

private:
    HRESULT _hr;
};

constexpr inline void exit_on_failed(HRESULT hr);
void printEndpoints(CComPtr<IMMDeviceCollection> pColletion);
std::string wchars_to_mbs(const wchar_t* s);

int main()
{
    HRESULT hr{};

    CoInitializeGuard coInitializeGuard;
    exit_on_failed(coInitializeGuard.result());

    // COM 对象都用 CComPtr 包装，会自动调用 Release
    // COM 接口分配的堆变量用 CComHeapPtr 包装，会自动调用 CoTaskMemFree
    CComPtr<IMMDeviceEnumerator> pEnumerator;
    hr = pEnumerator.CoCreateInstance(__uuidof(MMDeviceEnumerator));
    exit_on_failed(hr);

    // 打印所有可用的音频设备
    //CComPtr<IMMDeviceCollection> pColletion;
    //hr = pEnumerator->EnumAudioEndpoints(eRender, DEVICE_STATE_ACTIVE, &pColletion);
    //exit_on_failed(hr);
    //printEndpoints(pColletion);

    // 使用默认的 Audio Endpoint，eRender 表示音频播放设备，而不是录音设备
    CComPtr<IMMDevice> pEndpoint;
    hr = pEnumerator->GetDefaultAudioEndpoint(eRender, eConsole, &pEndpoint);
    exit_on_failed(hr);

    // 打印出播放设备的名字，可能包含中文
    CComPtr<IPropertyStore> pProps;
    hr = pEndpoint->OpenPropertyStore(STGM_READ, &pProps);
    exit_on_failed(hr);
    PROPVARIANT varName;
    PropVariantInit(&varName);
    hr = pProps->GetValue(PKEY_Device_FriendlyName, &varName);
    exit_on_failed(hr);
    std::cout << "select audio endpoint: " << wchars_to_mbs(varName.pwszVal) << std::endl;
    PropVariantClear(&varName);

    // 由 IMMDevice 对象 得到 IAudioClient 对象 
    CComPtr<IAudioClient> pAudioClient;
    hr = pEndpoint->Activate(__uuidof(IAudioClient), CLSCTX_ALL, nullptr, (void**)&pAudioClient);
    exit_on_failed(hr);

    // 获得音频播放设备格式信息
    CComHeapPtr<WAVEFORMATEX> pDeviceFormat;
    pAudioClient->GetMixFormat(&pDeviceFormat);

    constexpr int REFTIMES_PER_SEC = 10000000;      // 1 reference_time = 100ns
    constexpr int REFTIMES_PER_MILLISEC = 10000;

    // 初始化 IAudioClient 对象
    const REFERENCE_TIME hnsRequestedDuration = 2 * REFTIMES_PER_SEC; // 1s
    hr = pAudioClient->Initialize(AUDCLNT_SHAREMODE_SHARED, AUDCLNT_STREAMFLAGS_LOOPBACK, hnsRequestedDuration, 0, pDeviceFormat, nullptr);
    exit_on_failed(hr);

    // 获得缓冲区大小
    UINT32 bufferFrameCount{};
    hr = pAudioClient->GetBufferSize(&bufferFrameCount);
    exit_on_failed(hr);

    // 由 IAudioClient 对象 得到 IAudioCaptureClient 对象，也就是将音频播放设备视为录音设备
    CComPtr<IAudioCaptureClient> pCaptureClient;
    hr = pAudioClient->GetService(__uuidof(IAudioCaptureClient), (void**)&pCaptureClient);
    exit_on_failed(hr);
    
    // 开始录音
    hr = pAudioClient->Start();
    exit_on_failed(hr);

    const REFERENCE_TIME hnsActualDuration = (long long)REFTIMES_PER_SEC * bufferFrameCount / pDeviceFormat->nSamplesPerSec;

    std::ofstream ofile("./out.wav", std::ios::binary);
    if (!ofile) {
        exit(-1);
    }

    // 写入各种 header 信息

    constexpr UINT32 sizePlaceholder{};
    // master RIFF chunk
    ofile.write("RIFF", 4);
    ofile.write((const char*)&sizePlaceholder, 4);
    ofile.write("WAVE", 4);
    // 12

    // fmt chunk
    ofile.write("fmt ", 4);
    UINT32 fmt_ckSize = sizeof(WAVEFORMATEX) + pDeviceFormat->cbSize;
    ofile.write((const char*)&fmt_ckSize, 4);
    {
        auto p = pDeviceFormat.Detach();
        ofile.write((const char*)p, fmt_ckSize);
        pDeviceFormat.Attach(p);
    }
    // 8 + fmt_ckSize

    // fact chunk
    bool has_fact_chunt = pDeviceFormat->wFormatTag != WAVE_FORMAT_PCM;
    if (has_fact_chunt) {
        ofile.write("fact", 4);
        UINT32 fact_ckSize = 4;
        ofile.write((const char*)&fact_ckSize, 4);
        DWORD dwSampleLength{};
        ofile.write((const char*)&dwSampleLength, 4);
    }
    // 12

    // data chunk
    ofile.write("data", 4);
    ofile.write((const char*)&sizePlaceholder, 4);

    UINT32 data_ckSize = 0; // samples data 的大小
    UINT32 frame_count = 0; // 帧数

    constexpr int max_duration = 60;    // 录制 60s
    int seconds{};                      // 已经录制的时间

    time_t t_begin = time(NULL);

    //UINT32 
    do {
        // 睡眠一定时间，防止CPU占用率高
        Sleep(9);

        BYTE* pData{};                  // samples 数据
        UINT32 numFramesAvailable{};    // 缓冲区有多少帧
        DWORD dwFlags{};

        hr = pCaptureClient->GetBuffer(&pData, &numFramesAvailable, &dwFlags, NULL, NULL);
        exit_on_failed(hr);

        int frame_bytes = pDeviceFormat->nChannels * pDeviceFormat->wBitsPerSample / 8;
        int count = numFramesAvailable * frame_bytes;
        ofile.write((const char*)pData, count);
        data_ckSize += count;
        frame_count += numFramesAvailable;
        seconds = frame_count / pDeviceFormat->nSamplesPerSec;
        std::cout << "numFramesAvailable: " << numFramesAvailable << " seconds: " << seconds << std::endl;

        hr = pCaptureClient->ReleaseBuffer(numFramesAvailable);
        exit_on_failed(hr);

    } while (seconds < max_duration);

    // 检测实际花了多久，实际时间 - max_duration = 延迟
    time_t t_end = time(NULL);
    std::cout << "use wall clock: " << t_end - t_begin << "s" << std::endl;

    if (data_ckSize % 2) {
        ofile.put(0);
        ++data_ckSize;
    }

    UINT32 wave_ckSize = 4 + (8 + fmt_ckSize) + (8 + data_ckSize);
    ofile.seekp(4);
    ofile.write((const char*)&wave_ckSize, 4);

    if (has_fact_chunt) {
        ofile.seekp(12 + (8 + fmt_ckSize) + 8);
        ofile.write((const char*)&frame_count, 4);
    }

    ofile.seekp(12 + (8 + fmt_ckSize) + 12 + 4);
    ofile.write((const char*)&data_ckSize, 4);

    ofile.close();

    //所有 COM 对象和 Heap 都会自动释放
}

void printEndpoints(CComPtr<IMMDeviceCollection> pColletion)
{
    HRESULT hr{};

    UINT count{};
    hr = pColletion->GetCount(&count);
    exit_on_failed(hr);

    for (UINT i = 0; i < count; ++i) {
        CComPtr<IMMDevice> pEndpoint;
        hr = pColletion->Item(i, &pEndpoint);
        exit_on_failed(hr);

        CComHeapPtr<WCHAR> pwszID;
        hr = pEndpoint->GetId(&pwszID);
        exit_on_failed(hr);

        CComPtr<IPropertyStore> pProps;
        hr = pEndpoint->OpenPropertyStore(STGM_READ, &pProps);
        exit_on_failed(hr);

        PROPVARIANT varName;
        PropVariantInit(&varName);
        hr = pProps->GetValue(PKEY_Device_FriendlyName, &varName);
        exit_on_failed(hr);

        std::cout << wchars_to_mbs(varName.pwszVal) << std::endl;

        PropVariantClear(&varName);
    }
}

constexpr inline void exit_on_failed(HRESULT hr) {
    if (FAILED(hr)) {
        exit(-1);
    }
}

// 汉字会有编码问题，全部转成窄字符
std::string wchars_to_mbs(const wchar_t* src)
{
    UINT cp = GetACP();
    int ccWideChar = (int)wcslen(src);
    int n = WideCharToMultiByte(cp, 0, src, ccWideChar, 0, 0, 0, 0);

    std::vector<char> buf(n);
    WideCharToMultiByte(cp, 0, src, ccWideChar, buf.data(), (int)buf.size(), 0, 0);
    std::string dst(buf.data(), buf.size());
    return dst;
}
```
