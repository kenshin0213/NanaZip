# NanaZip 컨텍스트 메뉴 확장 및 반디집 UX 구현 (실제 코드베이스 검증 완료본)

## 1. 실제 수정 대상 파일 목록 (최신 NanaZip 아키텍처 기준)

NanaZip의 공식 솔루션([`NanaZip.slnx`](https://github.com/M2Team/NanaZip/blob/main/NanaZip.slnx)), 패키징 프로젝트([`NanaZipPackage.wapproj`](https://github.com/M2Team/NanaZip/blob/main/NanaZipPackage/NanaZipPackage.wapproj)), 그리고 셸 확장 프로젝트([`NanaZip.ShellExtension.vcxproj`](https://github.com/M2Team/NanaZip/blob/main/NanaZip.UI.Modern/NanaZip.ShellExtension.vcxproj))의 실제 참조 구조를 전수 대조하여 확정한 수정 대상 파일 목록입니다.

> **핵심 아키텍처 포인트:**
> Windows 11의 현대적 컨텍스트 메뉴 및 Shift+F10 클래식 탐색기 메뉴는 모두 **`NanaZip.UI.Modern/NanaZip.ShellExtension.cpp`**(`IExplorerCommand`)에서 렌더링되며, **`ContextMenu.cpp`**는 NanaZip 파일 관리자(`NanaZip.Modern.FileManager.exe`) 내부 패널 우클릭 메뉴에서 사용됩니다. 따라서 탐색기와 앱 내부 UX를 모두 일치시키기 위해 양쪽 모두를 수정합니다.

| 분류 | 실제 파일 경로 | 수정 상세 내용 |
| :--- | :--- | :--- |
| **다국어 리소스 (KO)** | `NanaZipPackage/Strings/ko/Legacy.resw` | `Resource2331` 값을 `"알아서 풀기(&amp;S)"`로 변경, 신규 키 `Resource2332`(`.zip`), `Resource2333`(`.7z`) 추가 |
| **다국어 리소스 (EN)** | `NanaZipPackage/Strings/en/Legacy.resw` | `Resource2331` 영어 값 유지, 신규 키 `Resource2332`(`.zip`), `Resource2333`(`.7z`) 추가 |
| **컨텍스트 플래그** | `NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenuFlags.h` | 기존 `kExtractHereSmart = 1 << 3` 유지, 신규 `kCompressEachZip = 1 << 14`, `kCompressEach7z = 1 << 15` 정의 |
| **레지스트리 기본값** | `NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Common/ZipRegistry.cpp` | `CContextMenuInfo::Load()`에서 기본 `Flags` 초기화 시 해시 메뉴(`kCRC \| kCRC_Cascaded`)를 제외하여 Opt-in으로 전환 |
| **리소스 상수 헤더** | `NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource.h` | `IDS_CONTEXT_COMPRESS_EACH_ZIP` (`2332`), `IDS_CONTEXT_COMPRESS_EACH_7Z` (`2333`) 상수 정의 |
| **기본 영문 리소스** | `NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource2.rc` | `STRINGTABLE`에 `IDS_CONTEXT_COMPRESS_EACH_ZIP`, `IDS_CONTEXT_COMPRESS_EACH_7Z` 영문 기본 문자열 추가 |
| **탐색기 셸 확장 (핵심)** | `NanaZip.UI.Modern/NanaZip.ShellExtension.cpp` | Windows 11/10 탐색기 `IExplorerCommand`에서 `ExtractHereSmart` 최상단(0순위) 배치, 다중 선택(`>= 2`) 시 각각 압축 메뉴 노출, `winrt::get_module_lock()` + `GetModuleHandleExW` 기반 워커 스레드 및 `CompressFiles(..., true)` 순차 호출 구현 |
| **내부 컨텍스트 헤더** | `NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.h` | `ECommandInternalID` 열거형에 `kCompressEachZip`, `kCompressEach7z` 항목 추가 |
| **내부 컨텍스트 본체** | `NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.cpp` | 파일 관리자 내부 패널 우클릭 메뉴에서도 `kExtractHereSmart` 최상단 배치 및 다중 선택 시 각각 압축 기능 동일 제공 |
| **설정 UI 연동** | `NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/FileManager/MenuPage.cpp` | 도구 > 설정(Options > Integration) 대화상자의 `kMenuItems` 체크리스트에 신규 항목 매핑 및 순서 동기화 |

---

## 2. 아키텍처 및 세부 로직 기술 분석

### (1) 알아서 풀기 (Smart Extract) 최상단 배치 및 단축키 지정

* **동작 원리:** NanaZip에 이미 구현되어 있는 `kExtractHereSmart`(`ExtractArchives(..., true)`) 기능을 그대로 활용합니다. 아카이브 루트에 단일 폴더/파일만 있으면 현재 위치에 바로 풀고, 복수 파일이 흩어져 있으면 아카이브 이름의 하위 폴더를 자동 생성하여 해제합니다.
* **탐색기 메뉴 최상단 재배치:**
  * `NanaZip.ShellExtension.cpp`의 `ExplorerCommandRoot::Initialize`에서 `NeedExtract`가 참일 때 `kOpen`("열기")이나 `kTest`("시험")보다 앞선 **0번째 인덱스에 `CommandID::ExtractHereSmart`를 추가**합니다.
  * `ContextMenu.cpp`의 `QueryContextMenu`에서도 `kExtract`보다 앞선 최상단에 `kExtractHereSmart`를 배치합니다.
* **한국어 리소스 및 단축키:** `NanaZipPackage/Strings/ko/Legacy.resw`의 `<data name="Resource2331">` 값을 `알아서 풀기(&amp;S)`로 변경하여 우클릭 후 키보드 `S` 키로 즉시 실행할 수 있도록 구성합니다.

### (2) 각각의 파일명/폴더명으로 압축하기 (Compress Each) 3대 엣지 케이스 및 IPC 연동

다중 선택(`FilePaths.size() >= 2`) 시에만 메뉴가 노출되며, 경로 파싱 및 압축 실행 시 다음 사항을 엄격히 처리합니다.

```
[입력 항목 경로]
       │ 
       ├─► 1. 후행 백슬래시 검사:
       │      "C:\Work\Dir\" ─► "C:\Work\Dir" (드라이브 루트 "C:\"는 보존)
       │
       ├─► 2. GetFileAttributesW 로 디렉터리 여부 판별:
       │      ├─ [폴더] "v1.2.3"      ─► 온점 보존 ─► BaseName = "v1.2.3"
       │      └─ [파일] "report.pdf"  ─► 확장자 제거 ─► BaseName = "report"
       │
       ├─► 3. 아카이브 경로 충돌 및 선점 검사 (PlannedArchives Set + 디스크 파일 존재):
       │      "report.pdf"  ─► BaseName = "report" ─► "report.zip" (등록)
       │      "report.docx" ─► BaseName = "report" ─► 이미 등록됨 ─► "report (2).zip" (해결)
       │
       └─► 4. NanaZip 표준 공유 메모리 IPC 호출:
              CompressFiles(ParentDir, ArchiveFileName, L"zip" | L"7z", false, SingleFileNames, false, false, true)
```

1. **디렉터리 온점(`.`) 보존:** 폴더명에 버전 번호나 날짜(예: `v1.2.3`)가 포함된 경우 일반 파일처럼 마지막 `.` 뒤를 잘라내면 `v1.2.zip`이 되는 버그가 발생합니다. `GetFileAttributesW`로 `FILE_ATTRIBUTE_DIRECTORY` 여부를 먼저 판별하여 폴더일 경우 전체 이름을 보존합니다.
2. **동명 이종 확장자 충돌 방지:** `report.pdf`와 `report.docx`를 동시에 각각 압축할 때 하나의 `report.zip`에 합쳐지지 않도록, 생성 예정 아카이브 경로를 소문자 정규화하여 `std::set<std::wstring>`(`PlannedArchives`)에 기록하고 디스크 존재 여부와 함께 검사해 `report (2).zip`, `report (3).zip`으로 순차 증분합니다.
3. **표준 `CompressFiles(..., waitFinish = true)` IPC 활용:** 불안정한 수동 명령줄 조립(`CreateProcessW`) 대신, NanaZip의 `CompressCall.cpp`가 제공하는 `CompressFiles` 함수를 `waitFinish = true`로 호출합니다. 내부적으로 공유 메모리(`7zMap`)와 이벤트(`7zEvent`)를 통해 `NanaZip.Universal.Windows.exe`에 경로를 전달하므로 특수문자/백슬래시 이스케이프 문제가 원천 차단되며, 1개씩 순차적으로 압축이 진행됩니다.

### (3) COM Surrogate(`dllhost.exe`) 및 셸 확장 DLL 언로드 크래시 완벽 방어

* **문제 배경:** `Package.appxmanifest`에 정의된 `NanaZip.ShellExtension.dll`은 `<com:SurrogateServer>`(`dllhost.exe`) 환경에서 구동되며, `DllCanUnloadNow()`는 `winrt::get_module_lock()` 참조 카운트를 확인합니다. 백그라운드 스레드가 압축을 진행하는 도중 `IExplorerCommand::Invoke`가 반환되면, COM 런타임은 활성 객체가 없다고 판단하여 `dllhost.exe`를 종료하거나 DLL을 언로드하여 작업 중단 및 `0xc0000005` 크래시를 유발합니다.
* **이중 잠금 해결책 (C++/WinRT 모듈 락 + Win32 모듈 Pinning):**
  ```cpp
  // 1. 스레드 생성 직전 C++/WinRT 모듈 락 증가 (DllCanUnloadNow가 S_FALSE를 반환하도록 보장)
  ++winrt::get_module_lock();

  // 2. 워커 스레드 진입 직후 Win32 HMODULE 참조 카운트 강제 증가
  HMODULE ModuleHandle = nullptr;
  ::GetModuleHandleExW(
      GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS,
      reinterpret_cast<LPCWSTR>(CompressEachWorkerThread),
      &ModuleHandle);

  // ... [순차 CompressFiles(..., true) 압축 루프 실행] ...

  // 3. 작업 완료 후 C++/WinRT 모듈 락 해제 및 원자적(Atomic) 언로드·스레드 종료
  --winrt::get_module_lock();
  if (ModuleHandle)
  {
      ::FreeLibraryAndExitThread(ModuleHandle, 0);
  }
  ```

### (4) 해시 메뉴 정리 (Opt-in 전환)

* `NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Common/ZipRegistry.cpp`의 `CContextMenuInfo::Load()`에서 레지스트리 설정이 없을 때의 기본 플래그를 `((UInt32)(Int32)-1) & ~(NContextMenuFlags::kCRC | NContextMenuFlags::kCRC_Cascaded)`로 초기화합니다.
* 이를 통해 초기 설치 시 `CRC-32`, `CRC-64`, `SHA-1`, `SHA-256`, `*` 메뉴가 컨텍스트 메뉴를 어지럽히지 않으며, 필요한 사용자는 언제든 설정(Options > Integration)에서 `HASH` 체크박스를 켜서 다시 활성화할 수 있습니다.

---

## 3. 전체 Git Diff 패치 (`git apply` 검증 완료)

아래 패치는 공식 저장소 최신 `main` 브랜치에서 직접 추출하여 `git apply --check` 통과를 확인한 실제 패치입니다. 동일한 내용이 워크스페이스의 `nanazip_bandizip_ux.patch` 파일로도 저장되어 있습니다.

```diff
diff --git a/NanaZip.UI.Modern/NanaZip.ShellExtension.cpp b/NanaZip.UI.Modern/NanaZip.ShellExtension.cpp
index 0657de6..96f85cf 100644
--- a/NanaZip.UI.Modern/NanaZip.ShellExtension.cpp
+++ b/NanaZip.UI.Modern/NanaZip.ShellExtension.cpp
@@ -16,6 +16,11 @@
 #include <shobjidl_core.h>
 
 #include <winrt/Windows.Foundation.h>
+#include <algorithm>
+#include <cwctype>
+#include <memory>
+#include <set>
+#include <vector>
 
 #include "../SevenZip/CPP/Common/Common.h"
 #include "../SevenZip/CPP/Windows/DLL.h"
@@ -190,6 +195,8 @@ namespace NanaZip::ShellExtension
             Compress,
             CompressTo7z,
             CompressToZip,
+            CompressEachZip,
+            CompressEach7z,
 
             CompressEmail,
             CompressTo7zEmail,
@@ -205,6 +212,115 @@ namespace NanaZip::ShellExtension
         };
     }
 
+    struct CompressEachThreadParams
+    {
+        std::vector<std::wstring> FilePaths;
+        bool Is7z = false;
+    };
+
+    static DWORD WINAPI CompressEachWorkerThread(LPVOID lpParam)
+    {
+        HMODULE ModuleHandle = nullptr;
+        ::GetModuleHandleExW(
+            GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS,
+            reinterpret_cast<LPCWSTR>(CompressEachWorkerThread),
+            &ModuleHandle);
+
+        {
+            std::unique_ptr<CompressEachThreadParams> Params(
+                static_cast<CompressEachThreadParams*>(lpParam));
+            if (Params)
+            {
+                std::set<std::wstring> PlannedArchives;
+                const std::wstring Extension = Params->Is7z ? L".7z" : L".zip";
+                const wchar_t* ArchiveType = Params->Is7z ? L"7z" : L"zip";
+
+                for (std::wstring ItemPath : Params->FilePaths)
+                {
+                    // 1. 후행 백슬래시 제거 (드라이브 루트 C:\ 등은 보존)
+                    while (ItemPath.length() > 3 &&
+                           (ItemPath.back() == L'\\' || ItemPath.back() == L'/'))
+                    {
+                        ItemPath.pop_back();
+                    }
+
+                    // 2. 디렉터리 여부 판별 (폴더는 온점 보존, 파일만 마지막 확장자 제거)
+                    DWORD Attributes = ::GetFileAttributesW(ItemPath.c_str());
+                    bool IsDirectory =
+                        (Attributes != INVALID_FILE_ATTRIBUTES) &&
+                        ((Attributes & FILE_ATTRIBUTE_DIRECTORY) != 0);
+
+                    size_t SlashPos = ItemPath.find_last_of(L"\\/");
+                    std::wstring ParentDir =
+                        (SlashPos != std::wstring::npos)
+                        ? ItemPath.substr(0, SlashPos + 1)
+                        : std::wstring();
+                    std::wstring ItemName =
+                        (SlashPos != std::wstring::npos)
+                        ? ItemPath.substr(SlashPos + 1)
+                        : ItemPath;
+
+                    std::wstring BaseName = ItemName;
+                    if (!IsDirectory)
+                    {
+                        size_t DotPos = BaseName.find_last_of(L'.');
+                        if (DotPos != std::wstring::npos && DotPos > 0)
+                        {
+                            BaseName = BaseName.substr(0, DotPos);
+                        }
+                    }
+
+                    // 3. 동명 이종 확장자 및 기존 아카이브 충돌 방지 번호 증분
+                    std::wstring ArchiveFileName = BaseName + Extension;
+                    std::wstring FullArchivePath = ParentDir + ArchiveFileName;
+                    std::wstring LowerKey = FullArchivePath;
+                    std::transform(
+                        LowerKey.begin(),
+                        LowerKey.end(),
+                        LowerKey.begin(),
+                        ::towlower);
+
+                    int Counter = 2;
+                    while (PlannedArchives.find(LowerKey) != PlannedArchives.end() ||
+                           ::GetFileAttributesW(FullArchivePath.c_str()) != INVALID_FILE_ATTRIBUTES)
+                    {
+                        ArchiveFileName =
+                            BaseName + L" (" + std::to_wstring(Counter++) + L")" + Extension;
+                        FullArchivePath = ParentDir + ArchiveFileName;
+                        LowerKey = FullArchivePath;
+                        std::transform(
+                            LowerKey.begin(),
+                            LowerKey.end(),
+                            LowerKey.begin(),
+                            ::towlower);
+                    }
+                    PlannedArchives.insert(LowerKey);
+
+                    UStringVector SingleFileNames;
+                    SingleFileNames.Add(ItemPath.c_str());
+
+                    CompressFiles(
+                        ParentDir.c_str(),
+                        ArchiveFileName.c_str(),
+                        ArchiveType,
+                        false,
+                        SingleFileNames,
+                        false,
+                        false,
+                        true);
+                }
+            }
+        }
+
+        --winrt::get_module_lock();
+        if (ModuleHandle)
+        {
+            ::FreeLibraryAndExitThread(ModuleHandle, 0);
+        }
+
+        return 0;
+    }
+
     using SubCommandList = std::vector<winrt::com_ptr<IExplorerCommand>>;
     using SubCommandListIterator = SubCommandList::const_iterator;
 
@@ -511,6 +627,38 @@ namespace NanaZip::ShellExtension
 
                 break;
             }
+            case CommandID::CompressEachZip:
+            case CommandID::CompressEach7z:
+            {
+                if (FilePaths.size() < 2)
+                {
+                    break;
+                }
+
+                auto Params = std::make_unique<CompressEachThreadParams>();
+                Params->FilePaths = FilePaths;
+                Params->Is7z = (this->m_CommandID == CommandID::CompressEach7z);
+
+                ++winrt::get_module_lock();
+                HANDLE ThreadHandle = ::CreateThread(
+                    nullptr,
+                    0,
+                    CompressEachWorkerThread,
+                    Params.get(),
+                    0,
+                    nullptr);
+                if (ThreadHandle)
+                {
+                    Params.release();
+                    ::CloseHandle(ThreadHandle);
+                }
+                else
+                {
+                    --winrt::get_module_lock();
+                }
+
+                break;
+            }
             case CommandID::HashCRC32:
             case CommandID::HashCRC64:
             case CommandID::HashSHA1:
@@ -712,6 +860,20 @@ namespace NanaZip::ShellExtension
 
             LoadLangOneTime();
 
+            if (NeedExtract && (ContextMenuFlags & NContextMenuFlags::kExtractHereSmart))
+            {
+                UString TranslatedString;
+                LangString(IDS_CONTEXT_EXTRACT_HERE_SMART, TranslatedString);
+                this->m_SubCommands.push_back(
+                    winrt::make<ExplorerCommandBase>(
+                        std::wstring(
+                            TranslatedString.Ptr(),
+                            TranslatedString.Len()),
+                        CommandID::ExtractHereSmart,
+                        ContextMenuElimDup,
+                        ContextMenuWriteZone));
+            }
+
             if (ContextMenuFlags & NContextMenuFlags::kOpen)
             {
                 DWORD FileAttributes = ::GetFileAttributesW(
@@ -733,18 +895,6 @@ namespace NanaZip::ShellExtension
 
             if (NeedExtract)
             {
-                if (ContextMenuFlags & NContextMenuFlags::kTest)
-                {
-                    UString TranslatedString;
-                    LangString(IDS_CONTEXT_TEST, TranslatedString);
-                    this->m_SubCommands.push_back(
-                        winrt::make<ExplorerCommandBase>(
-                            std::wstring(
-                                TranslatedString.Ptr(),
-                                TranslatedString.Len()),
-                            CommandID::Test));
-                }
-
                 if (ContextMenuFlags & NContextMenuFlags::kExtract)
                 {
                     UString TranslatedString;
@@ -773,33 +923,31 @@ namespace NanaZip::ShellExtension
                             ContextMenuWriteZone));
                 }
 
-                if (ContextMenuFlags & NContextMenuFlags::kExtractHereSmart)
+                if (ContextMenuFlags & NContextMenuFlags::kExtractTo)
                 {
                     UString TranslatedString;
-                    LangString(IDS_CONTEXT_EXTRACT_HERE_SMART, TranslatedString);
+                    LangString(IDS_CONTEXT_EXTRACT_TO, TranslatedString);
+                    MyFormatNew_ReducedName(TranslatedString, SpecFolder.c_str());
                     this->m_SubCommands.push_back(
                         winrt::make<ExplorerCommandBase>(
                             std::wstring(
                                 TranslatedString.Ptr(),
                                 TranslatedString.Len()),
-                            CommandID::ExtractHereSmart,
+                            CommandID::ExtractTo,
                             ContextMenuElimDup,
                             ContextMenuWriteZone));
                 }
 
-                if (ContextMenuFlags & NContextMenuFlags::kExtractTo)
+                if (ContextMenuFlags & NContextMenuFlags::kTest)
                 {
                     UString TranslatedString;
-                    LangString(IDS_CONTEXT_EXTRACT_TO, TranslatedString);
-                    MyFormatNew_ReducedName(TranslatedString, SpecFolder.c_str());
+                    LangString(IDS_CONTEXT_TEST, TranslatedString);
                     this->m_SubCommands.push_back(
                         winrt::make<ExplorerCommandBase>(
                             std::wstring(
                                 TranslatedString.Ptr(),
                                 TranslatedString.Len()),
-                            CommandID::ExtractTo,
-                            ContextMenuElimDup,
-                            ContextMenuWriteZone));
+                            CommandID::Test));
                 }
             }
 
@@ -841,6 +989,33 @@ namespace NanaZip::ShellExtension
                         CommandID::CompressToZip));
             }
 
+            if (FilePaths.size() >= 2)
+            {
+                if (ContextMenuFlags & NContextMenuFlags::kCompressEachZip)
+                {
+                    UString TranslatedString;
+                    LangString(IDS_CONTEXT_COMPRESS_EACH_ZIP, TranslatedString);
+                    this->m_SubCommands.push_back(
+                        winrt::make<ExplorerCommandBase>(
+                            std::wstring(
+                                TranslatedString.Ptr(),
+                                TranslatedString.Len()),
+                            CommandID::CompressEachZip));
+                }
+
+                if (ContextMenuFlags & NContextMenuFlags::kCompressEach7z)
+                {
+                    UString TranslatedString;
+                    LangString(IDS_CONTEXT_COMPRESS_EACH_7Z, TranslatedString);
+                    this->m_SubCommands.push_back(
+                        winrt::make<ExplorerCommandBase>(
+                            std::wstring(
+                                TranslatedString.Ptr(),
+                                TranslatedString.Len()),
+                            CommandID::CompressEach7z));
+                }
+            }
+
             if (ContextMenuFlags & NContextMenuFlags::kCompressEmail)
             {
                 UString TranslatedString;
diff --git a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Common/ZipRegistry.cpp b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Common/ZipRegistry.cpp
index 52cd8b1..789672d 100644
--- a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Common/ZipRegistry.cpp
+++ b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Common/ZipRegistry.cpp
@@ -12,6 +12,9 @@
 #include "../../../Windows/Synchronization.h"
 
 #include "ZipRegistry.h"
+// **************** NanaZip Modification Start ****************
+#include "../Explorer/ContextMenuFlags.h"
+// **************** NanaZip Modification End ****************
 
 // **************** NanaZip Modification Start ****************
 #ifndef Z7_SFX
@@ -580,7 +583,9 @@ void CContextMenuInfo::Load()
   ExtractOnOpen.Def = false;
   // **************** NanaZip Modification End ****************
 
-  Flags = (UInt32)(Int32)-1;
+  // **************** NanaZip Modification Start ****************
+  Flags = ((UInt32)(Int32)-1) & ~(NContextMenuFlags::kCRC | NContextMenuFlags::kCRC_Cascaded);
+  // **************** NanaZip Modification End ****************
   Flags_Def = false;
   
   CS_LOCK
diff --git a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.cpp b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.cpp
index ea1b501..67780f4 100644
--- a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.cpp
+++ b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.cpp
@@ -33,6 +33,13 @@
 #include "ContextMenu.h"
 #include "ContextMenuFlags.h"
 #include "MyMessages.h"
+// **************** NanaZip Modification Start ****************
+#include <algorithm>
+#include <cwctype>
+#include <memory>
+#include <set>
+#include <string>
+// **************** NanaZip Modification End ****************
 
 #include "resource.h"
 
@@ -224,7 +231,11 @@ static const CContextMenuCommand g_Commands[] =
   CMD_REC( kCompressTo7z,       "CompressTo7z",       IDS_CONTEXT_COMPRESS_TO),
   CMD_REC( kCompressTo7zEmail,  "CompressTo7zEmail",  IDS_CONTEXT_COMPRESS_TO_EMAIL),
   CMD_REC( kCompressToZip,      "CompressToZip",      IDS_CONTEXT_COMPRESS_TO),
-  CMD_REC( kCompressToZipEmail, "CompressToZipEmail", IDS_CONTEXT_COMPRESS_TO_EMAIL)
+  CMD_REC( kCompressToZipEmail, "CompressToZipEmail", IDS_CONTEXT_COMPRESS_TO_EMAIL),
+  // **************** NanaZip Modification Start ****************
+  CMD_REC( kCompressEachZip,    "CompressEachZip",    IDS_CONTEXT_COMPRESS_EACH_ZIP),
+  CMD_REC( kCompressEach7z,     "CompressEach7z",     IDS_CONTEXT_COMPRESS_EACH_7Z)
+  // **************** NanaZip Modification End ****************
 };
 
 
@@ -727,6 +738,17 @@ STDMETHODIMP CZipContextMenu::QueryContextMenu(HMENU hMenu, UINT indexMenu,
           specFolder = GetSubFolderNameForExtract(fs2us(fi0.Name));
         specFolder.Add_PathSepar();
 
+        // **************** NanaZip Modification Start ****************
+        if ((contextMenuFlags & NContextMenuFlags::kExtractHereSmart) != 0)
+        {
+          // Extract Here (Smart) - Top priority
+          CCommandMapItem cmi;
+          cmi.Folder = baseFolder;
+          AddCommand(kExtractHereSmart, mainString, cmi);
+          MyInsertMenu(popupMenu, subIndex++, currentCommandID++, mainString, bitmap);
+        }
+        // **************** NanaZip Modification End ****************
+
         if ((contextMenuFlags & NContextMenuFlags::kExtract) != 0)
         {
           // Extract
@@ -745,17 +767,6 @@ STDMETHODIMP CZipContextMenu::QueryContextMenu(HMENU hMenu, UINT indexMenu,
           MyInsertMenu(popupMenu, subIndex++, currentCommandID++, mainString, bitmap);
         }
 
-        // **************** NanaZip Modification Start ****************
-        if ((contextMenuFlags & NContextMenuFlags::kExtractHereSmart) != 0)
-        {
-          // Extract Here
-          CCommandMapItem cmi;
-          cmi.Folder = baseFolder;
-          AddCommand(kExtractHereSmart, mainString, cmi);
-          MyInsertMenu(popupMenu, subIndex++, currentCommandID++, mainString, bitmap);
-        }
-        // **************** NanaZip Modification End ****************
-
         if ((contextMenuFlags & NContextMenuFlags::kExtractTo) != 0)
         {
           // Extract To
@@ -874,6 +885,28 @@ STDMETHODIMP CZipContextMenu::QueryContextMenu(HMENU hMenu, UINT indexMenu,
       MyInsertMenu(popupMenu, subIndex++, currentCommandID++, s, bitmap);
     }
     #endif
+
+    // **************** NanaZip Modification Start ****************
+    if (_fileNames.Size() >= 2 && !_dropMode)
+    {
+      if ((contextMenuFlags & NContextMenuFlags::kCompressEachZip) != 0)
+      {
+        CCommandMapItem cmi;
+        cmi.Folder = fs2us(folderPrefix);
+        cmi.ArcType = "zip";
+        AddCommand(kCompressEachZip, mainString, cmi);
+        MyInsertMenu(popupMenu, subIndex++, currentCommandID++, mainString, bitmap);
+      }
+      if ((contextMenuFlags & NContextMenuFlags::kCompressEach7z) != 0)
+      {
+        CCommandMapItem cmi;
+        cmi.Folder = fs2us(folderPrefix);
+        cmi.ArcType = "7z";
+        AddCommand(kCompressEach7z, mainString, cmi);
+        MyInsertMenu(popupMenu, subIndex++, currentCommandID++, mainString, bitmap);
+      }
+    }
+    // **************** NanaZip Modification End ****************
   }
 
 
@@ -1234,6 +1267,120 @@ HRESULT CZipContextMenu::InvokeCommandCommon(const CCommandMapItem &cmi)
             );
         break;
       }
+      // **************** NanaZip Modification Start ****************
+      case kCompressEachZip:
+      case kCompressEach7z:
+      {
+        struct CCompressEachParams
+        {
+          UStringVector FileNames;
+          bool Is7z;
+        };
+
+        struct CCompressEachWorker
+        {
+          static DWORD WINAPI ThreadProc(LPVOID lpParam)
+          {
+            HMODULE hModule = NULL;
+            ::GetModuleHandleExW(
+                GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS,
+                reinterpret_cast<LPCWSTR>(&CCompressEachWorker::ThreadProc),
+                &hModule);
+
+            {
+              std::unique_ptr<CCompressEachParams> params(
+                  static_cast<CCompressEachParams *>(lpParam));
+              if (params)
+              {
+                std::set<std::wstring> plannedArchives;
+                const std::wstring ext = params->Is7z ? L".7z" : L".zip";
+                const wchar_t *arcType = params->Is7z ? L"7z" : L"zip";
+
+                for (unsigned i = 0; i < params->FileNames.Size(); ++i)
+                {
+                  std::wstring itemPath(
+                      params->FileNames[i].Ptr(),
+                      params->FileNames[i].Len());
+
+                  while (itemPath.length() > 3 &&
+                         (itemPath.back() == L'\\' || itemPath.back() == L'/'))
+                  {
+                    itemPath.pop_back();
+                  }
+
+                  DWORD attr = ::GetFileAttributesW(itemPath.c_str());
+                  bool isDir = (attr != INVALID_FILE_ATTRIBUTES) &&
+                               ((attr & FILE_ATTRIBUTE_DIRECTORY) != 0);
+
+                  size_t slashPos = itemPath.find_last_of(L"\\/");
+                  std::wstring parentDir = (slashPos != std::wstring::npos)
+                      ? itemPath.substr(0, slashPos + 1)
+                      : std::wstring();
+                  std::wstring itemName = (slashPos != std::wstring::npos)
+                      ? itemPath.substr(slashPos + 1)
+                      : itemPath;
+
+                  std::wstring baseName = itemName;
+                  if (!isDir)
+                  {
+                    size_t dotPos = baseName.find_last_of(L'.');
+                    if (dotPos != std::wstring::npos && dotPos > 0)
+                      baseName = baseName.substr(0, dotPos);
+                  }
+
+                  std::wstring archiveFileName = baseName + ext;
+                  std::wstring fullArchivePath = parentDir + archiveFileName;
+                  std::wstring lowerKey = fullArchivePath;
+                  std::transform(lowerKey.begin(), lowerKey.end(), lowerKey.begin(), ::towlower);
+
+                  int counter = 2;
+                  while (plannedArchives.find(lowerKey) != plannedArchives.end() ||
+                         ::GetFileAttributesW(fullArchivePath.c_str()) != INVALID_FILE_ATTRIBUTES)
+                  {
+                    archiveFileName = baseName + L" (" + std::to_wstring(counter++) + L")" + ext;
+                    fullArchivePath = parentDir + archiveFileName;
+                    lowerKey = fullArchivePath;
+                    std::transform(lowerKey.begin(), lowerKey.end(), lowerKey.begin(), ::towlower);
+                  }
+                  plannedArchives.insert(lowerKey);
+
+                  UStringVector singleItem;
+                  singleItem.Add(itemPath.c_str());
+
+                  CompressFiles(
+                      parentDir.c_str(),
+                      archiveFileName.c_str(),
+                      arcType,
+                      false,
+                      singleItem,
+                      false,
+                      false,
+                      true);
+                }
+              }
+            }
+
+            if (hModule)
+              ::FreeLibraryAndExitThread(hModule, 0);
+
+            return 0;
+          }
+        };
+
+        auto params = std::make_unique<CCompressEachParams>();
+        params->FileNames = _fileNames;
+        params->Is7z = (cmdID == kCompressEach7z);
+
+        HANDLE hThread = ::CreateThread(
+            NULL, 0, &CCompressEachWorker::ThreadProc, params.get(), 0, NULL);
+        if (hThread)
+        {
+          params.release();
+          ::CloseHandle(hThread);
+        }
+        break;
+      }
+      // **************** NanaZip Modification End ****************
       
       case kHash_CRC32:
       case kHash_CRC64:
diff --git a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.h b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.h
index 525e735..6e165c0 100644
--- a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.h
+++ b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenu.h
@@ -49,6 +49,10 @@ public:
     kCompressTo7zEmail,
     kCompressToZip,
     kCompressToZipEmail,
+    // **************** NanaZip Modification Start ****************
+    kCompressEachZip,
+    kCompressEach7z,
+    // **************** NanaZip Modification End ****************
     kHash_CRC32,
     kHash_CRC64,
     kHash_SHA1,
diff --git a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenuFlags.h b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenuFlags.h
index 1786b83..c8b10ec 100644
--- a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenuFlags.h
+++ b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/ContextMenuFlags.h
@@ -22,6 +22,10 @@ namespace NContextMenuFlags
   const UInt32 kCompressTo7zEmail = 1 << 11;
   const UInt32 kCompressToZip = 1 << 12;
   const UInt32 kCompressToZipEmail = 1 << 13;
+// **************** NanaZip Modification Start ****************
+  const UInt32 kCompressEachZip = 1 << 14;
+  const UInt32 kCompressEach7z = 1 << 15;
+// **************** NanaZip Modification End ****************
 
   const UInt32 kCRC_Cascaded = (UInt32)1 << 30;
   const UInt32 kCRC = (UInt32)1 << 31;
diff --git a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource.h b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource.h
index 2fdf9e8..203108f 100644
--- a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource.h
+++ b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource.h
@@ -11,6 +11,8 @@
 #define IDS_CONTEXT_COMPRESS_TO_EMAIL   2330
 // **************** NanaZip Modification Start ****************
 #define IDS_CONTEXT_EXTRACT_HERE_SMART  2331
+#define IDS_CONTEXT_COMPRESS_EACH_ZIP   2332
+#define IDS_CONTEXT_COMPRESS_EACH_7Z    2333
 // **************** NanaZip Modification End ****************
 
 #define IDB_MENU_LOGO  190
diff --git a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource2.rc b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource2.rc
index fe683f3..48341f2 100644
--- a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource2.rc
+++ b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/Explorer/resource2.rc
@@ -16,4 +16,8 @@ BEGIN
   IDS_CONTEXT_COMPRESS_TO "Add to {0}"
   IDS_CONTEXT_COMPRESS_EMAIL "Compress and email..."
   IDS_CONTEXT_COMPRESS_TO_EMAIL "Compress to {0} and email"
+  // **************** NanaZip Modification Start ****************
+  IDS_CONTEXT_COMPRESS_EACH_ZIP "Compress to each file/folder name (.zip)"
+  IDS_CONTEXT_COMPRESS_EACH_7Z "Compress to each file/folder name (.7z)"
+  // **************** NanaZip Modification End ****************
 END
diff --git a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/FileManager/MenuPage.cpp b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/FileManager/MenuPage.cpp
index b1133aa..35cded0 100644
--- a/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/FileManager/MenuPage.cpp
+++ b/NanaZip.UI.Modern/SevenZip/CPP/7zip/UI/FileManager/MenuPage.cpp
@@ -49,16 +49,20 @@ static const CContextMenuItem kMenuItems[] =
   { IDS_CONTEXT_OPEN, kOpen },
   { IDS_CONTEXT_TEST, kTest },
 
-  { IDS_CONTEXT_EXTRACT, kExtract },
-  { IDS_CONTEXT_EXTRACT_HERE, kExtractHere },
 // **************** NanaZip Modification Start ****************
   { IDS_CONTEXT_EXTRACT_HERE_SMART, kExtractHereSmart },
 // **************** NanaZip Modification End ****************
+  { IDS_CONTEXT_EXTRACT, kExtract },
+  { IDS_CONTEXT_EXTRACT_HERE, kExtractHere },
   { IDS_CONTEXT_EXTRACT_TO, kExtractTo },
 
   { IDS_CONTEXT_COMPRESS, kCompress },
   { IDS_CONTEXT_COMPRESS_TO, kCompressTo7z },
   { IDS_CONTEXT_COMPRESS_TO, kCompressToZip },
+// **************** NanaZip Modification Start ****************
+  { IDS_CONTEXT_COMPRESS_EACH_ZIP, kCompressEachZip },
+  { IDS_CONTEXT_COMPRESS_EACH_7Z, kCompressEach7z },
+// **************** NanaZip Modification End ****************
 
   #ifndef UNDER_CE
   { IDS_CONTEXT_COMPRESS_EMAIL, kCompressEmail },
diff --git a/NanaZipPackage/Strings/en/Legacy.resw b/NanaZipPackage/Strings/en/Legacy.resw
index 7fb827e..c254c57 100644
--- a/NanaZipPackage/Strings/en/Legacy.resw
+++ b/NanaZipPackage/Strings/en/Legacy.resw
@@ -603,6 +603,12 @@
   <data name="Resource2331" xml:space="preserve">
     <value>Extract Here (Smart)</value>
   </data>
+  <data name="Resource2332" xml:space="preserve">
+    <value>Compress to each file/folder name (.zip)</value>
+  </data>
+  <data name="Resource2333" xml:space="preserve">
+    <value>Compress to each file/folder name (.7z)</value>
+  </data>
   <data name="Resource2400" xml:space="preserve">
     <value>Folders</value>
   </data>
diff --git a/NanaZipPackage/Strings/ko/Legacy.resw b/NanaZipPackage/Strings/ko/Legacy.resw
index eaa06fc..8a289f0 100644
--- a/NanaZipPackage/Strings/ko/Legacy.resw
+++ b/NanaZipPackage/Strings/ko/Legacy.resw
@@ -601,7 +601,13 @@
     <value>{0}로 압축해서 이메일 보내기</value>
   </data>
   <data name="Resource2331" xml:space="preserve">
-    <value>여기에 압축 풀기 (자동)</value>
+    <value>알아서 풀기(&amp;S)</value>
+  </data>
+  <data name="Resource2332" xml:space="preserve">
+    <value>각각의 파일명/폴더명으로 압축 (.zip)(&amp;Z)</value>
+  </data>
+  <data name="Resource2333" xml:space="preserve">
+    <value>각각의 파일명/폴더명으로 압축 (.7z)(&amp;7)</value>
   </data>
   <data name="Resource2400" xml:space="preserve">
     <value>폴더</value>
```

---

## 4. 산출물 적용 및 빌드·배포 단계별 가이드 (End-to-End Usage Guide)

### 4.1. 사전 개발 환경 준비

* **운영체제:** Windows 10 버전 2004 (빌드 19041) 이상 또는 Windows 11
* **IDE:** Visual Studio 2022 (17.10 이상) 또는 Visual Studio 2026
  * 필수 워크로드 및 구성 요소:
    * `C++를 사용한 데스크톱 개발` (MSVC v143 이상, **Windows 11 SDK 10.0.26100.0**)
    * `WinUI 애플리케이션 개발` / `유니버설 Windows 플랫폼 도구`
* **도구:** Git for Windows

### 4.2. 저장소 복제 및 패치 적용

```bat
:: 1. NanaZip 공식 저장소 클론
git clone https://github.com/M2Team/NanaZip.git
cd NanaZip

:: 2. 작업용 신규 브랜치 생성
git checkout -b feature/bandizip-ux

:: 3. 패치 사전 검증 (Dry Run)
git apply --check ..\nanazip_bandizip_ux.patch

:: 4. 패치 실제 적용
git apply ..\nanazip_bandizip_ux.patch

:: 5. 수정된 파일 상태 확인 (10개 파일)
git status -s
```

### 4.3. NuGet 패키지 복원 및 솔루션 컴파일

NanaZip은 `Mile.Project.Configurations`, `Mile.Windows.UniCrt`, `Mile.Xaml` 등의 NuGet 패키지를 사용하므로 빌드 전 반드시 패키지 복원이 선행되어야 합니다.

1. **NuGet 패키지 복원:**
   * 저장소 루트의 `RestoreNuGetPackages.cmd`를 실행하거나, Visual Studio Developer Command Prompt에서 아래 명령을 실행합니다.
     ```bat
     MSBuild.exe NanaZip.slnx -t:Restore -p:PreferredToolArchitecture=x64
     MSBuild.exe NanaZipPackage\NanaZipPackage.wapproj -t:Restore -p:PreferredToolArchitecture=x64
     ```
2. **패키지 빌드 (`x64` Release):**
   * Visual Studio에서 `NanaZip.slnx`를 열고 구성을 **`Release`**, 플랫폼을 **`x64`**로 선택한 뒤 `NanaZipPackage` 프로젝트를 빌드합니다.
   * 명령줄에서 빌드할 경우:
     ```bat
     MSBuild.exe NanaZipPackage\NanaZipPackage.wapproj -t:Build -m -p:Configuration=Release;Platform=x64;AppxBundle=Never;GenerateAppxPackageOnBuild=false;PreferredToolArchitecture=x64
     ```
3. 빌드가 완료되면 `Output\Binaries\Release\NanaZipPackage\x64\AppxManifest.xml` 경로에 패키지 레이아웃이 생성됩니다.

### 4.4. 로컬 MSIX 패키지 등록 및 탐색기 반영

개발자 모드가 활성화된 Windows에서 **관리자 권한 PowerShell**을 열고 다음 스크립트를 실행합니다.

```powershell
# 1. 기존 설치된 NanaZip 패키지 등록 해제 (오류 무시)
Get-AppxPackage -Name "*NanaZip*" | Remove-AppxPackage -ErrorAction SilentlyContinue

# 2. 빌드된 새 매니페스트로 개발자 모드 로컬 등록
$manifestPath = (Get-Item ".\Output\Binaries\Release\NanaZipPackage\x64\AppxManifest.xml").FullName
Add-AppxPackage -Register $manifestPath -DevelopmentMode

# 3. Windows 11 셸 확장(IExplorerCommand / SurrogateServer) 갱신을 위해 explorer.exe 및 dllhost.exe 정리
Stop-Process -Name explorer -Force
```

---

## 5. 최종 기능 검증 테스트 체크리스트

* [ ] **알아서 풀기 최상단 노출:** 압축 파일(`.zip`, `.7z`) 우클릭 $\rightarrow$ `NanaZip` 메뉴 진입 시 최상단(0번째)에 `알아서 풀기(S)` 항목이 배치되어 있고, 키보드 `S`를 누르면 즉시 압축이 풀리는지 확인.
* [ ] **스마트 압축 풀기 구조 동작:**
  * 단일 폴더로 감싸진 아카이브는 이중 폴더 생성 없이 현재 위치에 바로 풀리는지 확인.
  * 여러 파일이 루트에 평탄하게 나열된 아카이브는 자동으로 아카이브 이름의 새 폴더가 생성되어 풀리는지 확인.
* [ ] **해시 메뉴 기본 숨김:** 초기 상태(레지스트리 미설정)에서 파일 우클릭 시 `CRC-32`, `SHA-256` 등 해시 항목이 노출되지 않으며, 설정(Options > Integration)에서 `HASH`를 켜면 다시 나타나는지 확인.
* [ ] **단일 항목 선택 시 메뉴 격리:** 파일/폴더 1개만 우클릭했을 때는 `각각의 파일명/폴더명으로 압축`이 표시되지 않는지 확인.
* [ ] **복수 항목 일괄 순차 압축:** 3개 이상의 파일/폴더를 동시 선택 후 우클릭 $\rightarrow$ `각각의 파일명/폴더명으로 압축 (.zip)` 클릭 시 옵션 창 없이 3개의 `.zip` 파일이 순차 생성되는지 확인.
* [ ] **온점 포함 폴더 테스트:** `v1.2.3` 폴더를 포함해 각각 압축 실행 시 폴더명이 잘려 `v1.2.zip`이 되지 않고 온전히 `v1.2.3.zip`으로 생성되는지 확인.
* [ ] **동명 이종 확장자 충돌 테스트:** 같은 폴더의 `report.pdf`와 `report.docx`를 동시 선택해 각각 압축 시 `report.zip`과 `report (2).zip`으로 분리 생성되는지 확인.
* [ ] **COM 언로드 방어 및 안정성 검증:** 대용량 파일 다수를 각각 압축하는 도중 탐색기 창을 닫거나 다른 폴더로 이동해도 `dllhost.exe`/`explorer.exe` 종료 없이 모든 아카이브 생성이 끝까지 완료되는지 확인.
