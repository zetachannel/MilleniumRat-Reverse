# Millenium RAT — By sirius

**By sirius**  
**Discord : zetachannel**  
**24/09/06 01:33**

---

## yo, contexte rapide

Bon. J'ai eu l'occase de reverse **Millenium RAT** (certains l'appellent Millenium Stealer ou Millenium Grabber). C'est un infostealer Windows, codé en C++, build MSVC. Y fait le tour classique : navigateurs, Discord, Telegram, wallets, screens, webcam, tout le tralala.

J'publie ça pour que les gens voient ce que ça fait vraiment, comment c'est foutu, et surtout pour que si vous tombez sur un truc chelou vous sachiez à quoi vous avez affaire. Le binôme est **pas packé** au niveau de l'entry point, donc l'analyse statique passe crème.

Petit rappel avant de continuer, parce qu'il faut le dire :

> ⚠️ **J'suis pas responsable de ce que vous faites avec ça.**  
> Analyse faite en VM isolée. **Aucun binaire malveillant dans ce repo.**  
> Si t'es infecté : coupe le net, scan, change tes mdp depuis un autre appareil, check tes sessions Discord/Telegram/nav/wallets.

---

## 🔍 Le PE en gros

| Truc | Valeur |
|---|---|
| Archi | x86 (32 bits) |
| ImageBase | `0x00400000` |
| Compilo | MSVC (Visual Studio) |
| Packing | Nan, pas au niveau de l'entry |
| Sections | `.text`, `.rdata`, `.data`, `.rsrc`, `.reloc` |
| Build | ~2025, assez récent |

### L'entry point sous Ghidra

```c
void entry(void)
{
  ___security_init_cookie();
  FUN_005ca76b();
  return;
}
```

Rien de fou, c'est le **CRT standard MSVC** :
- `___security_init_cookie()` → cookie de sécu (`/GS`), classique
- `FUN_005ca76b` → c'est `__scrt_common_main_seh()`, le wrapper qui appelle `main`

Donc si tu cherches le vrai code, c'est **pas dans entry**, c'est dans `main` (et tout ce qu'il appelle). Va falloir suivre les xrefs.

### Libs embarquées

- `libcurl 8.10.0-DEV`
- `SQLite3`
- `nlohmann/json 3.12.0`
- `easywsclient` (pour les WebSockets)
- `Gdiplus` + `AVICAP32` (screens + webcam)
- La clique habituelle : `BCrypt`, `NCrypt`, `Crypt32`, `Advapi32`, `User32`, `GDI32`, `Shell32`, `Ole32`, `OleAut32`, `Shlwapi`, `Ws2_32`, `Wldap32`, `Iphlpapi`, `Kernel32`, `Ntdll`, `Mscoree`

---

## 🎯 Ce qu'il vole

### 🧭 Navigateurs (Chrome, Edge, Brave, Yandex, Opera, Opera GX, Firefox)

Le grand classique. Il tape dans les bases SQLite et sort tout :

```sql
SELECT host_key, name, path, encrypted_value, expires_utc, is_secure, is_httponly FROM cookies
SELECT action_url, username_value, password_value FROM logins
SELECT name_on_card, expiration_month, expiration_year, card_number_encrypted FROM credit_cards
SELECT tab_url, target_path FROM downloads
SELECT name, value FROM autofill
SELECT company_name, street_address, city, state, zipcode, country_code FROM autofill_profiles
SELECT first_name, middle_name, last_name FROM autofill_profile_names
SELECT email FROM autofill_profile_emails
SELECT number FROM autofill_profile_phones
SELECT url FROM urls
SELECT url FROM moz_places WHERE url IS NOT NULL
SELECT host, path, name, value, isSecure, isHttpOnly, expiry FROM moz_cookies
```

Cookies, mdp, cartes bancaires, autofill, historique, downloads... bref la totale.

Les chemins qu'il vise :
```
\Google\Chrome\User Data (Default + Profiles 1 à 5)
\Google\Chrome SxS\User Data
\BraveSoftware\Brave-Browser\User Data
\Yandex\YandexBrowser\User Data
\Chromium\User Data
\Opera Software\Opera GX Stable
\Microsoft\Edge\User Data
\Mozilla\Firefox\Profiles
\Local State
\Network\Cookies
\History
\Web Data
\Login Data
\cookies.sqlite
\places.sqlite
```

Et y sort ça dans des ptits fichiers bien rangés :
```
BrowserData/cookies/Chrome [Default].txt
BrowserData/cookies/Edge [Default].txt
BrowserData/cookies/Brave [Default].txt
BrowserData/cookies/Yandex [Default].txt
BrowserData/cookies/Opera [Default].txt
BrowserData/cookies/OperaGX [Default].txt
BrowserData/cookies/Firefox [Default].txt
BrowserData/BrowserDownloads.txt
BrowserData/BrowserAutoFills.txt
BrowserData/BrowserPasswords.txt
BrowserData/CreditCards.txt
BrowserHistory/Chrome [Default].txt
BrowserHistory/Edge [Default].txt
BrowserHistory/Brave [Default].txt
BrowserHistory/Yandex [Default].txt
BrowserHistory/Opera [Default].txt
BrowserHistory/OperaGX [Default].txt
BrowserHistory/Firefox [Default].txt
```

### 💬 Discord

Il cherche les tokens dans les fichiers locaux, regex à l'appui :
- `[\w-]{24}\.[\w-]{6}\.[\w-]{25,110}`
- `mfa\.[\w-]{84}`
- `dQw4w9WgXcQ:([^.*\['(.*)'\].*$][^"]*)`

Il tape dans `\discord\Local State` et `\Local Storage\leveldb`, et il sort un `DiscordTokens.txt`. Les variantes qu'il connait : `discordcanary`, `discordptb`, `Lightcord`.

### ✈️ Telegram

Il zip le dossier `tdata` de Telegram Desktop :
```
\Telegram Desktop\tdata
C:\Program Files\Telegram Desktop\tdata
C:\Program Files (x86)\Telegram Desktop\
```
Et y choppe le chemin d'install via :
```
Software\Microsoft\Windows\CurrentVersion\Uninstall\{53F49750-6209-4FBF-9CA8-7A333C87D1ED}_is1
Inno Setup: App Path
```
Résultat : `TelegramData/tdata.zip`

### 💰 Wallets crypto

Là c'est du lourd, il cible un paquet d'extensions. Chaque wallet a son ID d'extension :

| Wallet | Extension ID |
|---|---|
| MetaMask | `nkbihfbeogaeaoehlefnkodbefgpgknn` |
| Binance | `fhbohimaelbohpjbbldcngcnapndodjp` |
| Phantom | `bfnaelmomeimhlpmgjnjophhpkkoljpa` |
| Coinbase | `hnfanknocfeofbddgcijnmhnfnkdnaad` |
| Ronin | `fnjhmkhhmkbjkkabndcnnogagogbneec` |
| Exodus_Browser | `aholpfdialjgjfhomihkjbmgjidlcdno` |
| Coin98 | `aeachknmefphepccionboohckonoeemg` |
| KardiaChain | `pdadjkfkgcafgbceimcpbkalnfnepbnk` |
| TerraStation | `aiifbnbfobpmeekipheeijimdpnlpgpp` |
| Wombat | `amkmjjmmflddogmhpjloimipbofnfjih` |
| Harmony | `fnnegphlobjdpkhecapkijjdkgcjhkib` |
| MartianAptos | `efbglgofoippbgcjepnhiblaibcnclgk` |
| Braavos | `jnlgamecbpmbajjfhmmmlhejkemejdma` |
| XDEFI | `hmeobnfnfcmdkdcmlblgagmfpfboieaf` |
| Yoroi | `ffnbelfdoeiohenkjibnmadjiehjhajb` |
| MetaMask_Edge | `ejbalbakoplchlghecdalmeeeajnimhm` |
| Trust | `egjidjbpglichdcondbcbdnbeeppgdph` |
| SafePal | `lgmpcpglpngdoalbgeoldeajfclnhafa` |
| Venom | `ojggmchlghnjlapmfbnjholfjkiidbch` |
| Keplr | `dmkamcknogkgcdfhhbddcghachkejeap` |
| ArgentX | `dlcobpjiigpikoobohmabehhmhfoodbb` |
| Rabby | `acmacodkjbdgmoleebolmdjonilkdbch` |
| Compass | `anokgmphncpekkhclmingpimjmcooifb` |
| Pontem | `phkbamefinggmakgklpkljjmgibohnba` |
| Soflare | `bhhhlbepdkbapadjdnnojkbgioiodbic` |
| Bitapp | `fihkakfobkmkjojpchpfgcmhfjnmnfpi` |
| BoltX | `aodkkagnadcbobfpggfnjeongemjbjca` |
| Crocobit | `pnlfjmlcjdjgkddecgincndfgegkecke` |
| Fewcha | `ebfidpplhabeedpnhjnobghokpiioolj` |
| Finnie | `cjmkndjhnagcfbpiemnkdpomccnjblmj` |
| Guarda | `hpglfhgfnhbgpjdenjgmdgoeiappafln` |
| Guild | `nanjmdknhkinifnkgdcggcfnhdaammmj` |
| Iconex | `flpiciilemghbmfalicajoolhkkenfel` |
| JaxxLiberty | `cjelfplplebdjjenllpjcblmjkfcffne` |
| Kaikas | `jblndlipeogpafnldhgmapagcccfchpi` |
| Liquality | `kpfopkelmapcoipemfendmdcghnegimn` |
| MEWCX | `nlbmnnijcnlegkjjpcfjclmcfggfefdm` |
| MaiarDEFI | `dngmlblcodfobpdpecaadgfbcggfjfnm` |
| Martian | `fcckkdbjnoikooededlapcalpionmalo` |
| Mobox | `jbdaocneiiinmjbjlgalhcelgbejmnid` |
| Nifty | `pdliaogehgdbhbnmkklieghmmjkpigpa` |
| Bybit | `fhilaheimglignddkjgofkcbgekhenbh` |
| Oxygen | `mgffkfbidihjpoaomajlbgchddlicgpn` |
| PaliWallet | `ejjladinnckdgjemekebdpeokbikhfci` |
| Petra | `nkddgncdjgjfcddamfgcmfnlhccnimig` |
| Saturn | `pocmplpaccanhmnllbbkpgfliimjljgo` |
| Slope | `fhmfendgdocmcbmfikdcogofphimnkno` |
| Sollet | `mfhbebgoclkghebffdldpobeajmbecfk` |
| Starcoin | `cmndjbecilbocjfkibfbifhngkdmjgog` |
| Swash | `ookjlbkiijinhpmnjffcofjonbfbgaoc` |
| TempleTezos | `eigblbgjknlfbajkfhopmcojidlgcehm` |
| XMR_PT | `bocpokimicclpaiekenaeelehdjllofo` |
| XinPay | `kncchdigobghenbbaddojjnnaogfppfj` |
| iWallet | `ookjlbkiijinhpmnjffcofjonbfbgaoc` |

Chemins visés :
```
\Default\Local Extension Settings
\Local Extension Settings
\Profile 
\Opera Software\Opera GX Stable\Local Extension Settings
```

Et pour **Exodus desktop** :
```
\Exodus
DesktopExodus/
DesktopExodus/Exodus.zip
```

### 📸 Screens + webcam

Via GDI+ et AVICAP32 :
- `GdiplusStartup`, `GdipCreateBitmapFromHBITMAP`, `GdipSaveImageToStream`, `GdipGetImageEncoders`, `GdipGetImageEncodersSize`, `GdipDisposeImage`, `GdipCloneImage`, `GdipFree`, `GdipAlloc`
- `capCreateCaptureWindowA` (AVICAP32.dll)

Fichiers :
```
WebPhoto.jpeg
Screenshots/
screenshot_
```

Strings associées :
```
CaptureWindow
Failed to create capture window
Failed to connect to webcam
No bitmap in clipboard
Failed to encode JPEG
```

### 📋 Presse-papiers

APIs classiques (`GetClipboardData`, `OpenClipboard`, `CloseClipboard`, `EmptyClipboard`), sortie dans `ClipBoard.txt`.

### 🧾 Process + infos système

Il liste les process (`CreateToolhelp32Snapshot`, `Process32First/Next`, `K32EnumProcesses`, `QueryFullProcessImageNameW`...) → `ProcessList.txt`

Et y récupère tout via WMI :
```sql
SELECT * FROM Win32_OperatingSystem
SELECT Name FROM Win32_Processor
SELECT Name FROM Win32_VideoController
SELECT TotalPhysicalMemory FROM Win32_ComputerSystem
SELECT ProcessorId FROM Win32_Processor
SELECT * FROM Win32_ComputerSystem
SELECT * FROM AntivirusProduct
```

Namespaces : `ROOT\CIMV2`, `\root\SecurityCenter2`

Sortie dans `PC_info.txt` :
```
Computer info:
System: 
Computer name: 
User name: 
System time: 
CPU: 
GPU: 
RAM: 
HWID: 
Security:
Installed antivirus: 
Started as admin: 
```

### 🗂️ Fichiers du bureau

Il zip le Desktop → `DesktopFiles/desktop.zip`

### 🌍 Géolocalisation IP

Via `http://ip-api.com/json/` :
```
Whois:
Country: 
City: 
Region: 
Internet provider: 
```

---

## 🛡️ Les techniques chelous

### Déchiffrement

- **DPAPI** : `CryptUnprotectData`, `CryptStringToBinary`
- **BCrypt / NCrypt** : `BCryptDecrypt`, `NCryptDecrypt`, `BCryptImportKey`, etc.
- **Chrome app-bound** avec clé hardcodée :
  ```
  E98F37D7F4E1FA433D19304DC2258042090E2D1D7EEA7670D41F738D08729660
  ```
- **Chrome DevTools Protocol** : il lance Chrome en headless avec `--remote-debugging-port=9222`, se co en WebSocket et demande `Network.getAllCookies`

Strings du DevTools :
```
" --headless --disable-gpu
--remote-debugging-port=9222 --user-data-dir="
" --profile-directory="
Failed to start browser process
Browser launched.
Failed to init curl
Curl failed
http://localhost:9222/json
webSocketDebuggerUrl
Requesting cookies...
Timeout reached while waiting for cookies.
Network.getAllCookies
{"id":
,"method":"Network.getAllCookies"}
```

Strings de déchiffrement :
```
Invalid validation blob
Invalid validation blob layout
Incomplete inner blob
Failed to init sodium.
Failed to open Local State
CryptStringToBinary failed
CryptUnprotectData failed
Failed to calculate Base64 decoded length
Failed to decode Base64
Failed to open key file.
encrypted_key
Encrypted key too short
os_crypt
app_bound_encrypted_key
ChainingModeGCM
ChainingModeCCM
ChainingMode
KeyDataBlob
ObjectLength
Microsoft Software Key Storage Provider
Google Chromekey1
```

### 🔐 Élévation de privilèges

Il active `SeDebugPrivilege`, ouvre `lsass.exe`, duplicate son token et s'impersonne pour choper SYSTEM.

```
SeDebugPrivilege
lsass.exe
OpenProcessToken failed
LookupPrivilegeValue failed
AdjustTokenPrivileges failed
EnumProcesses failed
lsass.exe not found
DuplicateToken failed
ImpersonateLoggedOnUser failed
```

APIs : `OpenProcessToken`, `LookupPrivilegeValueA`, `AdjustTokenPrivileges`, `OpenThreadToken`, `RevertToSelf`, `GetTokenInformation`, `SetThreadToken`, `DuplicateTokenEx`, `OpenProcess`, `EnumProcesses`.

### 🕵️ Anti-VM / anti-analyse

DLLs de sandbox qu'il check :
```
SbieDll.dll
SxIn.dll
Sf2.dll
snxhk.dll
cmdvrt32.dll
```

Détection VM :
```
vmware
virtualbox
VMware
microsoft corporation
virtual
WDAGUtilityAccount
```

Outils d'analyse qu'il cherche (et qui le font capoter) :
```
processhacker.exe
netstat.exe
netmon.exe
tcpview.exe
wireshark.exe
filemon.exe
regmon.exe
cain.exe
```

### 🔁 Persistance

Clé de registre :
```
Software\Microsoft\Windows\CurrentVersion\Run
```

APIs : `RegSetValueExW`, `RegCreateKeyExW`, `RegOpenKeyExW`, `RegQueryValueExW`, `RegDeleteValueW`, `RegDeleteKeyW`, `RegCloseKey`.

### ☠️ Auto-suppression

```
cmd /c timeout /t 3 /nobreak & rd /s /q "
cmd /c timeout /t 3 /nobreak & del /f /q "
 & rd /s /q "
```

---

## 🌐 C2 / Exfiltration

### 📡 Telegram Bot API

```
https://api.telegram.org/
/sendMessage?chat_id=
/sendDocument?chat_id=
/getMe
&text=
&caption=
/send
```

Fichier temporaire : `text.txt`

Strings associées :
```
Error occurred while sending the file
??Error occurred during file upload
??Error: 
The text was too long, so it was sent as a file
Document
```

### 📤 Gofile

```
https://upload.gofile.io/uploadfile
downloadPage
```

### 🐙 GitHub raw (config distante)

```
https://raw.githubusercontent.com/attatier/Cloud/main/MilInfo.txt
https://raw.githubusercontent.com/attatier/Cloud/main/Mil2.txt
```

### 🌍 IP Geolocation

```
http://ip-api.com/json/
```

### 🧭 Chrome DevTools

```
http://localhost:9222/json
```

---

## 🧩 APIs Windows utilisées (le paquet)

### Crypto / DPAPI
`CryptUnprotectData`, `CryptStringToBinary`, `CryptStringToBinaryA`, `BCryptDecrypt`, `BCryptImportKey`, `BCryptOpenAlgorithmProvider`, `BCryptCloseAlgorithmProvider`, `BCryptGenerateSymmetricKey`, `BCryptGenRandom`, `BCryptGetProperty`, `BCryptSetProperty`, `BCryptDestroyKey`, `NCryptOpenStorageProvider`, `NCryptOpenKey`, `NCryptFreeObject`, `NCryptDecrypt`, `CryptImportKey`, `CryptEncrypt`, `CryptCreateHash`, `CryptHashData`, `CryptAcquireContextA`, `CryptReleaseContext`, `CryptGetHashParam`, `CryptDestroyHash`, `CryptDestroyKey`, `CryptDecodeObjectEx`, `CryptQueryObject`, `SystemFunction036`

### Certificats
`CertOpenStore`, `CertCloseStore`, `CertEnumCertificatesInStore`, `CertFindCertificateInStore`, `CertFreeCertificateContext`, `PFXImportCertStore`, `CertAddCertificateContextToStore`, `CertFindExtension`, `CertGetNameStringA`, `CertCreateCertificateChainEngine`, `CertFreeCertificateChainEngine`, `CertGetCertificateChain`, `CertFreeCertificateChain`

### Privilèges / Tokens
`OpenProcessToken`, `LookupPrivilegeValueA`, `LookupPrivilegeValue`, `AdjustTokenPrivileges`, `DuplicateToken`, `DuplicateTokenEx`, `ImpersonateLoggedOnUser`, `OpenThreadToken`, `RevertToSelf`, `GetTokenInformation`, `SetThreadToken`

### Processus
`CreateToolhelp32Snapshot`, `Process32First/Next`, `Process32NextW`, `Process32FirstW`, `OpenProcess`, `TerminateProcess`, `K32EnumProcesses`, `K32GetModuleFileNameExW/A`, `QueryFullProcessImageNameW`, `CreateProcessW/A`, `ShellExecuteExW`

### Registre
`RegOpenKeyExW`, `RegQueryValueExW`, `RegSetValueExW`, `RegCreateKeyExW`, `RegDeleteValueW`, `RegDeleteKeyW`, `RegCloseKey`

### Fichiers
`CreateFileW/A/2`, `ReadFile`, `WriteFile`, `SetFilePointer(Ex)`, `SetEndOfFile`, `GetFileSize(Ex)`, `GetFileAttributesW/A/ExW`, `FindFirstFileW/ExW`, `FindNextFileW`, `FindClose`, `CreateDirectoryW`, `RemoveDirectoryW`, `CopyFileW`, `MoveFileExA`, `DeleteFileW/A`, `GetFullPathNameW/A`, `GetTempPathW/A/2W`, `GetCurrentDirectoryW`, `LockFile(Ex)`, `UnlockFile(Ex)`, `FlushFileBuffers`, `GetFileInformationByHandle(Ex)`, `CreateFileMappingW/A/FromApp`, `MapViewOfFile(FromApp)`, `UnmapViewOfFile`, `FlushViewOfFile`

### Système
`GetComputerNameW`, `GetUserNameW`, `GetSystemInfo`, `GetNativeSystemInfo`, `GetSystemTime`, `GetSystemTimeAsFileTime`, `GetSystemTimePreciseAsFileTime`, `SystemTimeToFileTime`, `FileTimeToSystemTime`, `FileTimeToLocalFileTime`, `GetTickCount(64)`, `QueryPerformanceCounter`, `QueryPerformanceFrequency`, `Sleep(Ex)`, `GetVersionExA/W`, `VerSetConditionMask`, `VerifyVersionInfoW`, `GetCurrentProcessId/ThreadId`, `GetCurrentProcess/Thread`, `GetProcessHeap`, `HeapCreate/Destroy/Alloc/Free/ReAlloc/Size/Validate/Compact`, `LocalFree`, `GlobalLock/Unlock`, `GetModuleHandleW/A/ExW`, `GetModuleFileNameW/A`, `LoadLibraryA/W/ExW/ExA/PackagedLibrary`, `FreeLibrary(AndExitThread)`, `GetProcAddress`

### Console / Debug
`GetConsoleMode`, `ReadConsoleW`, `WriteConsoleW`, `GetConsoleOutputCP`, `OutputDebugStringA/W`, `SetStdHandle`, `GetStdHandle`, `GetFileType`, `PeekNamedPipe`

### Localisation
`GetLocaleInfoEx/W`, `GetStringTypeW`, `GetDateFormatW/Ex`, `GetTimeFormatW/Ex`, `GetUserDefaultLCID`, `GetUserDefaultLocaleName`, `EnumSystemLocalesW/Ex`, `IsValidLocale(Name)`, `CompareStringW/Ex`, `LCMapStringW/Ex`, `LCIDToLocaleName`, `LocaleNameToLCID`, `GetTimeZoneInformation`, `IsValidCodePage`, `GetACP`, `GetOEMCP`, `GetCPInfo`, `MultiByteToWideChar`, `WideCharToMultiByte`, `GetCommandLineA/W`, `GetEnvironmentStringsW`, `FreeEnvironmentStringsW`, `SetEnvironmentVariableW`, `GetEnvironmentVariableA`, `GetDriveTypeW`

### Threading / Sync
`CreateThread`, `ExitThread`, `CreateEventA/ExW`, `SetEvent`, `WaitForSingleObject(Ex)`, `WaitForMultipleObjects`, `TryEnterCriticalSection`, `EnterCriticalSection`, `LeaveCriticalSection`, `InitializeCriticalSection(Ex/AndSpinCount)`, `DeleteCriticalSection`, `TryAcquireSRWLockExclusive`, `AcquireSRWLockExclusive`, `ReleaseSRWLockExclusive`, `InitializeSListHead`, `InitializeConditionVariable`, `WakeConditionVariable`, `WakeAllConditionVariable`, `SleepConditionVariableSRW`, `CreateThreadpoolWork`, `SubmitThreadpoolWork`, `CloseThreadpoolWork`, `FreeLibraryWhenCallbackReturns`, `InitOnceComplete`, `InitOnceBeginInitialize`

### Sécurité / Exception
`RtlUnwind`, `RaiseException`, `UnhandledExceptionFilter`, `SetUnhandledExceptionFilter`, `IsDebuggerPresent`, `GetStartupInfoW`, `EncodePointer`, `DecodePointer`, `TlsAlloc/Free/GetValue/SetValue`, `FlsAlloc/Free/GetValue/SetValue`

### Réseau
`WSAIoctl`, `WSACloseEvent`, `WSACreateEvent`, `WSAEnumNetworkEvents`, `WSAEventSelect`, `WSAResetEvent`, `WSAWaitForMultipleEvents`, `WS2_32.dll`, `WLDAP32.dll`, `freeaddrinfo`, `getaddrinfo`, `inet_pton`

### GDI+ / Webcam
`GdiplusStartup/Shutdown`, `GdipCreateBitmapFromHBITMAP`, `GdipSaveImageToStream`, `GdipGetImageEncoders(Size)`, `GdipDisposeImage`, `GdipCloneImage`, `GdipFree`, `GdipAlloc`, `capCreateCaptureWindowA`, `AVICAP32.dll`

### COM / OLE
`CoUninitialize`, `CoCreateInstance`, `CoSetProxyBlanket`, `CoInitializeSecurity`, `CoInitializeEx`, `CoTaskMemFree`, `CreateStreamOnHGlobal`, `ole32.dll`, `OLEAUT32.dll`

### Shell
`SHGetFolderPathW`, `ShellExecuteExW`, `SHGetKnownFolderPath`, `SHELL32.dll`, `StrStrIW`, `SHLWAPI.dll`, `SetProcessDpiAwareness`, `api-ms-win-shcore-scaling-l1-1-1.dll`

### User32 / GDI32
`MessageBoxW`, `ReleaseDC`, `GetClipboardData`, `CloseClipboard`, `GetMonitorInfoW`, `OpenClipboard`, `EnumDisplayMonitors`, `SendMessageA`, `EmptyClipboard`, `GetDC`, `DestroyWindow`, `USER32.dll`, `DeleteObject`, `DeleteDC`, `CreateDCW`, `CreateCompatibleDC`, `SelectObject`, `CreateCompatibleBitmap`, `BitBlt`, `GetObjectA`, `GetDIBits`, `GDI32.dll`

### Divers
`GetTempPath2W`, `kernel32.dll`, `ntdll.dll`, `mscoree.dll`, `CorExitProcess`, `advapi32.dll`, `kernelbase.dll`

---

## 📦 Artefacts créés (récap)

```
BrowserData/
BrowserData/cookies/
BrowserData/cookies/Chrome [Default].txt
BrowserData/cookies/Edge [Default].txt
BrowserData/cookies/Brave [Default].txt
BrowserData/cookies/Yandex [Default].txt
BrowserData/cookies/Opera [Default].txt
BrowserData/cookies/OperaGX [Default].txt
BrowserData/cookies/Firefox [Default].txt
BrowserData/BrowserDownloads.txt
BrowserData/BrowserAutoFills.txt
BrowserData/BrowserPasswords.txt
BrowserData/CreditCards.txt

BrowserHistory/
BrowserHistory/Chrome [Default].txt
BrowserHistory/Edge [Default].txt
BrowserHistory/Brave [Default].txt
BrowserHistory/Yandex [Default].txt
BrowserHistory/Opera [Default].txt
BrowserHistory/OperaGX [Default].txt
BrowserHistory/Firefox [Default].txt

ExtensionWallets/
DesktopExodus/
DesktopExodus/Exodus.zip

TelegramData/
TelegramData/tdata.zip

Screenshots/
screenshot_

DesktopFiles/
DesktopFiles/desktop.zip

DiscordTokens.txt
PC_info.txt
ProcessList.txt
ClipBoard.txt
WebPhoto.jpeg
log.zip
log.dot
text.txt
```

---

## 🔑 Autres strings notables (preuves)

### 🧪 Test / debug
```
testkeyloggerpath
test.key
JustATestKey
testMutexName
BasicFolder
testversion.exe
fE5RNoV378Z2KsG6KG4CmTXf6v5lpKSW
```

### 🕐 Timestamps / IDs
```
17360670233000000
2025-02-18 13:38:58 873d4e274b4988d260ba8354a9718324a1c26187a4ab4c1cc0227c03d0f10e70
961c151d2e87f2686a955a9be24d316f1362bf21 3.12.0
```

### 📝 Logs
```
p files|*|
Download link: 
, Location: 
NEW LOG (NOT ENCRYPTED)
Username: 
NEW LOG
Userna
NOEXIT
Software\
DELETE
```

### 📊 Rapport
```
------ Downloads ------
------ AutoFills ------
------ Passwords ------
------ Credit Cards ------
General information
Browser Data
Browser History
WebPhoto
Screenshots
ClipBoard
Process list:
Process List
Discord Tokens
Desktop Files
Telegram Data
Extension Wallets
Desktop Exodus
```

### ❌ Erreurs
```
Fail to schedule the chore!
This function cannot be called on a default constructed task
Failed to create capture window
Failed to connect to webcam
No bitmap in clipboard
Failed to encode JPEG
Failed to start browser process
Failed to init curl
Curl failed
Requesting cookies...
Timeout reached while waiting for cookies.
```

### 🏷️ RTTI / C++
```
.?AVWebSocket@easywsclient@@
.?AV_RealWebSocket@?A0xaa93fbb0@@
.?AUBytesCallback_Imp@easywsclient@@
.?AUCallback_Imp@easywsclient@@
.?AVlength_error@std@@
.?AVout_of_range@std@@
.?AVbad_function_call@std@@
.?AVregex_error@std@@
.?AVbad_exception@std@@
.?AV_com_error@@
.?AVbad_alloc@std@@
.?AVexception@std@@
.?AVbad_array_new_length@std@@
.?AVfailure@ios_base@std@@
.?AVruntime_error@std@@
.?AVsystem_error@std@@
.?AVbad_cast@std@@
.?AVfilesystem_error@filesystem@std@@
.?AV_System_error@std@@
.?AVtype_error@detail@json_abi_v3_12_0@nlohmann@@
.?AVother_error@detail@json_abi_v3_12_0@nlohmann@@
.?AVout_of_range@detail@json_abi_v3_12_0@nlohmann@@
.?AVparse_error@detail@json_abi_v3_12_0@nlohmann@@
.?AVlogic_error@std@@
.?AVfuture_error@std@@
.?AVinvalid_iterator@detail@json_abi_v3_12_0@nlohmann@@
.?AVexception@detail@json_abi_v3_12_0@nlohmann@@
.?AVrange_error@std@@
.?AVinvalid_argument@std@@
.?AVtask_canceled@Concurrency@@
.?AVGdiplusBase@Gdiplus@@
.?AVImage@Gdiplus@@
.?AVBitmap@Gdiplus@@
```

### 🥷 Anti-VM
```
SbieDll.dll
SxIn.dll
Sf2.dll
snxhk.dll
cmdvrt32.dll
vmware
virtualbox
VMware
microsoft corporation
virtual
WDAGUtilityAccount
```

### 💳 Wallets (extensions)
Déjà listés plus haut, mais ouais, y'en a un paquet. MetaMask, Binance, Phantom, Coinbase, Ronin, Exodus, etc.

### 📂 Chemins navigateurs
Déjà listés plus haut. Chrome, Edge, Brave, Yandex, Chromium, Opera, Opera GX, Firefox, avec tous les profils.

### 🌐 Exécutables navigateurs
```
C:\Program Files\Google\Chrome\Application\chrome.exe
C:\Program Files (x86)\Google\Chrome\Application\chrome.exe
\Google\Chrome\Application\chrome.exe
C:\Program Files\Microsoft\Edge\Application\msedge.exe
C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe
\Microsoft\Edge\Application\msedge.exe
C:\Program Files\BraveSoftware\Brave-Browser\Application\brave.exe
C:\Program Files (x86)\BraveSoftware\Brave-Browser\Application\brave.exe
\BraveSoftware\Brave-Browser\Application\brave.exe
SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\
SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\App Paths\
chrome.exe
msedge.exe
brave.exe
browser.exe
opera.exe
```

### 💬 Discord
```
\discord\Local State
\Local Storage\leveldb
DiscordTokens.txt
Discord Tokens
discordcanary
discordptb
Lightcord
dQw4w9WgXcQ:([^.*\['(.*)'\].*$][^"]*)
mfa\.[\w-]{84}
[\w-]{24}\.[\w-]{6}\.[\w-]{27,110}
APPDATA
Adiscord
```

### ✈️ Telegram
```
\Telegram Desktop\tdata
C:\Program Files\Telegram Desktop\tdata
C:\Program Files (x86)\Telegram Desktop\
Inno Setup: App Path
Software\Microsoft\Windows\CurrentVersion\Uninstall\{53F49750-6209-4FBF-9CA8-7A333C87D1ED}_is1
\tdata
Telegram.exe
working
user_data
emoji
dumps
tdummy
webview
market-history-cache.json
Partitions
Cache
undefined
Unknown
```

---

## 🧠 TL;DR

Millenium RAT, c'est un infostealer Windows bien ficelé, build 2025, qui :

- 🧭 Vol les **navigateurs** (cookies, mdp, cartes, autofill, historique)
- 💬 Chope les **tokens Discord**
- ✈️ Zip les **sessions Telegram**
- 💰 Braque les **wallets crypto** (extensions + Exodus)
- 📸 Prend des **screens** et allume la **webcam**
- 📋 Copie le **presse-papiers**
- 🧾 Liste les **process** et collecte les **infos système**
- 🗂️ Zip le **bureau**
- 🌍 Géolocalise l'IP

Niveau techniques :
- 🔓 Déchiffre via **DPAPI / BCrypt / NCrypt**
- 🧬 Utilise la **clé hardcodée Chrome app-bound**
- 🌐 Passe par le **Chrome DevTools Protocol** (WebSocket sur `localhost:9222`)
- 🔐 Monte en **SYSTEM** via `lsass.exe`
- 🥷 Détecte les **VM et outils d'analyse**
- 🔁 Persiste via `Run`
- ☠️ S'auto-supprime

Et il envoie tout ça sur **Telegram** + **Gofile**, avec une config récupérée depuis **GitHub raw**.

Le binaire est **pas packé**, donc si tu veux t'amuser, ouvre-le dans Ghidra, va dans `main` et suis les xrefs des strings que j'ai listées. Bon courage.

---

## 📬 Me contacter

Si t'as des questions, si tu veux discuter, ou juste m'ajouter :

- **Discord :** `zetachannel`

Hésite pas.

---

**By sirius**  
**Discord : zetachannel**  
**24/09/06 01:25**
