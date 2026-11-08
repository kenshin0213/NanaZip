# ![NanaZip](Assets/NanaZip.png) NanaZip (Bandizip-Style UX Custom Edition)

> 본 저장소는 공식 [M2Team/NanaZip](https://github.com/M2Team/NanaZip)을 기반으로, **반디집(Bandizip) 스타일의 Windows 탐색기 우클릭 컨텍스트 메뉴 UX**를 적용한 커스텀 포크(Fork) 저장소입니다.

---

## 🇰🇷 주요 변경 사항 (Bandizip UX 개선 내역)

| 기능 | 기존 NanaZip 동작 | 변경 후 동작 |
| :--- | :--- | :--- |
| **1. 알아서 풀기 (Extract Smart) 최상단 배치** | 파일 열기, 여기에 풀기 아래쪽에 위치 | 압축 파일 우클릭 시 **알아서 풀기(&S)**를 **최상단(0순위)**에 배치하여 즉시 클릭 가능 (단일 루트 항목은 현재 폴더에, 다중 항목은 압축파일명 폴더를 생성해 해제) |
| **2. 각각의 파일명/폴더명으로 개별 압축 (.zip / .7z)** | 다중 선택 시 하나의 압축 파일로만 묶음 | 탐색기에서 **2개 이상의 파일/폴더 선택 시** **각각의 파일명/폴더명으로 압축 (.zip)** 및 **(.7z)** 메뉴 노출. 백그라운드 스레드에서 각 항목을 개별 압축 파일로 순차 생성 |
| **3. CRC / 해시(Hash) 메뉴 기본 숨김** | 모든 파일 우클릭 시 CRC SHA 메뉴 항상 표시 | 기본값에서 CRC SHA 컨텍스트 메뉴를 비활성화하여 우클릭 메뉴를 간소화 (필요 시 NanaZip 설정에서 다시 활성화 가능) |

### 🛠️ 세부 구현 및 안정성 처리
- **Windows 11/10 모던 셸 확장 & 내부 파일 매니저 동시 지원:** Out-of-proc COM 서버인 [NanaZip.ShellExtension.cpp](NanaZip.UI.Modern/NanaZip.ShellExtension.cpp)와 클래식/내부 컨텍스트 메뉴인 [ContextMenu.cpp](NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.cpp) 양쪽에 동일하게 반영되었습니다.
- **COM Surrogate (dllhost.exe) 수명 보호:** 다중 개별 압축 진행 중 dllhost.exe가 언로드되어 작업이 끊기지 않도록 winrt::get_module_lock() 및 GetModuleHandleExW + FreeLibraryAndExitThread 패턴으로 백그라운드 스레드 수명을 안전하게 보호합니다.
- **경로/파일명 엣지케이스 처리:**
  - 드라이브 루트(C:\) 보존 및 일반 폴더 끝 구분자 제거
  - 점이 포함된 폴더명(예: Project.v1.2)이 확장자로 오인되어 잘리지 않도록 파일 속성과 구분하여 처리
  - 동일한 이름의 압축 파일이 이미 존재할 경우 대소문자 구분 없이 파일명 (2).zip, 파일명 (3).zip 형태로 자동 넘버링
- 📄 **상세 설계 및 코드 검증 문서:** **[Documents/nanazip_bandizip_ux_verified.md](Documents/nanazip_bandizip_ux_verified.md)**

---

## 📦 다운로드 및 설치 방법 (Windows 10 / 11 x64)

직접 빌드하지 않고 바로 설치하려면 **[Releases 페이지](https://github.com/kenshin0213/NanaZip/releases/latest)**에서 배포용 패키지를 다운로드하세요.

1. **[최신 릴리스(v7.0.1882.0-bandizip-ux)](https://github.com/kenshin0213/NanaZip/releases/tag/v7.0.1882.0-bandizip-ux)**에서 **NanaZip_BandizipUX_x64_Deploy.zip** 파일을 다운로드하여 압축을 풉니다.
2. 압축을 푼 폴더 안의 **Install-FullPackage.cmd**를 더블클릭하여 실행합니다.
3. 관리자 권한 승인(UAC) 창에서 **예**를 누르면 아래 과정이 자동으로 진행됩니다:
   - 서명 인증서(NanaZip_DevCert.cer)를 로컬 컴퓨터 신뢰 저장소(Cert:\LocalMachine\TrustedPeople)에 등록
   - 정식 MSIX 패키지(NanaZip_BandizipUX_x64.msix)를 C:\Program Files\WindowsApps에 설치
   - Windows 탐색기(explorer.exe) 재시작 및 우클릭 메뉴 즉시 적용

---

## 📂 수정된 소스 파일 목록 (10개)

- [NanaZip.UI.Modern/NanaZip.ShellExtension.cpp](NanaZip.UI.Modern/NanaZip.ShellExtension.cpp) — Windows 11/10 탐색기 IExplorerCommand 컨텍스트 메뉴 및 개별 압축 워커 스레드 구현
- [NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Common/ZipRegistry.cpp](NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Common/ZipRegistry.cpp) — CContextMenuInfo::Load() 기본 플래그에서 kCRC | kCRC_Cascaded 제외
- [NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.cpp](NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.cpp) — 클래식/내부 컨텍스트 메뉴 kExtractHereSmart 최상단 배치 및 개별 압축 구현
- [NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.h](NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.h) — ECommandInternalID 열거형에 kCompressEachToZip, kCompressEachTo7z 추가
- [NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenuFlags.h](NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenuFlags.h) — 컨텍스트 메뉴 비트 플래그(1 << 16, 1 << 17) 정의
- [NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource.h](NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource.h) — 리소스 ID (2332, 2333) 정의
- [NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource2.rc](NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource2.rc) — 기본 영문 문자열 리소스 테이블 추가
- [NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/FileManager/MenuPage.cpp](NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/FileManager/MenuPage.cpp) — NanaZip 설정 > 컨텍스트 메뉴 항목 체크박스에 개별 압축 옵션 노출
- [NanaZipPackage/Strings/ko/Legacy.resw](NanaZipPackage/Strings/ko/Legacy.resw) — 한국어 다국어 리소스 (알아서 풀기(&S), 각각의 파일명/폴더명으로 압축 (.zip/.7z))
- [NanaZipPackage/Strings/en/Legacy.resw](NanaZipPackage/Strings/en/Legacy.resw) — 영어 다국어 리소스 (Resource2332, Resource2333)

---

*(아래는 원본 M2Team/NanaZip 프로젝트의 공식 영문 ReadMe입니다.)*

# ![NanaZip](Assets/NanaZip.png) NanaZip

[![GitHub Actions Build Status](https://github.com/M2Team/NanaZip/actions/workflows/BuildBinaries.yml/badge.svg?branch=master&event=push)](https://github.com/M2Team/NanaZip/actions/workflows/BuildBinaries.yml?query=event%3Apush+branch%3Amaster)
[![Total Downloads](https://img.shields.io/github/downloads/M2Team/NanaZip/total)](https://github.com/M2Team/NanaZip/releases)
[![Rating](https://img.shields.io/endpoint?url=https%3A%2F%2Fmicrosoft-store-badge.fly.dev%2Fapi%2Frating%3FstoreId%3D9N8G7TSCL18R%26market%3DUS&style=flat&color=brightgreen)](https://www.microsoft.com/store/productId/9N8G7TSCL18R)

[![Windows Store - Release Channel](https://img.shields.io/badge/Windows%20Store-Release%20Channel-blue)](https://www.microsoft.com/store/apps/9N8G7TSCL18R)
[![Windows Store - Preview Channel](https://img.shields.io/badge/Windows%20Store-Preview%20Channel-blue)](https://www.microsoft.com/store/apps/9NZL0LRP1BNL)

[![Latest Version - Release Channel](https://img.shields.io/github/v/release/M2Team/NanaZip?display_name=release&sort=date&color=%23a4a61d)](https://github.com/M2Team/NanaZip/releases/latest)
[![Latest Version - Preview Channel](https://img.shields.io/github/v/release/M2Team/NanaZip?include_prereleases&display_name=release&sort=date&color=%23a4a61d)](https://github.com/M2Team/NanaZip/releases)

[![Latest Release Downloads - Release Channel](https://img.shields.io/github/downloads/M2Team/NanaZip/latest/total)](https://github.com/M2Team/NanaZip/releases/latest)
[![Latest Release Downloads - Preview Channel](https://img.shields.io/github/downloads-pre/M2Team/NanaZip/latest/total)](https://github.com/M2Team/NanaZip/releases)

[![Download NanaZip from SourceForge mirror](https://img.shields.io/badge/SourceForge-Download-orange)](https://sourceforge.net/projects/nanazip/files/latest/download)

![ContextMenu](Documents/ContextMenu.png)
![MainWindowLightMode](Documents/MainWindowLightMode.png)
![MainWindowDarkMode](Documents/MainWindowDarkMode.png)

NanaZip is an open source file archiver intended for the modern Windows
experience, forked from the source code of well-known open source file archiver
7-Zip.

**All kinds of contributions will be appreciated. All suggestions, pull
requests, and issues are welcome.**

If you want to sponsor the development of NanaZip, please read the document of
[NanaZip Sponsor Edition](Documents/SponsorEdition.md). It's free for all
NanaZip source code repository contributors.

In general, NanaZip Sponsor Edition is more like contributor's edition, but
we provide a way to buy the honor and make someone look like a source code
contributor, because NanaZip is always a community-friendly open-source project.

If you'd like me to add features or improvements ahead of time, please use
[paid services](https://github.com/MouriNaruto/MouriNaruto/blob/main/PaidServices.md).

Besides NanaZip.org, NanaZip's official project pages are only hosted on GitHub,
Microsoft Store, My Digital Life Forums, SourceForge, and Windows App Community.

Kenji Mouri

## Features

- Inherit features from [7-Zip], [7-Zip ZS] and [7-Zip NSIS].
- Support the dark mode for all GUI components.
- Support the Mica effect on the main window. You will enjoy the full window
  immersive Mica for all GUI components if you are using dark mode with HDR
  disabled.
- Packaging with MSIX for modern deployment experience.
- Support the context menu in Windows 10/11 File Explorer.
- Provide 7-Zip execution alias for helping users to migrate to NanaZip.
- Support the Per-Monitor DPI-Aware for all GUI components.
- Support the i18n for GUI edition of Self Extracting Executable.
- Modernize message boxes and folder browsers.
- Provide the Smart Extraction feature.
- Provide the Open folder after extraction option.
- Provide the policy mechanism for enforcing settings (see
  [Policies.md](/Documents/Policies.md) for details)
- Propagates Mark-of-the-Web to all files by default.
- Provide additional hash algorithms.
  - MD2 (Inherit from 7-Zip ZS, but reimplemented with Windows CNG API)
  - MD4 (Inherit from 7-Zip ZS, but reimplemented with Windows CNG API)
  - MD5 (Inherit from 7-Zip, but reimplemented with Windows CNG API)
  - SHA-1 (Inherit from 7-Zip)
  - SHA-256 (Inherit from 7-Zip)
  - SHA512-224 (Inherit from 7-Zip)
  - SHA512-256 (Inherit from 7-Zip)
  - SHA-384 (Inherit from 7-Zip)
  - SHA-512 (Inherit from 7-Zip)
  - SHA3-224 (Inherit from 7-Zip)
  - SHA3-256 (Inherit from 7-Zip)
  - SHA3-384 (Inherit from 7-Zip)
  - SHA3-512 (Inherit from 7-Zip)
  - SHAKE128 (Inherit from 7-Zip)
  - SHAKE256 (Inherit from 7-Zip)
  - SHAKE128-256 (Inherit from 7-Zip)
  - SHAKE256-512 (Inherit from 7-Zip)
  - ED2K (Implemented with Windows CNG API)
  - AICH (Implemented with Windows CNG API and inlined RHash)
  - BTIH (Implemented with Windows CNG API and inlined RHash)
  - SHA-224 (Implemented with inlined RHash)
  - BLAKE2b (Implemented with RHash)
  - EDON-R 224 (Implemented with RHash)
  - EDON-R 256 (Implemented with RHash)
  - EDON-R 384 (Implemented with RHash)
  - EDON-R 512 (Implemented with RHash)
  - GOST R 34.11-94 (Implemented with RHash)
  - GOST R 34.11-94 CryptoPro (Implemented with RHash)
  - GOST R 34.11-2012 256 (Implemented with RHash)
  - GOST R 34.11-2012 512 (Implemented with RHash)
  - HAS-160 (Implemented with RHash)
  - RIPEMD-160 (Implemented with RHash)
  - Snefru-128 (Implemented with RHash)
  - Snefru-256 (Implemented with RHash)
  - Tiger (Implemented with RHash)
  - Tiger2 (Implemented with RHash)
  - TTH (Implemented with RHash)
  - Whirlpool (Implemented with RHash)
  - XXH32 (Implemented with xxHash)
  - XXH64 (Implemented with xxHash)
  - XXH3_64bits (Implemented with xxHash)
  - XXH3_128bits (Implemented with xxHash)
  - SM3 (Implemented with GmSSL)
- Provide additional codecs. (Note: Decoder can be considered as decompression
  support, Encoder can be considered as compression support, and Archiver can be
  considered as archive format support)
  - Brotli (Decoder & Encoder & Archiver, inherited from 7-Zip ZS)
  - Lizard (Decoder & Encoder & Archiver, inherited from 7-Zip ZS)
  - LZ (Archiver, inherited from 7-Zip ZS)
  - LZ4 (Decoder & Encoder & Archiver, inherited from 7-Zip ZS)
  - LZ5 (Decoder & Encoder & Archiver, inherited from 7-Zip ZS)
  - Zstandard (Decoder & Encoder & Archiver, inherited from 7-Zip ZS)
  - XChaCha20 (Decoder & Encoder, inherited from 7-Zip ZS)
  - XChaCha20-Poly1305 (Decoder & Encoder, inherited from 7-Zip ZS)
  - AES+XChaCha20-Poly1305 (Decoder & Encoder, inherited from 7-Zip ZS)
  - AES+XChaCha20+Ascon (Decoder & Encoder, inherited from 7-Zip ZS)
  - XChaCha20+AES+AEGIS (Decoder & Encoder, inherited from 7-Zip ZS)
  - NSIS (Decoder's NSIS script decompiling support, inherited from 7-Zip NSIS)
  - UFS/UFS2 file system image (Archiver, read-only, both big and little endian)
  - .NET Single File Application bundle (Archiver, read-only, extracting
    compressed files in the bundle are not supported)
  - Electron Archive (asar) (Archiver, read-only)
  - ROMFS file system image (Archiver, read-only)
  - ZealFS file system image (Archiver, read-only)
  - WebAssembly (WASM) binary file (Archiver, read-only)
  - littlefs file system image (**Work In Progress**) (Archiver, only block
    information can acquired)
- Provide additional security mitigations.
  - Enable Control Flow Guard (CFG) to all target binaries for mitigating ROP
    attacks.
  - Mark all x86 and x64 target binaries as compatible with Control-flow
    Enforcement Technology (CET) Shadow Stack.
  - Strict handle checks at runtime to block the use of invalid handles.
  - Disable dynamic code generation in Release builds prevents generating
    malicious code at runtime.
  - Block loading unexpected libraries from remote sources at runtime.
  - Enable Package Integrity Check.
  - Enable EH Continuation Metadata.
  - Enable Signed Returns.
  - Enable disabling child process creation for NanaZip CLI and Self Extracting
    Executables. (Except installer mode of Self Extracting Executables, which
    compiled binaries is not provided in the NanaZip MSIX package.)
- Various UI bug fixes and enhancements.

[7-Zip]: https://7-zip.org/
[7-Zip ZS]: https://github.com/mcmilk/7-Zip-zstd
[7-Zip NSIS]: https://github.com/myfreeer/7z-build-nsis

## Differences between NanaZip and NanaZip Classic

NanaZip plans to have two distribution flavors called NanaZip and NanaZip
Classic. Here are the differences between them.

- NanaZip (Modern)
  - Only MSIX packaged version.
  - Support the context menu in Windows 10/11 File Explorer.
  - Support the file associations.
  - Provide 7-Zip execution alias for helping users to migrate to NanaZip.
  - XAML-based GUI.

- NanaZip Classic (**Work In Progress**)
  - Only portable version which is suitable for Server Core, Windows PE, Windows
    RE, and Wine.
  - Don't have the context menu support.
  - Don't have the file associations support.
  - Don't have the 7-Zip execution alias support.
  - Win32 GUI.

> [!CAUTION]
> NanaZip can be used as portable version if you use the official portable
> release package. It's designed for debugging/testing/development purpose and
> scenarios (a.k.a. Server Core, Windows PE, Windows RE, and Wine) really need
> portable version. But please note that some features is not available, such as
> context menu and file associations. Some issues will not be fixed if you are
> using NanaZip in portable mode.

> [!CAUTION]
> We don't provide any support for users if they are using any non-official
> release packages, such as extracting the files from the MSIX package or copy
> from the NanaZipPackage project output. For example, if you are using the
> files extracted from the MSIX package of NanaZip like Scoop package manager
> does. We hugely recommend you to use the official release packages of NanaZip.
> For example, WinGet package manager uses NanaZip official release package to
> install NanaZip.

## System Requirements

Here are the system requirements starting with NanaZip 6.0:

- Supported Operating Systems:
  - Windows 10, version 2004 (Build 19041) or later
  - Windows Server 2022 (Build 20348) or later
- Supported Platforms:
  - x86 (64-bit)
  - ARM (64-bit)

In general, NanaZip follows the 2025 baseline defined by Kenji Mouri's document,
read [MD23: The baselines of Windows targets for all my open-source projects]
for more information.

[MD23: The baselines of Windows targets for all my open-source projects]: https://github.com/MouriNaruto/MouriDocs/tree/main/docs/23

Note: Due to the Self Extracting Executables (SFX) will be migrated to x86
(64-bit) starting with NanaZip 6.0, it gives users chances to use Self
Extracting Executables (SFX) archives under 64-bit Microsoft official Windows
PE and Windows RE instances:

- For ARM (64-bit) Microsoft official Windows PE and Windows RE, you need to
  make sure you Windows PE and Windows RE images are built with Windows ADK
  Build 25398 or later and integrated the [x64 emulation optional component].
- For ARM (64-bit) Windows 10, I suggest you to upgrade to Windows 11 because
  ARM (64-bit) Windows 11 provides x64 emulation support, and Windows 10 is
  nearly end of support from Microsoft.

[x64 emulation optional component]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-add-packages--optional-components-reference?#32

We chose Windows as the major platform for the NanaZip project supported
because Windows has maintained a good and proven ABI and compatibility, and
stands out in the history of computing. But we also care about the non-Windows
platform. Here are the policy for non-Windows platform support:

- The modern flavor of NanaZip won't support non-Windows unless someone ported
  WinRT XAML to POSIX. So, non-Windows support policy is only for NanaZip Core
  and NanaZip Classic.
- The developer team will try best to support Wine which has the Windows Vista
  or later feature level. Currenly tested simply under Wine 9.x.
- We don't support ReactOS and its derivatives because they are too hard to
  support. But testing under the ReactOS and its derivatives may be interesting.

> [!NOTE]
> In theory, NanaZip Classic can run on Windows 10 (Build 10240) or later if
> `ucrtbase.dll` in the `System32` folder has been replaced with version
> 10.0.19041.0 or later. However, this is a high-risk operation and is not
> recommended.

## Download and Installation

Here are some available installation methods for NanaZip.

### Microsoft Store

This is the recommended way to install NanaZip.

Search and install `NanaZip` in Windows Store for the stable release, and `NanaZip
Preview` for the preview release.

Also, you can also click the Microsoft Store link you needed.

- [NanaZip](https://www.microsoft.com/store/apps/9N8G7TSCL18R)
- [NanaZip Preview](https://www.microsoft.com/store/apps/9NZL0LRP1BNL)

### MSIX Package

You can download the MSIX package and the corresponding XML license file from
[GitHub Releases](https://github.com/M2Team/NanaZip/releases). After you have
finished downloading the relevant files, you can use one of the methods
described below to install NanaZip:

> [!NOTE]
> All required dependencies are bundled with NanaZip's MSIX package, as users
> without access to the Microsoft Store may have difficulty obtaining them
> separately. This is intended to make NanaZip more robust and easier to deploy
> in a wide range of environments.

#### App Installer

According to [Install and update the App Installer - MSIX | Microsoft Learn], we
will know that the App Installer ships as part of Windows SKUs which can access
to the Microsoft Store, or you can follow the instructions in the document to
install the App Installer if your Windows SKUs doesn't have access to the
Microsoft Store.

If your environment has the App Installer, you can double click the MSIX package
to install NanaZip.

> [!CAUTION]
> If NanaZip is installed using this method without an Internet connection, you
> must connect to the Internet and launch it once so that Windows can acquire
> the required license. Otherwise, NanaZip may not launch properly.

[Install and update the App Installer - MSIX | Microsoft Learn]: https://learn.microsoft.com/en-us/windows/msix/app-installer/install-update-app-installer

#### Command Line (With license file)

If your downloaded NanaZip release provides the XML license file, you can use
the following command to install NanaZip for all users.

If you open the PowerShell as Administrator, you can execute:

> Add-AppxProvisionedPackage -Online -PackagePath `The path of the MSIX package` -LicensePath `The path of the XML license file`

If you open the Command Prompt as Administrator, you can execute:

> PowerShell -NoLogo -NoProfile -NonInteractive -InputFormat None -ExecutionPolicy Bypass Add-AppxProvisionedPackage -Online -PackagePath `The path of the MSIX package` -LicensePath `The path of the XML license file`

Or you can execute:

> DISM.exe /Online /Add-ProvisionedAppxPackage /PackagePath:`The path of the MSIX package` /LicensePath:`The path of the XML license file`

If you want to uninstall NanaZip if you installed it with this method, you can
use the following command to uninstall NanaZip.

If you open the PowerShell as Administrator, you can execute:

> Get-AppxPackage -Name *40174MouriNaruto.NanaZip* -AllUsers | Remove-AppxPackage -AllUsers -Confirm

#### Command Line (Without license file)

If your downloaded NanaZip release doesn't provide the XML license file, or you
don't want to install NanaZip for all users, you can use the following command
to install NanaZip for the current user.

If you open the PowerShell as Administrator, you can execute:

> Add-AppxPackage -DeferRegistrationWhenPackagesAreInUse -ForceUpdateFromAnyVersion -Path `The path of the MSIX package`

If you open the Command Prompt as Administrator, you can execute:

> PowerShell -NoLogo -NoProfile -NonInteractive -InputFormat None -ExecutionPolicy Bypass Add-AppxPackage -DeferRegistrationWhenPackagesAreInUse -ForceUpdateFromAnyVersion -Path `The path of the MSIX package`

> [!CAUTION]
> If NanaZip is installed using this method without an Internet connection, you
> must connect to the Internet and launch it once so that Windows can acquire
> the required license. Otherwise, NanaZip may not launch properly.

#### Useful Links

- https://learn.microsoft.com/en-us/powershell/module/appx/add-appxpackage
- https://learn.microsoft.com/en-us/powershell/module/dism/add-appxprovisionedpackage
- https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism-app-package--appx-or-appxbundle--servicing-command-line-options

## Known issues

- If you can't find NanaZip in the context menu, please restart all File
  Explorer processes via Task Manager.
- Due to the design of MSIX, drives' context menu in Explorer only show in
  Windows 11, version 22H2 or later.
- Due to the Microsoft Store limitations, NanaZip 1.2 and later won't support
  languages not mentioned in
  https://docs.microsoft.com/en-us/windows/uwp/publish/supported-languages.
- If you turn off the Windows Firewall, NanaZip may fail to install.
  (https://github.com/M2Team/NanaZip/issues/204)
  (https://github.com/microsoft/terminal/issues/12269)
- Due to the System Settings limitations, starting with Windows 11+
  (Build 22000.1817+ & 22621.1555+), you can launch directly to the Settings
  page for file association for NanaZip.
  (https://learn.microsoft.com/en-us/windows/uwp/launch-resume/launch-default-apps-settings)
- Due to the Microsoft Store Licensing API implementation, you will find NanaZip
  will try to access the Internet beginning with NanaZip 3.0 because NanaZip
  needs to use that to check the Sponsor Edition addon licensing status.
- For optimizing the user experience, NanaZip will only check the Sponsor
  Edition addon licensing status at the first time your launch NanaZip File
  Manager. You can click the sponsor button if you want to refresh the status
  after you have purchased or redeemed the Sponsor Edition addon.

## Documents

- [License](License.md)
- [Relevant People](Documents/People.md)
- [Privacy Policy](Documents/Privacy.md)
- [Code of Conduct](CODE_OF_CONDUCT.md)
- [Contributing Guide](CONTRIBUTING.md)
- [NanaZip Sponsor Edition](Documents/SponsorEdition.md)
- [NanaZip Release Notes](Documents/ReleaseNotes.md)
- [NanaZip Preview Release Notes](Documents/ReleaseNotesPreview.md)
- [NanaZip Upstream Synchronization Status](Documents/UpstreamSynchronization.md)
- [NanaZip Group Policy Administrative Templates (ADMX/ADML)](Documents/PolicyDefinitions)
- [NanaZip Accessibility Conformance Report for Section 508](Documents/Section508)
- [Security Policy](Security.md)
- [Versioning](Documents/Versioning.md)
- [My Digital Life Forums](https://forums.mydigitallife.net/threads/84171)
