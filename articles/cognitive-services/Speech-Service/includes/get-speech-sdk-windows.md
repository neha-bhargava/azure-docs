---
author: eric-urban
ms.service: cognitive-services
ms.topic: include
ms.date: 03/27/2020
ms.author: eur
ms.custom: devx-track-csharp
---

:::row:::
    :::column span="3":::
        The Speech SDK supports Windows 10 and Windows Server 2016, or later versions. Earlier versions are *not* officially supported. It's possible to use parts of the Speech SDK with earlier versions of Windows, although it's not advised.
    :::column-end:::
    :::column:::
        <br>
        <div class="icon is-large">
            <img alt="Windows" src="/media/logos/logo_Windows.svg" width="60px">
        </div>
    :::column-end:::
:::row-end:::

### System requirements

The Speech SDK on Windows requires the <a href="https://support.microsoft.com/help/2977003/the-latest-supported-visual-c-downloads" target="_blank">Microsoft Visual C++ Redistributable for Visual Studio 2019 </a> on the system.

- <a href="https://aka.ms/vs/16/release/vc_redist.x86.exe" target="_blank">Install for x86 </a>
- <a href="https://aka.ms/vs/16/release/vc_redist.x64.exe" target="_blank">Install for x64 </a>
- <a href="https://aka.ms/vs/16/release/vc_redist.arm64.exe" target="_blank">Install for ARMx64 </a>

### C#

[!INCLUDE [Get .NET Speech SDK](get-speech-sdk-dotnet.md)]

The required Speech SDK files can be deployed in the same directory as your application. This way your application can directly access the libraries. Make sure you select the correct version (x86/x64) that matches your application.

| Name                                            | Function                                             |
|-------------------------------------------------|------------------------------------------------------|
| *Microsoft.CognitiveServices.Speech.core.dll*   | Core SDK, required for native and managed deployment |
| *Microsoft.CognitiveServices.Speech.csharp.dll* | Required for managed deployment                      |

Starting with the release 1.3.0, the *Microsoft.CognitiveServices.Speech.csharp.bindings.dll* file that shipped in previous releases isn't needed anymore. The functionality is now integrated in the core SDK.

> [!IMPORTANT]
> For the Windows Forms App (.NET Framework) C# project, make sure the libraries are included in your project's deployment settings. You can check this under **Properties** > **Publish Section**. Select the **Application Files** button and find corresponding libraries from the dropdown list. Make sure the value is set to **Included**. Visual Studio includes the file when the project is published or deployed.

### C++

[!INCLUDE [Get C++ Speech SDK](get-speech-sdk-cpp.md)]

### Python

[!INCLUDE [Get Python Speech SDK](get-speech-sdk-python.md)]

### Java

[!INCLUDE [Get Java Speech SDK](get-speech-sdk-java.md)]
