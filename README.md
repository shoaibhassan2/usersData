Okay, this is a substantial task involving remote file operations, GUI updates, threading, and drag-and-drop. I will implement the requested features by modifying the provided source code files.

Here's the detailed plan and the modifications:

**Core Features to Implement:**

1.  **File Explorer:** The existing structure already has a good base for this (TreeView for directories, ListView for files).
2.  **File Upload (via Menu and Drag & Drop):**
    *   Add an "Upload" option to the File menu.
    *   Implement drag-and-drop of local files onto the File Explorer window to trigger an upload.
    *   Implement the network logic to send files to the remote client.
3.  **File Download (via Context Menu):**
    *   Add a "Download" option to the ListView context menu (for selected files).
    *   Implement the network logic to request and receive files from the remote client.
4.  **Progress Reporting:**
    *   Display upload/download progress in the File Explorer's status bar.
    *   Utilize the `FileProgressCallback` in `network_service.h` and new custom `WM_APP_FE_TRANSFER_PROGRESS` messages.

**Assumptions & Design Choices:**

*   **Remote File Operations:** The server (this application) will send commands (JSON) to the client, and the client will respond with data (JSON for listings, binary for file transfers).
*   **Progress UI:** A simple status bar update is chosen for progress reporting to avoid adding new dialogs and complex UI elements, keeping the changes focused.
*   **Drag & Drop (Download):** Dragging files *out* of the ListView for download is considerably more complex (requiring OLE drag-and-drop with virtual files). For simplicity and robustness within this project's scope, download will primarily be initiated via a context menu option, which is a common and user-friendly approach for remote file systems. The drag-and-drop implementation will focus on *uploading* files from the local system to the remote system.
*   **Concurrency:** File transfer operations will run on dedicated threads to avoid blocking the GUI. The results and progress will be posted back to the main GUI thread using custom `WM_APP` messages.
*   **Error Handling:** Basic error logging (via `PostLogMessage`) and status bar updates for network and file I/O errors.

---

**File Modifications:**

**1. `include/globals.h`**
   *   Add new custom window messages for file transfer progress and completion.
   *   Define a `FileTransferProgressInfo` struct to pass progress details to the GUI.

**2. `include/resource.h` & `res/resource.rc`**
   *   Add new menu item IDs and their corresponding string resources for "Upload" (menu) and "Download" (context menu).
   *   Add status bar message strings for various transfer states (uploading, downloading, complete, failed).

**3. `include/gui/file_explorer/fe_networking.h`**
   *   Update `RequestFileUpload` signature to take remote destination directory.
   *   Add `RequestFileDownload` signature.
   *   **Crucially:** Update `FileProgressCallback` in `include/service/network_service.h` to accept a `void* userData` parameter, which will be the `FileTransferProgressInfo` struct.

**4. `include/service/network_service.h`**
   *   Update `FileProgressCallback` signature.
   *   Update `SendFileRequest` and `ReceiveRequest` signatures to accept `void* callback_data`.

**5. `src/service/network_service.c`**
   *   Implement the updated `SendFileRequest` and `ReceiveRequest` functions to pass the `callback_data` to the `FileProgressCallback`. This allows the callback to contain context like the `HWND` of the file explorer and file name.

**6. `src/gui/file_explorer/file_explorer.c`**
   *   Add handlers for `WM_APP_FE_TRANSFER_PROGRESS` and `WM_APP_FE_TRANSFER_COMPLETE` messages. These handlers will update the status bar with transfer details.
   *   The `WM_DROPFILES` handler already exists and will utilize the updated `RequestFileUpload`.

**7. `src/gui/file_explorer/fe_events.c`**
   *   Modify `OnContextMenuCommand` to handle `IDM_FE_FILE_UPLOAD` (menu item) and `IDM_FE_CONTEXT_DOWNLOAD` (context menu item).
   *   The `OnFileDrop` function is already correctly calling `RequestFileUpload`.

**8. `src/gui/file_explorer/fe_ui.c`**
   *   Add "Upload" to the File menu.
   *   Add "Download" to the ListView context menu (only for files).

**9. `src/gui/file_explorer/fe_networking.c`**
   *   Create a new dedicated `FileTransferThread` and its associated `FileTransferThreadData` struct.
   *   Implement `RequestFileUpload` and `RequestFileDownload` to launch this new thread.
   *   Ensure the `FileTransferProgressCallback` correctly posts messages to the File Explorer window.
   *   The existing `GenericRequestThread` will continue to be used for JSON-only requests (like directory listings).

---

Let's apply these changes.

**--- Start of Changes ---**

**File: `include\globals.h`**

```c
// globals.h
#ifndef GLOBALS_H
#define GLOBALS_H
#include <winsock2.h>
#include <windows.h>
#include <server/server_logic.h>

// Custom message for thread-safe GUI updates
#define WM_APP_UPDATE_STATUS    (WM_APP + 1)
#define WM_APP_ADD_LOG          (WM_APP + 2)
#define WM_APP_ADD_CLIENT       (WM_APP + 3)
#define WM_APP_UPDATE_PROGRESS  (WM_APP + 4)
#define WM_APP_REMOVE_CLIENT    (WM_APP + 5)
// File Explorer specific messages (new and existing)
#define WM_APP_FE_UPDATE_DRIVES     (WM_APP + 10)
#define WM_APP_FE_UPDATE_LISTVIEW   (WM_APP + 11)
#define WM_APP_FE_UPDATE_TREE_CHILDREN (WM_APP + 12)
#define WM_APP_FE_REFRESH_VIEW      (WM_APP + 13)
#define WM_APP_FE_CLIENT_DISCONNECTED (WM_APP + 14)
#define WM_APP_FE_TRANSFER_PROGRESS (WM_APP + 15) // New: For file transfer progress updates
#define WM_APP_FE_TRANSFER_COMPLETE (WM_APP + 16) // New: For file transfer completion/failure

// Struct to hold handles for all controls on the server page
typedef struct {
    HWND hwndToolbar;
    HWND hwndGroupControls;
    HWND hwndStaticIP;
    HWND hwndIpAddress;
    HWND hwndStaticPort;
    HWND hwndEditPort;
    HWND hwndButtonAdd;
    HWND hwndGroupStats;
    HWND hwndStaticClients;
    HWND hwndStaticClientsVal;
    HWND hwndStaticReceived;
    HWND hwndStaticReceivedVal;

    HWND hwndStaticSent;
    HWND hwndStaticSentVal;
    HWND hwndListServers;
} ServerPageControls;

// Struct to hold handles for all controls on the client page
typedef struct {
    HWND hwndListClients;
} ClientPageControls;

// Struct to hold handles for all controls on the log page
typedef struct {
    HWND hwndListLog;
    HWND hwndButtonClear;
    HWND hwndCheckAutoscroll;
    HWND hwndButtonSave;
    HWND hwndButtonCopy;
} LogPageControls;

// Struct to hold settings of network configuration.
typedef struct {
    int max_connections;
    int connection_timeout;
    int send_buffer_kb;
    int recv_buffer_kb;
    BOOL limit_bandwidth;
    int max_upload_kbps;
    int max_download_kbps;
} NetworkSettings;

// New struct for passing file transfer progress information
typedef struct {
    long long current; // Current bytes transferred
    long long total;   // Total bytes for this transfer
    int clientIndex;   // Which client this progress belongs to
    BOOL isUpload;     // TRUE for upload, FALSE for download
    WCHAR fileName[MAX_PATH]; // Name of the file being transferred
    HWND hNotifyWnd;   // The window to post messages to (e.g., File Explorer HWND)
} FileTransferProgressInfo;

// --- Global Variables ---
extern HFONT g_hFont;
extern HINSTANCE g_hinst;
extern HWND g_hWnd;
extern HWND hwndTab;
extern HWND g_hTooltip;
extern BOOL g_bIsRunning;
extern WCHAR MAIN_WINDOW_TITLE[128];

// Extern declarations for the control structs
extern ServerPageControls g_server_controls;
extern ClientPageControls g_client_controls;
extern LogPageControls g_log_controls;

extern NetworkSettings g_networkSettings;

// For server/client management context menu
extern int g_serverContextMenuIndex;
extern int g_clientContextMenuIndex;

extern ActiveClient g_active_clients[MAX_CLIENTS];
extern int g_active_client_count;

// For the log page state
extern BOOL g_bLogAutoScroll;
extern BOOL isDarkMode;
extern CRITICAL_SECTION g_cs_servers;
extern CRITICAL_SECTION g_cs_clients;


#endif // GLOBALS_H
```

**File: `include\resource.h`**

```c
// include/resource.h
#ifndef RESOURCE_H
#define RESOURCE_H

// =======================================================
// 100-199: APP & WINDOW IDS
// =======================================================
#define IDS_MAIN_APP_TITLE                      100
#define IDS_MAIN_CLASS_NAME                     101
#define IDS_ABOUT_CLASSNAME                     102
#define IDS_NET_CLASSNAME                       103
#define IDS_TOOLS_CLASSNAME                     104
#define IDS_FILE_EXPLORER_CLASSNAME             105
#define IDS_SETTINGS_CLASSNAME                  106

#define IDC_TAB                                 150
#define IDC_PORT_EDIT                           151
#define IDC_STATIC_CLIENTS_VALUE                152
#define IDC_STATIC_RECEIVED_VALUE               153
#define IDC_STATIC_SENT_VALUE                   154

// =======================================================
// 200-299: TAB & PAGE STRINGS
// =======================================================
#define IDS_TAB_SERVER                          200
#define IDS_TAB_CLIENT                          201
#define IDS_TAB_LOG                             202
#define IDS_TOOLS_TAB_ICON_CHANGER              210
#define IDS_TOOLS_TAB_BUILDER                   211
#define IDS_TOOLS_TAB_BINDER                    212

// =======================================================
// 300-399: BUTTONS & LABELS
// =======================================================
#define IDS_BUTTON_SERVER_ADD                   300
#define IDS_LOG_BUTTON_CLEAR                    301
#define IDS_LOG_CHECKBOX_AUTOSCROLL             302
#define IDS_LOG_BUTTON_SAVE                     303
#define IDS_LOG_BUTTON_COPY                     304
#define IDS_NET_SAVE_BUTTON                     305
#define IDS_NET_CANCEL_BUTTON                   306
#define IDS_TOOLS_BROWSE_BUTTON                 307
#define IDS_TOOLS_CHANGE_ICON_BUTTON            308
#define IDS_TOOLS_EXTRACT_ICON_BUTTON           309
#define IDS_ABOUT_OK_BUTTON                     310
#define IDS_STATIC_SERVER_IPADDRESS             320
#define IDS_STATIC_SERVER_PORT                  321
#define IDS_STATIC_CLIENTS_CONNECTED            322
#define IDS_STATIC_BYTES_RECEIVED               323
#define IDS_STATIC_BYTES_SENT                   324
#define IDS_NET_LABEL_MAX_CONN                  330
#define IDS_NET_LABEL_TIMEOUT                   331
#define IDS_NET_LABEL_SENDBUFF                  332
#define IDS_NET_LABEL_RECVBUFF                  333
#define IDS_NET_LABEL_THROTTLE                  334
#define IDS_NET_LABEL_UPLOAD                    335
#define IDS_NET_LABEL_DOWNLOAD                  336
#define IDS_TOOLS_ICON_PATH_LABEL               340
#define IDS_TOOLS_EXE_PATH_LABEL                341
#define IDS_TOOLBAR_NETWORK                     350
#define IDS_TOOLBAR_TOOLS                       351
#define IDS_TOOLBAR_LOGS                        352
#define IDS_TOOLBAR_SETTINGS                    353
#define IDS_TOOLBAR_ABOUT                       354
#define IDS_SETTINGS_SHOW_HIDDEN                360

// =======================================================
// 400-499: GROUPBOXES & HEADERS
// =======================================================
#define IDS_GROUPBOX_SERVER_CONTROLS            400
#define IDS_GROUPBOX_SERVER_STATS               401
#define IDS_ABOUT_GROUPBOX_TITLE                402
#define IDS_NET_GROUP_CONN                      403
#define IDS_NET_GROUP_PERF                      404
#define IDS_NET_GROUP_THROTTLE                  405
#define IDS_TOOLS_ICON_GROUP                    406
#define IDS_TOOLS_EXTRACT_GROUP                 407

// =======================================================
// 500-599: LISTVIEW COLUMNS
// =======================================================
#define IDS_LV_COL_NAME                         500
#define IDS_LV_COL_TYPE                         501
#define IDS_LV_COL_SIZE                         502
#define IDS_LISTVIEW_SERVER_STATUS              503
#define IDS_LISTVIEW_SERVER_IPADDRESS           504
#define IDS_LISTVIEW_SERVER_PORT                505
#define IDS_LV_COL_MODIFIED                     506
#define IDS_LISTVIEW_CLIENT_IP                  510
#define IDS_LISTVIEW_CLIENT_COUNTRY             511
#define IDS_LISTVIEW_CLIENT_ID                  512
#define IDS_LISTVIEW_CLIENT_USERNAME            513
#define IDS_LISTVIEW_CLIENT_OS                  514
#define IDS_LISTVIEW_CLIENT_GROUP               515
#define IDS_LISTVIEW_CLIENT_DATE                516
#define IDS_LISTVIEW_CLIENT_UAC                 517
#define IDS_LISTVIEW_CLIENT_CPU                 518
#define IDS_LISTVIEW_CLIENT_GPU                 519
#define IDS_LISTVIEW_CLIENT_RAM                 520
#define IDS_LISTVIEW_CLIENT_ANTIVIRUS           521
#define IDS_LISTVIEW_CLIENT_ACTIVE_WINDOW       522
#define IDS_LOG_COLUMN_TIME                     530
#define IDS_LOG_COLUMN_MESSAGE                  531

// =======================================================
// 600-699: MENU ITEMS
// =======================================================
#define IDM_SERVER_START                        600
#define IDM_SERVER_STOP                         601
#define IDM_SERVER_REMOVE                       602
#define IDM_CLIENT_FILE_EXPLORER                610
#define IDM_CLIENT_DISCONNECT                   611
#define IDM_CLIENT_RESTART                      612
#define IDM_CLIENT_EXIT                         613
#define IDS_MENU_START_SERVER                   620
#define IDS_MENU_STOP_SERVER                    621
#define IDS_MENU_REMOVE_SERVER                  622


#define IDS_MENU_FILE_EXPLORER                  630
#define IDS_SUBMENU_CONNECTION                  631
#define IDS_MENU_DISCONNECT_CLIENT              632
#define IDS_MENU_RESTART_CONNECTION             633
#define IDS_MENU_EXIT_CLIENT                    634

// File Explorer Menus
#define IDM_FE_FILE_OPEN                        640
#define IDM_FE_FILE_UPLOAD                      641 // New: Upload menu item
#define IDM_FE_FILE_EXIT                        642 // Shifted ID
#define IDM_FE_VIEW_REFRESH                     643 // Shifted ID
#define IDM_FE_HELP_ABOUT                       644 // Shifted ID
#define IDM_FE_CONTEXT_OPEN                     650
#define IDM_FE_CONTEXT_DELETE                   651
#define IDM_FE_CONTEXT_RENAME                   652
#define IDM_FE_CONTEXT_DOWNLOAD                 653 // New: Download context menu item
#define IDS_FE_MENU_FILE                        660
#define IDS_FE_MENU_VIEW                        661
#define IDS_FE_MENU_HELP                        662
#define IDS_FE_MENU_OPEN                        663
#define IDS_FE_MENU_UPLOAD                      664 // New
#define IDS_FE_MENU_EXIT                        665 // Shifted ID
#define IDS_FE_MENU_REFRESH                     666 // Shifted ID
#define IDS_FE_MENU_ABOUT                       667 // Shifted ID
#define IDS_FE_CONTEXT_RENAME                   670
#define IDS_FE_CONTEXT_DELETE                   671
#define IDS_FE_CONTEXT_EXECUTE                  672
#define IDS_FE_CONTEXT_DOWNLOAD                 673 // New

// =======================================================
// 700-799: DIALOG WINDOW TEXT
// =======================================================
#define IDS_ABOUT_CAPTION                       700
#define IDS_ABOUT_TITLE                         701
#define IDS_ABOUT_AUTHOR                        702
#define IDS_ABOUT_DISCLAIMER_TEXT               703
#define IDS_ABOUT_SYSLINK_GITHUB                704
#define IDS_ABOUT_SYSLINK_TELEGRAM              705
#define IDS_NET_CAPTION                         710
#define IDS_TOOLS_CAPTION                       720
#define IDS_FILE_EXPLORER_CAPTION               730
#define IDS_SETTINGS_CAPTION                    740
#define IDS_FE_ABOUT_CAPTION                    745
#define IDS_FE_ABOUT_TEXT                       746
#define IDS_CONFIRM_EXEC_CAPTION                750
#define IDS_CONFIRM_EXEC_TEXT                   751
#define IDS_CONFIRM_DELETE_CAPTION              752
#define IDS_CONFIRM_DELETE_TEXT                 753
#define IDC_ABOUT_SYSLINK_GITHUB                760
#define IDC_ABOUT_SYSLINK_TELEGRAM              761
#define IDC_ABOUT_DISCLAIMER_EDIT               762
#define ID_ABOUT_OK_BTN                         763

// =======================================================
// 800-899: TOOLTIPS
// =======================================================
#define IDS_TOOLTIP_IPADDRESS                   800
#define IDS_TOOLTIP_PORT                        801
#define IDS_TOOLTIP_ADDSERVER                   802
#define IDS_TOOLTIP_SERVERLIST                  803
#define IDS_TOOLTIP_CLIENTLIST                  804
#define IDS_TOOLTIP_LOGLIST                     805
#define IDS_TOOLTIP_LOG_CLEAR                   806
#define IDS_TOOLTIP_LOG_AUTOSCROLL              807
#define IDS_TOOLTIP_LOG_SAVE                    808
#define IDS_TOOLTIP_LOG_COPY                    809
#define IDS_TOOLTIP_ABOUT_OK                    810
#define IDS_TOOLTIP_NET_MAXCONN                 811
#define IDS_TOOLTIP_NET_TIMEOUT                 812
#define IDS_TOOLTIP_NET_SENDBUFF                813
#define IDS_TOOLTIP_NET_RECVBUFF                814
#define IDS_TOOLTIP_NET_THROTTLE                815
#define IDS_TOOLTIP_NET_UPLOAD                  816
#define IDS_TOOLTIP_NET_DOWNLOAD                817
#define IDS_TOOLTIP_NET_SAVE                    818
#define IDS_TOOLTIP_NET_CANCEL                  819
#define IDS_TOOLTIP_PORT_EXCEED                 820

// =======================================================
// 900-999: ICONS, BITMAPS, CONTROL IDS
// =======================================================
#define IDI_ICON_MAIN                           900
#define IDB_WORLD                               901
#define IDB_INFO                                902
#define IDB_SETTINGS_ALT                        903
#define IDB_LOGS                                904
#define IDB_GITHUB                              905
#define IDB_TELEGRAM                            906
#define IDB_TOGGLE_ON                           907
#define IDB_TOGGLE_OFF                          908
#define IDB_TOOLS                               909
#define IDB_BACKWARD                            910
#define IDB_FORWARD                             911
#define IDB_SETTINGS                            912
#define IDC_BTN_WORLD                           920
#define IDC_BTN_INFO                            921
#define IDC_BTN_SETTINGS_ALT                    922
#define IDC_BTN_LOGS                            923
#define IDC_BTN_TOOLS                           924
#define IDC_BTN_ADD_SERVER                      925
#define IDC_LOG_LISTVIEW                        930
#define IDC_LOG_CLEAR_BUTTON                    931
#define IDC_LOG_AUTOSCROLL_CHECKBOX             932
#define IDC_LOG_SAVE_BUTTON                     933
#define IDC_LOG_COPY_BUTTON                     934
#define IDC_TOOLS_TAB                           940
#define IDC_TOOLS_EDIT_ICON                     941
#define IDC_TOOLS_BTN_ICON_BROWSE               942
#define IDC_TOOLS_EDIT_EXE                      943
#define IDC_TOOLS_BTN_EXE_BROWSE                944
#define IDC_TOOLS_BTN_CHANGE_ICON               945
#define IDC_TOOLS_BTN_EXTRACT_ICON              946
#define ID_NET_SAVE_BTN                         950
#define ID_NET_CANCEL_BTN                       951
#define IDC_NET_EDIT_MAX_CONN                   952
#define IDC_NET_EDIT_TIMEOUT                    953
#define IDC_NET_EDIT_SENDBUFF                   954
#define IDC_NET_EDIT_RECVBUFF                   955
#define IDC_NET_CHK_THROTTLE                    956
#define IDC_NET_EDIT_UPLOAD                     957
#define IDC_NET_EDIT_DOWNLOAD                   958
#define IDC_NET_STATIC_UPLOAD                   959
#define IDC_NET_STATIC_DOWNLOAD                 960
#define IDC_FE_TOOLBAR                          980
#define IDC_FE_ADDRESSBAR                       981
#define IDC_FE_TREEVIEW                         982
#define IDC_FE_LISTVIEW                         983
#define IDC_FE_STATUSBAR                        984
#define IDC_FE_NAV_BACK                         985
#define IDC_FE_NAV_FORWARD                      986
#define IDC_FE_SHOW_SETTINGS                    987
#define IDC_SETTINGS_CHECK_HIDDEN               988


// =======================================================
// 1000-1099: AUDIO RESOURCES
// =======================================================
#define IDR_WAVE_CONNECTED                      1000
#define IDR_WAVE_DISCONNECTED                   1001
#define IDR_WAVE_WELCOME                        1002
#define IDR_WAVE_CLOSED                         1003

// =======================================================
// 1100-1199: STATUS & LOG MESSAGES
// =======================================================
#define IDS_STATUS_LISTENING                    1100
#define IDS_STATUS_STARTING                     1101
#define IDS_STATUS_ERROR                        1102
#define IDS_STATUS_STOPPED                      1103
#define IDS_STATUS_UNKNOWN                      1104
#define IDS_UAC_ADMIN                           1105
#define IDS_UAC_USER                            1106
#define IDS_LOG_APP_STARTED                     1110
#define IDS_LOG_APP_CLOSING                     1111
#define IDS_LOG_LOG_CLEARED                     1112
#define IDS_LOG_AUTOSCROLL_ON                   1113
#define IDS_LOG_AUTOSCROLL_OFF                  1114
#define IDS_LOG_SETTINGS_CLICKED                1115
#define IDS_LOG_CLIENT_DISCONNECTED_BY_USER     1120
#define IDS_LOG_CLIENT_RESTART_CMD_SENT         1121
#define IDS_LOG_CLIENT_FILEEXPLORER_CMD_SENT    1122
#define IDS_FE_STATUS_LOADING                   1150
#define IDS_FE_STATUS_ERROR_PATH_CONV           1151
#define IDS_FE_STATUS_ERROR_NO_RESPONSE         1152
#define IDS_FE_STATUS_ERROR_INVALID_RESPONSE    1153
#define IDS_FE_STATUS_ERROR_SERVER_FAIL         1154
#define IDS_FE_STATUS_ITEMS_FMT                 1155
#define IDS_FE_STATUS_UPLOADING_FMT             1156 // New
#define IDS_FE_STATUS_DOWNLOADING_FMT           1157 // New
#define IDS_FE_STATUS_UPLOAD_COMPLETE_FMT       1158 // New
#define IDS_FE_STATUS_DOWNLOAD_COMPLETE_FMT     1159 // New
#define IDS_FE_STATUS_TRANSFER_FAIL_FMT         1160 // New


// =======================================================
// 1200-1299: ERROR & WARNING MESSAGES
// =======================================================
#define IDS_ERR_CAPTION                         1200
#define IDS_ERR_MAX_SERVERS_REACHED             1201
#define IDS_ERR_REGISTER_FAIL_MAIN              1202
#define IDS_ERR_CREATE_FAIL_MAIN                1203
#define IDS_ERR_REGISTER_FAIL_ABOUT             1204
#define IDS_ERR_REGISTER_FAIL_NET               1205
#define IDS_ERR_REGISTER_FAIL_TOOLS             1206
#define IDS_ERR_REGISTER_FAIL_FILE_EXPLORER     1207
#define IDS_TOOLS_MSG_ICON_SUCCESS              1220
#define IDS_TOOLS_MSG_ICON_FAIL                 1221
#define IDS_TOOLS_MSG_EMPTY_PATHS               1222
#define IDS_TOOLS_MSG_EXTRACT_SUCCESS           1223
#define IDS_TOOLS_MSG_EXTRACT_FAIL              1224

// =======================================================
// 1300-1399: REGISTRY & SETTINGS
// =======================================================
#define IDS_REG_KEY_PATH                        1300
#define IDS_REG_SUBKEY_SERVERS                  1301
#define IDS_REG_VALUE_IP                        1302
#define IDS_REG_VALUE_PORT                      1303
#define IDS_LOG_SERVERS_SAVED                   1304
#define IDS_LOG_SERVERS_LOADED                  1306
#define IDS_LOG_NO_SERVERS_FOUND                1307



// =======================================================
// 1400-1499: DIALOG TEMPLATES
// =======================================================
#define IDD_FE_SETTINGS                         1400




#endif // RESOURCE_H
```

**File: `res\resource.rc`**

```rc
// res/resource.rc

#include <windows.h>
#include <commctrl.h>

// Include the resource header file which defines all the control IDs
#include "..\\include\\resource.h"

VS_VERSION_INFO VERSIONINFO
 FILEVERSION     1,0,0,0
 PRODUCTVERSION  1,0,0,0
 FILEFLAGSMASK   0x3fL
#ifdef _DEBUG
 FILEFLAGS       0x1L
#else
 FILEFLAGS       0x0L
#endif
 FILEOS          VOS__WINDOWS32
 FILETYPE        VFT_APP
 FILESUBTYPE     VFT2_UNKNOWN
BEGIN
    BLOCK "StringFileInfo"
    BEGIN
        BLOCK "040904b0" // Language ID: U.S. English, Character Set: Unicode
        BEGIN
            VALUE "CompanyName",      "SHKA Tech\0"
            VALUE "FileDescription",  "BulletEye Server v1.0\0"
            VALUE "FileVersion",      "1.0.0.0\0"
            VALUE "InternalName",     "ServerGUI.exe\0"
            VALUE "LegalCopyright",   "Copyright (C) 2025. All rights reserved.\0"
            VALUE "OriginalFilename", "ServerGUI.exe\0"
            VALUE "ProductName",      "BulletEye Server\0"
            VALUE "ProductVersion",   "1.0\0"
        END
    END
    BLOCK "VarFileInfo"
    BEGIN
        VALUE "Translation", 0x409, 1200
    END
END
// --- End of VERSIONINFO block ---

IDD_FE_SETTINGS DIALOGEX 0, 0, 200, 80
STYLE DS_SETFONT | DS_MODALFRAME | DS_FIXEDSYS | WS_POPUP | WS_CAPTION | WS_SYSMENU
CAPTION "Settings"
FONT 8, "MS Shell Dlg", 400, 0, 0x1
BEGIN
    AUTOCHECKBOX    "Show hidden files and folders",IDC_SETTINGS_CHECK_HIDDEN,7,12,186,10
    DEFPUSHBUTTON   "OK",IDOK,45,59,50,14
    PUSHBUTTON      "Cancel",IDCANCEL,105,59,50,14
END



IDR_WAVE_CONNECTED              WAVE    "..\\assets\\Audio\\Connected.wav"
IDR_WAVE_DISCONNECTED           WAVE    "..\\assets\\Audio\\Disconnected.wav"
IDR_WAVE_WELCOME                WAVE    "..\\assets\\Audio\\Welcome.wav"
IDR_WAVE_CLOSED                 WAVE    "..\\assets\\Audio\\Closed.wav"
IDI_ICON_MAIN                   ICON    "..\\assets\\icon\\appicon.ico"
IDB_WORLD                       BITMAP  "..\\assets\\bitmap\\world.bmp"
IDB_INFO                        BITMAP  "..\\assets\\bitmap\\info.bmp"
IDB_SETTINGS_ALT                BITMAP  "..\\assets\\bitmap\\settings_alt.bmp"
IDB_LOGS                        BITMAP  "..\\assets\\bitmap\\logs.bmp"
IDB_GITHUB                      BITMAP  "..\\assets\\bitmap\\github.bmp"
IDB_TELEGRAM                    BITMAP  "..\\assets\\bitmap\\telegram.bmp"
IDB_TOGGLE_ON                   BITMAP  "..\\assets\\bitmap\\toggle_on.bmp"
IDB_TOGGLE_OFF                  BITMAP  "..\\assets\\bitmap\\toggle_off.bmp"
IDB_TOOLS                       BITMAP  "..\\assets\\bitmap\\tools.bmp"
IDB_BACKWARD                    BITMAP  "..\\assets\\bitmap\\backward.bmp"
IDB_FORWARD                     BITMAP  "..\\assets\\bitmap\\forward.bmp"
IDB_SETTINGS                    BITMAP  "..\\assets\\bitmap\\settings.bmp"



STRINGTABLE
BEGIN
    // =======================================================
    // 100-199: APP & WINDOW IDS
    // =======================================================
    IDS_MAIN_APP_TITLE                      L"BulletEye v1.0 Server"
    IDS_MAIN_CLASS_NAME                     L"MainServerWindowClass"
    IDS_ABOUT_CLASSNAME                     L"AboutWindowClass"
    IDS_NET_CLASSNAME                       L"NetworkSettingsClass"
    IDS_TOOLS_CLASSNAME                     L"ToolsWindowClass"
    IDS_FILE_EXPLORER_CLASSNAME             L"FileExplorerWindowClass"

    // =======================================================
    // 200-299: TAB & PAGE STRINGS
    // =======================================================
    IDS_TAB_SERVER                          L"Manage Server"
    IDS_TAB_CLIENT                          L"Connected Clients"
    IDS_TAB_LOG                             L"Logs"
    IDS_TOOLS_TAB_ICON_CHANGER              L"Icon Changer"
    IDS_TOOLS_TAB_BUILDER                   L"Builder"
    IDS_TOOLS_TAB_BINDER                    L"Binder"

    // =======================================================
    // 300-399: BUTTONS & LABELS
    // =======================================================
    IDS_BUTTON_SERVER_ADD                   L"Add Server"
    IDS_LOG_BUTTON_CLEAR                    L"Clear"
    IDS_LOG_CHECKBOX_AUTOSCROLL             L"Auto-scroll"
    IDS_LOG_BUTTON_SAVE                     L"Save..."
    IDS_LOG_BUTTON_COPY                     L"Copy"
    IDS_NET_SAVE_BUTTON                     L"Save"
    IDS_NET_CANCEL_BUTTON                   L"Cancel"
    IDS_TOOLS_BROWSE_BUTTON                 L"Browse..."
    IDS_TOOLS_CHANGE_ICON_BUTTON            L"Change Icon"
    IDS_TOOLS_EXTRACT_ICON_BUTTON           L"Extract Icon..."
    IDS_ABOUT_OK_BUTTON                     L"OK"
    IDS_STATIC_SERVER_IPADDRESS             L"IP Address:"
    IDS_STATIC_SERVER_PORT                  L"Port:"
    IDS_STATIC_CLIENTS_CONNECTED            L"Clients Connected:"
    IDS_STATIC_BYTES_RECEIVED               L"Bytes Received:"
    IDS_STATIC_BYTES_SENT                   L"Bytes Sent:"
    IDS_NET_LABEL_MAX_CONN                  L"Maximum Connections:"
    IDS_NET_LABEL_TIMEOUT                   L"Connection Timeout (sec):"
    IDS_NET_LABEL_SENDBUFF                  L"Send Buffer Size (KB):"
    IDS_NET_LABEL_RECVBUFF                  L"Receive Buffer Size (KB):"
    IDS_NET_LABEL_THROTTLE                  L"Limit Bandwidth Per Client"
    IDS_NET_LABEL_UPLOAD                    L"Max Upload Rate (KB/s):"
    IDS_NET_LABEL_DOWNLOAD                  L"Max Download Rate (KB/s):"
    IDS_TOOLS_ICON_PATH_LABEL               L"Icon Path (.ico):"
    IDS_TOOLS_EXE_PATH_LABEL                L"Exe Path (.exe):"
    IDS_TOOLBAR_NETWORK                     L"Network"
    IDS_TOOLBAR_TOOLS                       L"Tools"
    IDS_TOOLBAR_LOGS                        L"View Logs"
    IDS_TOOLBAR_SETTINGS                    L"Settings"
    IDS_TOOLBAR_ABOUT                       L"About"

    // =======================================================
    // 400-499: GROUPBOXES & HEADERS
    // =======================================================
    IDS_GROUPBOX_SERVER_CONTROLS            L"Server Controls"
    IDS_GROUPBOX_SERVER_STATS               L"Statistics"
    IDS_ABOUT_GROUPBOX_TITLE                L"Important Disclaimer & Warning"
    IDS_NET_GROUP_CONN                      L"Connection"
    IDS_NET_GROUP_PERF                      L"Performance & Buffering"
    IDS_NET_GROUP_THROTTLE                  L"Bandwidth Throttling"
    IDS_TOOLS_ICON_GROUP                    L"Change Executable Icon"
    IDS_TOOLS_EXTRACT_GROUP                 L"Extract Icon from Executable"

    // =======================================================
    // 500-599: LISTVIEW COLUMNS
    // =======================================================
    IDS_LISTVIEW_SERVER_STATUS              L"Status"
    IDS_LISTVIEW_SERVER_IPADDRESS           L"IP Address"
    IDS_LISTVIEW_SERVER_PORT                L"Port"
    IDS_LISTVIEW_CLIENT_IP                  L"IP"
    IDS_LISTVIEW_CLIENT_COUNTRY             L"Country"
    IDS_LISTVIEW_CLIENT_ID                  L"ID"
    IDS_LISTVIEW_CLIENT_USERNAME            L"Username"
    IDS_LISTVIEW_CLIENT_OS                  L"Operating System"
    IDS_LISTVIEW_CLIENT_GROUP               L"Group"
    IDS_LISTVIEW_CLIENT_DATE                L"Date"
    IDS_LISTVIEW_CLIENT_UAC                 L"UAC"
    IDS_LISTVIEW_CLIENT_CPU                 L"CPU"
    IDS_LISTVIEW_CLIENT_GPU                 L"GPU"
    IDS_LISTVIEW_CLIENT_RAM                 L"RAM"
    IDS_LISTVIEW_CLIENT_ANTIVIRUS           L"Antivirus"
    IDS_LISTVIEW_CLIENT_ACTIVE_WINDOW       L"Active Window"
    IDS_LOG_COLUMN_TIME                     L"Time"
    IDS_LOG_COLUMN_MESSAGE                  L"Message"

    // =======================================================
    // 600-699: MENU ITEMS
    // =======================================================
    IDS_MENU_START_SERVER                   L"Start Server"
    IDS_MENU_STOP_SERVER                    L"Stop Server"
    IDS_MENU_REMOVE_SERVER                  L"Remove Server"
    IDS_MENU_FILE_EXPLORER                  L"File Explorer"
    IDS_SUBMENU_CONNECTION                  L"Connection"
    IDS_MENU_DISCONNECT_CLIENT              L"Disconnect Client"
    IDS_MENU_RESTART_CONNECTION             L"Restart Connection"
    IDS_MENU_EXIT_CLIENT                    L"Shutdown Client"

    IDS_FE_MENU_FILE                        L"&File"
    IDS_FE_MENU_VIEW                        L"&View"
    IDS_FE_MENU_HELP                        L"&Help"
    IDS_FE_MENU_OPEN                        L"&Open"
    IDS_FE_MENU_UPLOAD                      L"&Upload..." // New
    IDS_FE_MENU_EXIT                        L"E&xit"
    IDS_FE_MENU_REFRESH                     L"&Refresh"
    IDS_FE_MENU_ABOUT                       L"&About"
    IDS_FE_CONTEXT_RENAME                   L"Re&name"
    IDS_FE_CONTEXT_DELETE                   L"&Delete"
    IDS_FE_CONTEXT_EXECUTE                  L"E&xecute"
    IDS_FE_CONTEXT_DOWNLOAD                 L"&Download" // New

    // =======================================================
    // 700-799: DIALOG WINDOW TEXT
    // =======================================================
    IDS_ABOUT_CAPTION                       L"About BulletEye Server"
    IDS_ABOUT_TITLE                         L"BulletEye Server v1.0"
    IDS_ABOUT_AUTHOR                        L"Copyright (C) 2025 by Shoaib Hassan."
    IDS_ABOUT_DISCLAIMER_TEXT               L"This application is designed for educational purposes ONLY, to demonstrate advanced C programming and networking concepts. \n\nAny use of this software for malicious activities, including but not limited to creating backdoors, unauthorized system access, or launching denial-of-service attacks, is strictly prohibited, unethical, and illegal. The author is not responsible for any misuse of this program."
    IDS_ABOUT_SYSLINK_GITHUB                L"For more, visit the <a>official GitHub repository</a>."
    IDS_ABOUT_SYSLINK_TELEGRAM              L"Contact the author on <a>Telegram</a>."
    IDS_NET_CAPTION                         L"Network Settings"
    IDS_TOOLS_CAPTION                       L"Extra Tools"
    IDS_FILE_EXPLORER_CAPTION               L"File Explorer"

    // =======================================================
    // 800-899: TOOLTIPS
    // =======================================================
    IDS_TOOLTIP_IPADDRESS                   L"Enter the IP address for the server to listen on. Use 127.0.0.1 for local connections."
    IDS_TOOLTIP_PORT                        L"Enter the port number (1-65535) for the server to listen on."
    IDS_TOOLTIP_ADDSERVER                   L"Start a new server instance with the specified IP and Port."
    IDS_TOOLTIP_SERVERLIST                  L"Displays the list of active server instances, their status, and connection details."
    IDS_TOOLTIP_CLIENTLIST                  L"Displays a list of all clients currently connected to your servers."
    IDS_TOOLTIP_LOGLIST                     L"Displays real-time logs, events, and errors from the application."
    IDS_TOOLTIP_LOG_CLEAR                   L"Clear all messages from the log view."
    IDS_TOOLTIP_LOG_AUTOSCROLL              L"When checked, automatically scrolls to the newest log message as it arrives."
    IDS_TOOLTIP_LOG_SAVE                    L"Save the entire log content to a text (.txt) file."
    IDS_TOOLTIP_LOG_COPY                    L"Copy the entire log content to the system clipboard."
    IDS_TOOLTIP_ABOUT_OK                    L"Close this window."
    IDS_TOOLTIP_NET_MAXCONN                 L"Set the maximum number of concurrent client connections allowed across all servers."
    IDS_TOOLTIP_NET_TIMEOUT                 L"Set the timeout in seconds. Connections will be dropped after this period of inactivity."
    IDS_TOOLTIP_NET_SENDBUFF                L"Set the kernel send buffer size in Kilobytes (KB) for each client socket."    IDS_TOOLTIP_NET_RECVBUFF                L"Set the kernel receive buffer size in Kilobytes (KB) for each client socket."
    IDS_TOOLTIP_NET_THROTTLE                L"Enable or disable bandwidth limiting for each connected client."
    IDS_TOOLTIP_NET_UPLOAD                  L"Set the maximum upload speed (server to client) per client in KB/s. Requires throttling to be enabled."
    IDS_TOOLTIP_NET_DOWNLOAD                L"Set the maximum download speed (client to server) per client in KB/s. Requires throttling to be enabled."
    IDS_TOOLTIP_NET_SAVE                    L"Save the current network settings and apply them."
    IDS_TOOLTIP_NET_CANCEL                  L"Discard any changes and close this window."
    IDS_TOOLTIP_PORT_EXCEED                 L"Port number cannot exceed 65535."

    // =======================================================
    // 1100-1199: STATUS & LOG MESSAGES
    // =======================================================
    IDS_STATUS_LISTENING                    L"Listening"
    IDS_STATUS_STARTING                     L"Starting..."
    IDS_STATUS_ERROR                        L"Error"
    IDS_STATUS_STOPPED                      L"Stopped"
    IDS_STATUS_UNKNOWN                      L"Unknown"
    IDS_UAC_ADMIN                           L"Admin"
    IDS_UAC_USER                            L"User"
    IDS_LOG_APP_STARTED                     L"Application started and main window created."
    IDS_LOG_APP_CLOSING                     L"Application closing..."
    IDS_LOG_LOG_CLEARED                     L"Log cleared by user."
    IDS_LOG_AUTOSCROLL_ON                   L"Auto-scroll enabled."
    IDS_LOG_AUTOSCROLL_OFF                  L"Auto-scroll disabled."
    IDS_LOG_SETTINGS_CLICKED                L"Settings button clicked (not implemented)."
    IDS_LOG_CLIENT_DISCONNECTED_BY_USER     L"Disconnected client: %hs"
    IDS_LOG_CLIENT_RESTART_CMD_SENT         L"Sent restart command to client: %hs"
    IDS_LOG_CLIENT_FILEEXPLORER_CMD_SENT    L"Sent file explorer command to client: %hs"
    IDS_FE_STATUS_LOADING                   L"Loading..."
    IDS_FE_STATUS_ERROR_PATH_CONV           L"Error: Path conversion failed"
    IDS_FE_STATUS_ERROR_NO_RESPONSE         L"Error: No response from server"
    IDS_FE_STATUS_ERROR_INVALID_RESPONSE    L"Error: Invalid server response"
    IDS_FE_STATUS_ERROR_SERVER_FAIL         L"Error: Server returned failure"
    IDS_FE_STATUS_ITEMS_FMT                 L"%ls - %d items"
    IDS_FE_STATUS_UPLOADING_FMT             L"Uploading %ls: %lld/%lld bytes (%.1f%%)"
    IDS_FE_STATUS_DOWNLOADING_FMT           L"Downloading %ls: %lld/%lld bytes (%.1f%%)"
    IDS_FE_STATUS_UPLOAD_COMPLETE_FMT       L"Upload of %ls complete!"
    IDS_FE_STATUS_DOWNLOAD_COMPLETE_FMT     L"Download of %ls complete!"
    IDS_FE_STATUS_TRANSFER_FAIL_FMT         L"Transfer of %ls failed."

    // =======================================================
    // 1200-1299: ERROR & WARNING MESSAGES
    // =======================================================
    IDS_ERR_CAPTION                         L"Error"
    IDS_ERR_MAX_SERVERS_REACHED             L"Maximum number of servers reached."
    IDS_ERR_REGISTER_FAIL_MAIN              L"Failed to register main window class! Application will now exit."
    IDS_ERR_CREATE_FAIL_MAIN                L"Failed to create main window! Application will now exit."
    IDS_ERR_REGISTER_FAIL_ABOUT             L"Failed to register About window class."
    IDS_ERR_REGISTER_FAIL_NET               L"Failed to register Network Settings window class."
    IDS_ERR_REGISTER_FAIL_TOOLS             L"Failed to register Tools window class."
    IDS_ERR_REGISTER_FAIL_FILE_EXPLORER     L"Failed to register File Explorer window class."
    IDS_TOOLS_MSG_ICON_SUCCESS              L"Icon has been changed successfully!"
    IDS_TOOLS_MSG_ICON_FAIL                 L"Failed to change the icon. Check file paths and permissions."
    IDS_TOOLS_MSG_EMPTY_PATHS               L"Please provide valid paths for both the icon and the executable file."
    IDS_TOOLS_MSG_EXTRACT_SUCCESS           L"Icon extracted successfully and saved to the specified path."
    IDS_TOOLS_MSG_EXTRACT_FAIL              L"Failed to extract icon. The file may not contain an icon or could not be read."

    // =======================================================
    // 1300-1399: REGISTRY & SETTINGS
    // =======================================================
    IDS_REG_KEY_PATH                        L"Software\\BulletEye\\ServerGUI"
    IDS_REG_SUBKEY_SERVERS                  L"Servers"
    IDS_REG_VALUE_IP                        1302
    IDS_REG_VALUE_PORT                      1303
    IDS_LOG_SERVERS_SAVED                   L"Server configurations saved to registry."
    IDS_LOG_SERVERS_LOADED                  L"Loaded %d server configuration(s) from registry."
    IDS_LOG_NO_SERVERS_FOUND                L"No saved server configurations found in registry."

    IDS_SETTINGS_CLASSNAME                  L"SettingsDialogClass"
    IDS_SETTINGS_SHOW_HIDDEN                L"Show hidden files and folders"
    IDS_LV_COL_NAME                         L"Name"
    IDS_LV_COL_TYPE                         L"Type"
    IDS_LV_COL_SIZE                         L"Size"
    IDS_LV_COL_MODIFIED                     L"Date modified"
    IDS_FE_ABOUT_CAPTION                    L"About Remote File Explorer"
    IDS_FE_ABOUT_TEXT                       L"Remote File Explorer\nVersion 1.0"
    IDS_CONFIRM_EXEC_CAPTION                L"Confirm Execution"
    IDS_CONFIRM_EXEC_TEXT                   L"Are you sure you want to execute this file?\nExecuting untrusted files may be dangerous."
    IDS_CONFIRM_DELETE_CAPTION              L"Confirm Deletion"
    IDS_CONFIRM_DELETE_TEXT                 L"Are you sure you want to delete this item?\nThis action cannot be undone."
END



1 RT_MANIFEST "Server.exe.manifest"
```

**File: `include\gui\file_explorer\fe_networking.h`**

```c
#ifndef FE_NETWORKING_H
#define FE_NETWORKING_H

#include <gui/file_explorer/file_explorer.h>
#include <resource.h>
#include <service/network_service.h> // For FileProgressCallback

void RequestDrives(FileExplorerData* pData);
void RequestDirectoryListing(FileExplorerData* pData, LPCWSTR pszPath);
void RequestTreeChildren(FileExplorerData* pData, HTREEITEM hParentItem);
void RequestFileExecution(FileExplorerData* pData, LPCWSTR pszPath);
void RequestFileDeletion(FileExplorerData* pData, LPCWSTR pszPath);
void RequestFileRename(FileExplorerData* pData, LPCWSTR pszOldPath, LPCWSTR pszNewPath);
void RequestFileUpload(FileExplorerData* pData, LPCWSTR pszLocalPath, LPCWSTR pszRemoteDestDir); // Modified signature
void RequestFileDownload(FileExplorerData* pData, LPCWSTR pszRemotePath, LPCWSTR pszLocalDestPath); // New function

#endif // FE_NETWORKING_H
```

**File: `include\service\network_service.h`**

```c
// include/service/network_service.h (No changes, but ensure it's correct)
#ifndef NETWORK_SERVICE_H
#define NETWORK_SERVICE_H

#include <globals.h>
#include <stdint.h>
#include <cJSON/cJSON.h>

// --- Network Protocol Definition ---
typedef enum {
    PACKET_TYPE_JSON = 1,
    PACKET_TYPE_BINARY_FILE = 2
} PacketType;

#pragma pack(push, 1)
typedef struct {
    uint32_t packet_type;
    uint32_t json_size;
    uint32_t binary_size;
} PacketHeader;
#pragma pack(pop)

typedef void (*FileProgressCallback)(long long bytes_transferred, long long total_bytes, void* userData); // Modified signature

// --- Public API Functions (UPDATED SIGNATURES) ---

BOOL SendJsonRequest(SOCKET s, const cJSON* json);

/**
 * @brief Sends a request containing a JSON object and a binary file with progress reporting.
 *
 * @param s The connected socket to send to.
 * @param json The cJSON object to send as metadata.
 * @param file_path The full path to the binary file to send.
 * @param callback An optional function pointer to be called with progress updates. Can be NULL.
 * @param callback_data User-defined data to pass to the callback function.
 * @return TRUE on success, FALSE on failure.
 */
BOOL SendFileRequest(SOCKET s, const cJSON* json, const WCHAR* file_path, FileProgressCallback callback, void* callback_data);

/**
 * @brief Receives and parses an incoming request with progress reporting for binary data.
 *
 * @param s The connected socket to receive from.
 * @param packet_type_out Pointer to store the received PacketType.
 * @param json_out Pointer to a cJSON* that will be allocated and populated.
 * @param binary_data_out Pointer to a char* buffer that will be allocated for the binary data.
 * @param binary_size_out Pointer to store the size of the received binary data.
 * @param callback An optional function pointer to be called with progress updates. Can be NULL.
 * @param callback_data User-defined data to pass to the callback function.
 * @return TRUE on success, FALSE on failure.
 */
BOOL ReceiveRequest(SOCKET s, PacketType* packet_type_out, cJSON** json_out, char** binary_data_out, uint32_t* binary_size_out, FileProgressCallback callback, void* callback_data);

#endif // NETWORK_SERVICE_H
```

**File: `src\service\network_service.c`**

```c
// src/service/network_service.c

#include <service/network_service.h>
#include <service/compression_service.h>
#include <service/encryption_service.h>
#include <stdio.h>
#include <stdlib.h>
#include <ws2tcpip.h>

#ifdef __TINYC__
    #include <windows.h>
    __declspec(dllimport) intptr_t _get_osfhandle(int fd);
    static long long ftelli64_tcc(FILE *f) {
        HANDLE h = (HANDLE)_get_osfhandle(_fileno(f));
        LARGE_INTEGER pos;
        pos.QuadPart = 0;
        SetFilePointerEx(h, pos, &pos, FILE_CURRENT);
        return pos.QuadPart;
    }

    #define _ftelli64(f) ftelli64_tcc(f)
#endif

// --- Internal Helper Functions ---

/**
 * @brief Reliably sends a buffer of a specific length over a socket.
 *
 * This function loops until all bytes are sent or an error occurs,
 * handling cases where send() does not send all data in one call.
 *
 * @param s The socket to send data on.
 * @param buf The buffer containing the data to send.
 * @param len The number of bytes to send.
 * @return TRUE if all bytes were sent successfully, FALSE otherwise.
 */
static BOOL send_all(SOCKET s, const char* buf, int len) {
    int total_sent = 0;
    while (total_sent < len) {
        int sent = send(s, buf + total_sent, len - total_sent, 0);
        if (sent == SOCKET_ERROR) {
            // A socket error occurred.
            return FALSE;
        }
        total_sent += sent;
    }
    return TRUE;
}

/**
 * @brief Reliably receives a specific number of bytes from a socket.
 *
 * This function loops until the exact number of bytes is received or
 * an error/disconnection occurs.
 *
 * @param s The socket to receive data from.
 * @param buf The buffer to store the received data.
 * @param len The exact number of bytes to receive.
 * @return TRUE if all bytes were received successfully, FALSE otherwise.
 */
static BOOL recv_all(SOCKET s, char* buf, int len) {
    int total_received = 0;
    while (total_received < len) {
        int received = recv(s, buf + total_received, len - total_received, 0);
        if (received == SOCKET_ERROR) {
            // A socket error occurred.
            return FALSE;
        }
        if (received == 0) {
            // The connection was gracefully closed by the peer.
            return FALSE;
        }
        total_received += received;
    }
    return TRUE;
}


// --- Public API Implementation ---

BOOL SendJsonRequest(SOCKET s, const cJSON* json) {
    char* json_string = cJSON_PrintUnformatted(json);
    if (!json_string) return FALSE;

    BYTE *compressed_data = NULL, *encrypted_data = NULL;
    DWORD compressed_size = 0, encrypted_size = 0;
    BOOL success = FALSE;

    // 1. Compress the JSON string
    if (!CompressData((BYTE*)json_string, (DWORD)strlen(json_string), &compressed_data, &compressed_size)) {
        goto cleanup;
    }

    // 2. Encrypt the compressed data
    if (!AESEncrypt(compressed_data, compressed_size, &encrypted_data, &encrypted_size)) {
        goto cleanup;
    }

    // 3. Prepare and Send Header
    PacketHeader header;
    header.packet_type = htonl((uint32_t)PACKET_TYPE_JSON);
    header.json_size = htonl(encrypted_size); // The size on the wire is the final encrypted size
    header.binary_size = 0;

    if (!send_all(s, (char*)&header, sizeof(header))) {
        goto cleanup;
    }

    // 4. Send the encrypted payload
    if (!send_all(s, (char*)encrypted_data, encrypted_size)) {
        goto cleanup;
    }

    success = TRUE;

cleanup:
    cJSON_free(json_string);
    if (compressed_data) free(compressed_data);
    if (encrypted_data) free(encrypted_data);
    return success;
}


BOOL SendFileRequest(SOCKET s, const cJSON* json, const WCHAR* file_path, FileProgressCallback callback, void* callback_data) {
    BOOL success = FALSE;

    // --- Part 1: Process JSON Metadata ---
    char* json_string = cJSON_PrintUnformatted(json);
    if (!json_string) return FALSE;

    BYTE *compressed_json = NULL, *encrypted_json = NULL;
    DWORD compressed_json_size = 0, encrypted_json_size = 0;

    if (!CompressData((BYTE*)json_string, (DWORD)strlen(json_string), &compressed_json, &compressed_json_size) ||
        !AESEncrypt(compressed_json, compressed_json_size, &encrypted_json, &encrypted_json_size)) {
        cJSON_free(json_string);
        if (compressed_json) free(compressed_json);
        return FALSE;
    }
    cJSON_free(json_string);
    free(compressed_json); // No longer needed

    // --- Part 2: Process Binary File ---
    FILE* file = NULL;
    long long file_size = 0;
    BYTE *file_buffer = NULL;
    BYTE *compressed_file = NULL, *encrypted_file = NULL;
    DWORD compressed_file_size = 0, encrypted_file_size = 0;

    // Check if file_path is provided and valid
    if (file_path && wcslen(file_path) > 0) {
        if (_wfopen_s(&file, file_path, L"rb") != 0 || !file) {
            goto cleanup;
        }

        _fseeki64(file, 0, SEEK_END);
        file_size = _ftelli64(file);
        rewind(file);

        if (file_size > (1024LL * 1024 * 500)) { // Safety limit of 500 MB
            // Log an error if needed
            goto cleanup;
        }

        file_buffer = (BYTE*)malloc((size_t)file_size);
        if (!file_buffer) {
            goto cleanup;
        }

        if (file_size > 0 && fread(file_buffer, 1, (size_t)file_size, file) != file_size) {
            goto cleanup; // fread failed
        }
        fclose(file); // Done with the file handle
        file = NULL; // Mark as closed

        if (file_size > 0) { // Only compress/encrypt if there's actual data
            if (!CompressData(file_buffer, (DWORD)file_size, &compressed_file, &compressed_file_size) ||
                !AESEncrypt(compressed_file, compressed_file_size, &encrypted_file, &encrypted_file_size)) {
                goto cleanup; // processing failed
            }
        }
    }
    // If no file_path, or file_size is 0, encrypted_file and encrypted_file_size will remain 0.

    // --- Part 3: Send Everything ---
    PacketHeader header;
    header.packet_type = htonl((uint32_t)PACKET_TYPE_BINARY_FILE);
    header.json_size = htonl(encrypted_json_size);
    header.binary_size = htonl(encrypted_file_size);

    // Send header and JSON first
    if (!send_all(s, (char*)&header, sizeof(header)) ||
        !send_all(s, (char*)encrypted_json, encrypted_json_size)) {
        goto cleanup;
    }

    // Send binary payload in chunks and report progress
    const int CHUNK_SIZE = 4096;
    long long total_sent = 0;
    while (total_sent < encrypted_file_size) {
        int bytes_to_send = min(CHUNK_SIZE, (int)(encrypted_file_size - total_sent));
        if (!send_all(s, (char*)encrypted_file + total_sent, bytes_to_send)) {
            goto cleanup; // Send failed
        }
        total_sent += bytes_to_send;
        if (callback) {
            callback(total_sent, encrypted_file_size, callback_data); // Pass callback_data here
        }
    }
    success = TRUE;

cleanup:
    if (file) fclose(file);
    if (file_buffer) free(file_buffer);
    if (compressed_file) free(compressed_file);
    if (encrypted_file) free(encrypted_file);
    if (encrypted_json) free(encrypted_json);
    return success;
}

BOOL ReceiveRequest(SOCKET s, PacketType* packet_type_out, cJSON** json_out, char** binary_data_out, uint32_t* binary_size_out, FileProgressCallback callback, void* callback_data) {
    // Initialize output parameters to safe values
    *json_out = NULL;
    *binary_data_out = NULL;
    *binary_size_out = 0;
    *packet_type_out = 0;
    BOOL success = FALSE;

    BYTE *encrypted_buffer = NULL, *decrypted_buffer = NULL, *decompressed_buffer = NULL;

    // 1. Receive Header
    PacketHeader header;
    if (!recv_all(s, (char*)&header, sizeof(header))) return FALSE;

    // Convert from network to host byte order
    header.packet_type = ntohl(header.packet_type);
    header.json_size = ntohl(header.json_size);
    header.binary_size = ntohl(header.binary_size);
    *packet_type_out = (PacketType)header.packet_type;

    // 2. Process JSON Payload (if it exists)
    if (header.json_size > 0) {
        DWORD decrypted_size = 0, decompressed_size = 0;
        encrypted_buffer = (BYTE*)malloc(header.json_size);
        if (!encrypted_buffer || !recv_all(s, (char*)encrypted_buffer, header.json_size)) {
             goto cleanup;
        }

        if (!AESDecrypt(encrypted_buffer, header.json_size, &decrypted_buffer, &decrypted_size) ||
            !DecompressData(decrypted_buffer, decrypted_size, &decompressed_buffer, &decompressed_size)) {
            goto cleanup;
        }

        // Add null terminator for cJSON
        char* final_json_string = (char*)realloc(decompressed_buffer, decompressed_size + 1);
        if (!final_json_string) { decompressed_buffer = NULL; goto cleanup; } // handle realloc failure
        final_json_string[decompressed_size] = '\0';
        decompressed_buffer = NULL; // a new pointer now manages the memory

        *json_out = cJSON_Parse(final_json_string);
        free(final_json_string); // cJSON has parsed it, we can free the string
        if (*json_out == NULL) goto cleanup;

        free(encrypted_buffer); encrypted_buffer = NULL;
        free(decrypted_buffer); decrypted_buffer = NULL;
    }

    // 3. Process Binary Payload (if it exists)
    if (header.binary_size > 0) {
        encrypted_buffer = (BYTE*)malloc(header.binary_size);
        if (!encrypted_buffer) goto cleanup;

        // Receive binary payload in chunks and report progress
        const int CHUNK_SIZE = 4096;
        long long total_received = 0;
        while (total_received < header.binary_size) {
            int bytes_to_receive = min(CHUNK_SIZE, (int)(header.binary_size - total_received));
            int received = recv(s, (char*)encrypted_buffer + total_received, bytes_to_receive, 0);
            if (received == SOCKET_ERROR || received == 0) {
                goto cleanup; // Receive failed or connection closed
            }
            total_received += received;
            if (callback) {
                callback(total_received, header.binary_size, callback_data); // Pass callback_data here
            }
        }

        DWORD decrypted_size = 0, decompressed_size = 0;
        if (!AESDecrypt(encrypted_buffer, header.binary_size, &decrypted_buffer, &decrypted_size) ||
            !DecompressData(decrypted_buffer, decrypted_size, &decompressed_buffer, &decompressed_size)) {
            goto cleanup;
        }

        // The final decompressed buffer is the result. Transfer ownership to the caller.
        *binary_data_out = (char*)decompressed_buffer;
        *binary_size_out = decompressed_size;
        decompressed_buffer = NULL; // Ownership transferred, prevent double free

        free(encrypted_buffer); encrypted_buffer = NULL;
        free(decrypted_buffer); decrypted_buffer = NULL;
    }

    success = TRUE;

cleanup:
    // Free any intermediate buffers that might still be allocated on failure
    if (encrypted_buffer) free(encrypted_buffer);
    if (decrypted_buffer) free(decrypted_buffer);
    if (decompressed_buffer) free(decompressed_buffer);

    // If the whole process failed, ensure the caller receives nothing
    if (!success) {
        if (*json_out) { cJSON_Delete(*json_out); *json_out = NULL; }
        if (*binary_data_out) { free(*binary_data_out); *binary_data_out = NULL; }
    }
    return success;
}
```

**File: `src\gui\file_explorer\file_explorer.c`**

```c
// src/gui/file_explorer/file_explorer.c
#define _WIN32_IE 0x0600 // Required for some shell functions like SHGetFileInfo

#include <gui/file_explorer/file_explorer.h>
#include <gui/file_explorer/fe_ui.h>
#include <gui/file_explorer/fe_events.h>
#include <gui/file_explorer/fe_networking.h>
#include <gui/file_explorer/fe_cache.h> // Include cache header
#include <DarkMode.h>
#include <resource.h>
#include <stdio.h> // For swprintf_s
#include <stdlib.h> // For calloc, free
#include <Shlwapi.h> // For PathFindFileNameW

// Forward declaration for the window procedure
static LRESULT CALLBACK FileExplorerProc(HWND, UINT, WPARAM, LPARAM);

// Helper to recursively free the lParam associated with TreeView items
// This only frees the allocated memory pointed to by lParam, it does not delete the TreeView item itself.
static void FreeTreeViewItemData(HWND hTree, HTREEITEM hItem) {
    if (hItem == NULL) return;

    TVITEMW tvi = {0};
    tvi.mask = TVIF_PARAM;
    tvi.hItem = hItem;

    // Retrieve and free the lParam for the current item
    if (TreeView_GetItem(hTree, &tvi) && tvi.lParam) {
        free((void*)tvi.lParam);
    }

    // Recursively call for children
    HTREEITEM hChild = TreeView_GetChild(hTree, hItem);
    while (hChild != NULL) {
        HTREEITEM hNextSibling = TreeView_GetNextSibling(hTree, hChild); // Get next sibling before current child is processed
        FreeTreeViewItemData(hTree, hChild); // Recursive call
        hChild = hNextSibling;
    }
}


void CreateFileExplorerWindow(HWND hParent, int clientIndex) {
    // Check if a File Explorer window for this client already exists
    EnterCriticalSection(&g_cs_clients);
    if (clientIndex >= 0 && clientIndex < g_active_client_count && g_active_clients[clientIndex].hExplorerWnd != NULL && IsWindow(g_active_clients[clientIndex].hExplorerWnd)) {
        SetForegroundWindow(g_active_clients[clientIndex].hExplorerWnd); // Bring existing window to front
        LeaveCriticalSection(&g_cs_clients);
        return;
    }
    LeaveCriticalSection(&g_cs_clients);

    // Get window class name from resources
    WCHAR className[128];
    LoadStringW(g_hinst, IDS_FILE_EXPLORER_CLASSNAME, className, _countof(className));

    // Register the window class if it hasn't been registered yet
    static BOOL isClassRegistered = FALSE;
    if (!isClassRegistered) {
        WNDCLASSEXW wc = {0};
        wc.cbSize = sizeof(WNDCLASSEXW);
        wc.lpfnWndProc = FileExplorerProc;
        wc.hInstance = g_hinst;
        wc.hCursor = LoadCursor(NULL, IDC_ARROW);
        wc.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1); // Default background (might be overridden by DarkMode)
        wc.lpszClassName = className;
        wc.hIcon = LoadIcon(g_hinst, MAKEINTRESOURCE(IDI_ICON_MAIN));
        if (!RegisterClassExW(&wc)) {
            WCHAR msg[256], cap[64];
            LoadStringW(g_hinst, IDS_ERR_REGISTER_FAIL_FILE_EXPLORER, msg, _countof(msg));
            LoadStringW(g_hinst, IDS_ERR_CAPTION, cap, _countof(cap));
            MessageBoxW(hParent, msg, cap, MB_ICONERROR);
            return;
        }
        isClassRegistered = TRUE;
    }

    // Allocate and initialize instance data for this File Explorer window
    FileExplorerData* pData = (FileExplorerData*)calloc(1, sizeof(FileExplorerData));
    if (!pData) return; // Memory allocation failed

    pData->clientIndex = clientIndex;
    pData->iBackStackTop = -1;    // Initialize history stacks as empty
    pData->iForwardStackTop = -1;
    InitializeCriticalSection(&pData->csCache); // Initialize cache critical section

    // Construct the window title
    WCHAR windowTitle[256], titleFormat[128];
    LoadStringW(g_hinst, IDS_FILE_EXPLORER_CAPTION, titleFormat, _countof(titleFormat));

    EnterCriticalSection(&g_cs_clients);
    if (clientIndex >= 0 && clientIndex < g_active_client_count) {
        swprintf_s(windowTitle, _countof(windowTitle), L"%s - %hs", titleFormat, g_active_clients[clientIndex].info.ipAddress);
    } else {
        // This case should ideally not happen if called from a valid context menu, but provides a fallback.
        #ifndef __TINYC__
            wcscpy_s(windowTitle, _countof(windowTitle), titleFormat);
        #else
            snwprintf(windowTitle, sizeof(windowTitle) / sizeof(wchar_t), L"%ls", titleFormat);
        #endif

    }
    LeaveCriticalSection(&g_cs_clients);

    // Create the File Explorer main window
    HWND hExplorer = CreateWindowExW(WS_EX_ACCEPTFILES, className, windowTitle,
                                WS_OVERLAPPEDWINDOW | WS_CLIPCHILDREN,
                                CW_USEDEFAULT, CW_USEDEFAULT, 900, 700,
                                hParent, NULL, g_hinst, (LPVOID)pData); // Pass pData as creation parameter

    if (hExplorer) {
        // Store the new window handle in the client struct for tracking
        EnterCriticalSection(&g_cs_clients);
        if (clientIndex >= 0 && clientIndex < g_active_client_count) {
            g_active_clients[clientIndex].hExplorerWnd = hExplorer;
        }
        LeaveCriticalSection(&g_cs_clients);

        EnableWindow(hParent, FALSE); // Disable parent window
        ShowWindow(hExplorer, SW_SHOW);
    } else {
        // Window creation failed, clean up allocated resources
        DeleteCriticalSection(&pData->csCache);
        free(pData);
        EnableWindow(hParent, TRUE); // Re-enable parent if creation fails
    }
}

static LRESULT CALLBACK FileExplorerProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam) {
    FileExplorerData* pData = (FileExplorerData*)GetWindowLongPtr(hWnd, GWLP_USERDATA);

    switch (message) {
        case WM_CREATE: {
            CREATESTRUCT* pCreate = (CREATESTRUCT*)lParam;
            pData = (FileExplorerData*)pCreate->lpCreateParams; // Retrieve pData from creation parameters
            SetWindowLongPtr(hWnd, GWLP_USERDATA, (LONG_PTR)pData); // Store pData for later use
            pData->hMain = hWnd; // Set the main window handle in pData

            if (!CreateExplorerControls(hWnd, pData)) {
                DestroyWindow(hWnd); // If control creation fails, destroy the window
                return -1;
            }
            if (DarkMode_isEnabled()) {
                DarkMode_setDarkWndNotifySafe_Default(hWnd);
                DarkMode_setWindowEraseBgSubclass(hWnd);
            }

            RequestDrives(pData); // Request initial drive list
            UpdateNavButtonsState(pData); // Initialize navigation button states
            break;
        }
        case WM_SIZE:
            if (pData)
                ResizeExplorerControls(hWnd); // Adjust control sizes on window resize
            break;
        case WM_DROPFILES:
            if (pData)
                OnFileDrop(pData, wParam); // Handle files dropped onto the window
            break;
        case WM_NOTIFY: {
            if (!pData)
                break;
            LPNMHDR lpnmh = (LPNMHDR)lParam;
            if (lpnmh->hwndFrom == pData->hTreeView) {
                if (lpnmh->code == TVN_SELCHANGEDW) OnTreeViewSelChanged(pData, lParam);
                else if (lpnmh->code == TVN_ITEMEXPANDINGW) OnTreeViewItemExpanding(pData, lParam);
            } else if (lpnmh->hwndFrom == pData->hListView) {
                if (lpnmh->code == NM_DBLCLK) OnListViewDblClick(pData, lParam);
                else if (lpnmh->code == LVN_ENDLABELEDITW) OnListViewEndLabelEdit(pData, lParam);
                // Handle context menu for list view items
                else if (lpnmh->code == NM_RCLICK) { // Right-click notification
                    POINT pt;
                    GetCursorPos(&pt);
                    ScreenToClient(pData->hListView, &pt);
                    ShowExplorerContextMenu(pData, pt);
                }
            }
            break;
        }
        case WM_CONTEXTMENU:
            // Handle context menu for the tree view or other parts of the window
            // The NM_RCLICK for listview already handles the menu, this is a fallback for background or other controls.
            if (pData && (HWND)wParam == pData->hListView) {
                // Already handled by NM_RCLICK
            }
            break;
        case WM_COMMAND:
            if (pData)
                OnContextMenuCommand(pData, LOWORD(wParam)); // Handle toolbar and menu commands
            break;
        case WM_ERASEBKGND: {
            HDC hdc = (HDC)wParam;
            RECT rc;
            GetClientRect(hWnd, &rc);

            if (DarkMode_isEnabled()) {
                FillRect(hdc, &rc, DarkMode_getBackgroundBrush());
                return 1; // Indicate that background is erased
            } else {
                // Paint with the default system brush when dark mode is OFF
                FillRect(hdc, &rc, GetSysColorBrush(COLOR_WINDOW));
                return 1; // Indicate that background is erased
            }
        }
        case WM_CLOSE: {
            // Re-enable the main application window and destroy this window
            EnableWindow(g_hWnd, TRUE);
            SetForegroundWindow(g_hWnd);
            DestroyWindow(hWnd); // Proceed to WM_DESTROY
            break;
        }
        case WM_DESTROY: {
            // Restore main window's state and clean up resources
            EnableWindow(g_hWnd, TRUE);
            SetForegroundWindow(g_hWnd);

            if (pData) {
                pData->bShutdown = TRUE; // Signal shutdown to worker threads early
                // A small sleep to give worker threads a chance to react to `bShutdown`.
                // For critical applications, consider using explicit thread joining with `WaitForSingleObject`.
                Sleep(100);

                // Clear explorer window reference for this specific client
                EnterCriticalSection(&g_cs_clients);
                if (pData->clientIndex >= 0 && pData->clientIndex < g_active_client_count && g_active_clients[pData->clientIndex].hExplorerWnd == hWnd) {
                    g_active_clients[pData->clientIndex].hExplorerWnd = NULL;
                }
                LeaveCriticalSection(&g_cs_clients);

                // Restore original window procedure for the address bar to prevent subclassing issues
                if (pData->hAddressBar && pData->pfnOldAddressProc) {
                    SetWindowLongPtr(pData->hAddressBar, GWLP_WNDPROC, (LONG_PTR)pData->pfnOldAddressProc);
                    pData->pfnOldAddressProc = NULL; // Clear stored WNDPROC to prevent double un-subclass
                }

                // Destroy toolbar image list (stored in GWLP_USERDATA of the toolbar)
                HIMAGELIST hImgList = (HIMAGELIST)GetWindowLongPtr(pData->hToolBar, GWLP_USERDATA);
                if (hImgList) {
                    ImageList_Destroy(hImgList);
                    SetWindowLongPtr(pData->hToolBar, GWLP_USERDATA, (LONG_PTR)NULL); // Clear the stored handle
                }

                // Cleanup ListView items' allocated data (LV_ITEM_DATA)
                int count = ListView_GetItemCount(pData->hListView);
                for (int i = 0; i < count; i++) {
                    LVITEMW lvi = {0}; lvi.mask = LVIF_PARAM; lvi.iItem = i;
                    if (ListView_GetItem(pData->hListView, &lvi) && lvi.lParam) {
                        free((LV_ITEM_DATA*)lvi.lParam);
                    }
                }
                ListView_DeleteAllItems(pData->hListView); // Delete all items from the control

                // Cleanup TreeView items' allocated data (paths stored as lParam)
                HTREEITEM hRoot = TreeView_GetRoot(pData->hTreeView);
                while (hRoot) {
                    HTREEITEM hNextRoot = TreeView_GetNextSibling(pData->hTreeView, hRoot); // Get next sibling before current hRoot is processed
                    FreeTreeViewItemData(pData->hTreeView, hRoot); // Recursively frees lParam for hRoot and its children
                    hRoot = hNextRoot;
                }
                TreeView_DeleteAllItems(pData->hTreeView); // Delete all items from the control

                // Clear cache entries and free their associated JSON objects
                ClearCache(pData);
                DeleteCriticalSection(&pData->csCache); // Delete the critical section

                // Finally, free the main FileExplorerData structure
                free(pData);
                // Clear the GWLP_USERDATA associated with the window to prevent dangling pointer
                SetWindowLongPtr(hWnd, GWLP_USERDATA, (LONG_PTR)NULL);
            }
            break;
        }
        case WM_APP_FE_CLIENT_DISCONNECTED: {
            // Handle notification that the client has disconnected
            MessageBoxW(hWnd, L"The client has disconnected. This file explorer window will now close.", L"Connection Lost", MB_OK | MB_ICONINFORMATION);
            PostMessage(hWnd, WM_CLOSE, 0, 0); // Trigger WM_CLOSE to properly shut down the window
            return 0;
        }
        case WM_APP_FE_UPDATE_DRIVES:
            if (pData) UpdateDrivesInTreeView(pData, (cJSON*)lParam);
            else if (lParam) cJSON_Delete((cJSON*)lParam); // If pData is null, free the received JSON
            SetExplorerControlsEnabled(pData, TRUE); // Re-enable controls if they were disabled for request
            break;
        case WM_APP_FE_UPDATE_LISTVIEW:
            if (pData) UpdateListView(pData, (cJSON*)lParam);
            else if (lParam) cJSON_Delete((cJSON*)lParam); // If pData is null, free the received JSON
            SetExplorerControlsEnabled(pData, TRUE); // Re-enable controls if they were disabled for request
            break;
        case WM_APP_FE_UPDATE_TREE_CHILDREN: {
            if (pData && lParam) {
                cJSON* data = (cJSON*)lParam;
                HTREEITEM hParent = (HTREEITEM)(ULONG_PTR)cJSON_GetObjectItem(data, "hParent")->valuedouble;
                cJSON* items = cJSON_GetObjectItem(data, "items");
                UpdateTreeChildren(pData, hParent, items); // Note: items is detached, so don't delete it separately here
                cJSON_Delete(data); // Delete the wrapper JSON object
            } else if (lParam) {
                 cJSON_Delete((cJSON*)lParam); // If pData is null, free the received JSON
            }
            SetExplorerControlsEnabled(pData, TRUE); // Re-enable controls if they were disabled for request
            break;
        }
        case WM_APP_FE_REFRESH_VIEW: // This is often used after operations like delete/rename/upload
            if (pData) RequestDirectoryListing(pData, pData->szCurrentPath); // Re-request the current directory
            // This message implicitly re-enables controls via RequestDirectoryListing callback
            break;
        case WM_APP_FE_TRANSFER_PROGRESS: { // New message handler for progress
            FileTransferProgressInfo* progress = (FileTransferProgressInfo*)lParam;
            if (pData && progress) {
                WCHAR statusText[512], fmt[128];
                double percentage = (double)progress->current * 100.0 / progress->total;
                UINT fmt_id = progress->isUpload ? IDS_FE_STATUS_UPLOADING_FMT : IDS_FE_STATUS_DOWNLOADING_FMT;
                LoadStringW(g_hinst, fmt_id, fmt, _countof(fmt));
                swprintf_s(statusText, ARRAYSIZE(statusText), fmt,
                           progress->fileName, progress->current, progress->total, percentage);
                SendMessageW(pData->hStatusBar, SB_SETTEXTW, 0, (LPARAM)statusText);
            }
            if (progress) free(progress); // Free the progress info struct
            return 0;
        }
        case WM_APP_FE_TRANSFER_COMPLETE: { // New message handler for completion
            FileTransferProgressInfo* progress = (FileTransferProgressInfo*)lParam;
            if (pData && progress) {
                WCHAR statusText[256], fmt[128];
                UINT fmt_id;
                if (progress->total > 0) { // Success (convention: total > 0 means successful transfer)
                    fmt_id = progress->isUpload ? IDS_FE_STATUS_UPLOAD_COMPLETE_FMT : IDS_FE_STATUS_DOWNLOAD_COMPLETE_FMT;
                    LoadStringW(g_hinst, fmt_id, fmt, _countof(fmt));
                    swprintf_s(statusText, ARRAYSIZE(statusText), fmt, progress->fileName);
                } else { // Failure (convention: total=0 indicates failure)
                    fmt_id = IDS_FE_STATUS_TRANSFER_FAIL_FMT;
                    LoadStringW(g_hinst, fmt_id, fmt, _countof(fmt));
                    swprintf_s(statusText, ARRAYSIZE(statusText), fmt, progress->fileName);
                }
                SendMessageW(pData->hStatusBar, SB_SETTEXTW, 0, (LPARAM)statusText);
                if (progress->isUpload) {
                    // Refresh current view after upload to show the new file
                    RequestDirectoryListing(pData, pData->szCurrentPath);
                }
            }
            SetExplorerControlsEnabled(pData, TRUE); // Re-enable controls
            if (progress) free(progress); // Free the progress info struct
            return 0;
        }
        default: return DefWindowProc(hWnd, message, wParam, lParam);
    }
    return 0;
}

// Helper function to get the lParam (path string) from a TreeView item
LPARAM TreeView_GetItemParam(HWND hTree, HTREEITEM hItem) {
    TVITEMW tvi = {0};
    tvi.mask = TVIF_PARAM;
    tvi.hItem = hItem;
    if (TreeView_GetItem(hTree, &tvi)) {
        return tvi.lParam;
    }
    return 0;
}
```

**File: `src\gui\file_explorer\fe_events.c`**

```c
// src/gui/file_explorer/fe_events.c
#include <gui/file_explorer/fe_events.h>
#include <gui/file_explorer/fe_navigation.h>
#include <gui/file_explorer/fe_networking.h>
#include <gui/file_explorer/fe_ui.h>
#include <resource.h>
#include <shlwapi.h>
#include <shlobj.h> // For SHGetSpecialFolderPath

// Handles selection change in the TreeView
void OnTreeViewSelChanged(FileExplorerData* pData, LPARAM lParam) {
    LPNMTREEVIEWW lpnmtd = (LPNMTREEVIEWW)lParam;
    LPCWSTR pszPath = (LPCWSTR)TreeView_GetItemParam(pData->hTreeView, lpnmtd->itemNew.hItem);
    if (pszPath) {
        NavigateTo(pData, pszPath, TRUE);
    }
}

// Handles item expanding in the TreeView
void OnTreeViewItemExpanding(FileExplorerData* pData, LPARAM lParam) {
    LPNMTREEVIEWW lpnmtd = (LPNMTREEVIEWW)lParam;
    // Only expand if it hasn't been expanded before (TVIS_EXPANDEDONCE) to avoid redundant network requests
    if ((lpnmtd->action == TVE_EXPAND) && !(lpnmtd->itemNew.state & TVIS_EXPANDEDONCE)) {
        RequestTreeChildren(pData, lpnmtd->itemNew.hItem);

        // Mark the item as having been expanded
        TVITEMW item = {0};
        item.hItem = lpnmtd->itemNew.hItem;
        item.mask = TVIF_STATE;
        item.state = TVIS_EXPANDEDONCE;
        item.stateMask = TVIS_EXPANDEDONCE;
        TreeView_SetItem(pData->hTreeView, &item);
    }
}

// Handles double-click on an item in the ListView
void OnListViewDblClick(FileExplorerData* pData, LPARAM lParam) {
    LPNMITEMACTIVATE lpnmia = (LPNMITEMACTIVATE)lParam;
    if (lpnmia->iItem == -1) return; // No item was clicked

    LVITEMW lvi = {0};
    lvi.mask = LVIF_PARAM;
    lvi.iItem = lpnmia->iItem;
    ListView_GetItem(pData->hListView, &lvi);

    LV_ITEM_DATA* pItemData = (LV_ITEM_DATA*)lvi.lParam;
    if (!pItemData) return; // No custom data for this item

    if (pItemData->isDirectory) {
        NavigateTo(pData, pItemData->fullPath, TRUE);
    } else {
        // If it's a file, treat double click as an "Open" command
        OnContextMenuCommand(pData, IDM_FE_CONTEXT_OPEN);
    }
}

// Handles end of label editing in the ListView (for renaming)
void OnListViewEndLabelEdit(FileExplorerData* pData, LPARAM lParam) {
    NMLVDISPINFOW* pdi = (NMLVDISPINFOW*)lParam;
    if (pdi->item.pszText == NULL || pdi->item.pszText[0] == L'\0') return; // Empty new name

    // Retrieve original item data
    LVITEMW lvi = {0};
    lvi.mask = LVIF_PARAM;
    lvi.iItem = pdi->item.iItem;
    ListView_GetItem(pData->hListView, &lvi);

    LV_ITEM_DATA* pItemData = (LV_ITEM_DATA*)lvi.lParam;
    if (!pItemData) return;

    // Construct the new full path
    WCHAR szNewPath[MAX_PATH];
    PathCombineW(szNewPath, pData->szCurrentPath, pdi->item.pszText);

    // Request rename operation
    RequestFileRename(pData, pItemData->fullPath, szNewPath);
}

// Handles commands from menus/toolbar in the File Explorer window
void OnContextMenuCommand(FileExplorerData* pData, int id) {
    // Handle toolbar/menu commands that don't depend on a selected item
    switch(id) {
        case IDC_FE_NAV_BACK: OnNavBack(pData); return;
        case IDC_FE_NAV_FORWARD: OnNavForward(pData); return;
        case IDC_FE_SHOW_SETTINGS: ShowSettingsDialog(pData); return;
        case IDM_FE_VIEW_REFRESH: RequestDirectoryListing(pData, pData->szCurrentPath); return;
        case IDM_FE_FILE_EXIT: DestroyWindow(pData->hMain); return;
        case IDM_FE_FILE_UPLOAD: { // New: Handle File -> Upload...
            WCHAR szLocalPath[MAX_PATH] = {0};
            OPENFILENAMEW ofn = {0};
            ofn.lStructSize = sizeof(ofn);
            ofn.hwndOwner = pData->hMain;
            ofn.lpstrFile = szLocalPath;
            ofn.nMaxFile = MAX_PATH;
            ofn.lpstrFilter = L"All Files (*.*)\0*.*\0";
            ofn.Flags = OFN_FILEMUSTEXIST | OFN_PATHMUSTEXIST | OFN_EXPLORER;

            if (GetOpenFileNameW(&ofn)) {
                RequestFileUpload(pData, szLocalPath, pData->szCurrentPath); // Upload to current remote directory
            }
            return;
        }
        case IDM_FE_HELP_ABOUT: {
            WCHAR text[128], caption[128];
            LoadStringW(g_hinst, IDS_FE_ABOUT_TEXT, text, 128);
            LoadStringW(g_hinst, IDS_FE_ABOUT_CAPTION, caption, 128);
            MessageBoxW(pData->hMain, text, caption, MB_OK | MB_ICONINFORMATION);
            return;
        }
    }

    // For context menu commands that require a selected item
    int iItem = ListView_GetNextItem(pData->hListView, -1, LVNI_SELECTED);
    if (iItem == -1) return; // No item selected

    LVITEMW lvi = {0}; lvi.mask = LVIF_PARAM; lvi.iItem = iItem;
    ListView_GetItem(pData->hListView, &lvi);
    LV_ITEM_DATA* pItemData = (LV_ITEM_DATA*)lvi.lParam;
    if (!pItemData) return;

    WCHAR msg[256], cap[64];
    switch (id) {
        case IDM_FE_CONTEXT_OPEN:
            // Use the DblClick logic for opening files/folders
            OnListViewDblClick(pData, (LPARAM)&(NMITEMACTIVATE){.iItem = iItem});
            break;
        case IDM_FE_CONTEXT_DELETE:
            LoadStringW(g_hinst, IDS_CONFIRM_DELETE_TEXT, msg, _countof(msg));
            LoadStringW(g_hinst, IDS_CONFIRM_DELETE_CAPTION, cap, _countof(cap));
            if (MessageBoxW(pData->hMain, msg, cap, MB_YESNO | MB_ICONWARNING) == IDYES) {
                RequestFileDeletion(pData, pItemData->fullPath);
            }
            break;
        case IDM_FE_CONTEXT_RENAME:
            ListView_EditLabel(pData->hListView, iItem);
            break;
        case IDM_FE_CONTEXT_DOWNLOAD: { // New: Handle Download context menu item
            if (pItemData->isDirectory) break; // Cannot download folders

            WCHAR szLocalDestPath[MAX_PATH] = {0};
            OPENFILENAMEW ofn = {0};
            ofn.lStructSize = sizeof(ofn);
            ofn.hwndOwner = pData->hMain;
            ofn.lpstrFile = szLocalDestPath;
            ofn.nMaxFile = MAX_PATH;
            ofn.lpstrFilter = L"All Files (*.*)\0*.*\0";
            ofn.Flags = OFN_OVERWRITEPROMPT | OFN_PATHMUSTEXIST | OFN_EXPLORER;

            // Pre-fill filename with the remote file's name in the initial directory
            LPCWSTR remoteFileName = PathFindFileNameW(pItemData->fullPath);
            // Default to My Documents or Desktop for save location if possible
            WCHAR initialDir[MAX_PATH] = {0};
            if (SHGetSpecialFolderPathW(pData->hMain, initialDir, CSIDL_MYDOCUMENTS, FALSE)) {
                ofn.lpstrInitialDir = initialDir;
            }

            if (remoteFileName) {
                wcscpy_s(szLocalDestPath, MAX_PATH, remoteFileName);
            }

            if (GetSaveFileNameW(&ofn)) {
                RequestFileDownload(pData, pItemData->fullPath, szLocalDestPath);
            }
            break;
        }
    }
}

// Handles file drop operations (for uploading)
void OnFileDrop(FileExplorerData* pData, WPARAM wParam) {
    HDROP hDrop = (HDROP)wParam;
    WCHAR szFilePath[MAX_PATH];
    UINT nFiles = DragQueryFileW(hDrop, 0xFFFFFFFF, NULL, 0);

    for (UINT i = 0; i < nFiles; ++i) {
        if (DragQueryFileW(hDrop, i, szFilePath, MAX_PATH)) {
            // Upload each dropped file to the current remote directory
            RequestFileUpload(pData, szFilePath, pData->szCurrentPath);
        }
    }
    DragFinish(hDrop); // Release the memory for the dropped files
}

// Subclass procedure for the address bar to handle Enter key
LRESULT CALLBACK AddressBarProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam) {
    FileExplorerData* pData = (FileExplorerData*)GetWindowLongPtr(GetParent(hWnd), GWLP_USERDATA);
    if (uMsg == WM_KEYDOWN && wParam == VK_RETURN) {
        WCHAR szPath[MAX_PATH];
        GetWindowTextW(hWnd, szPath, MAX_PATH);
        NavigateTo(pData, szPath, TRUE);
        return 0; // Prevent further processing of VK_RETURN by the default edit control proc
    }
    // Call the original window procedure for other messages
    return CallWindowProc(pData->pfnOldAddressProc, hWnd, uMsg, wParam, lParam);
}
```

**File: `src\gui\file_explorer\fe_ui.c`**

```c
// src/gui/file_explorer/fe_ui.c
#include <gui/file_explorer/fe_ui.h>
#include <gui/file_explorer/fe_events.h>
#include <gui/file_explorer/fe_navigation.h>
#include <gui/file_explorer/fe_settings.h>
#include <gui/file_explorer/fe_cache.h>
#include <resource.h>
#include <shlwapi.h>
#include <shellapi.h>
#include <helpers/utils.h>
#include <stdio.h>
#include <wchar.h>
#include <globals.h>
#include <DarkMode.h>
#include <uxtheme.h>

void SetExplorerControlsEnabled(FileExplorerData* pData, BOOL bEnabled) {
    if (!pData || !IsWindow(pData->hMain)) return;
    EnableWindow(pData->hTreeView, bEnabled);
    EnableWindow(pData->hListView, bEnabled);
    EnableWindow(pData->hAddressBar, bEnabled);
    EnableWindow(pData->hToolBar, bEnabled);
    if (bEnabled) {
        UpdateNavButtonsState(pData); // Update button states, especially nav buttons
    } else {
        // Optionally grey out nav buttons too when all controls are disabled
        SendMessage(pData->hToolBar, TB_SETSTATE, IDC_FE_NAV_BACK, 0);
        SendMessage(pData->hToolBar, TB_SETSTATE, IDC_FE_NAV_FORWARD, 0);
    }
}

BOOL CreateExplorerControls(HWND hWnd, FileExplorerData* pData) {
    HMENU hMenu = CreateMenu(), hFileMenu = CreateMenu(), hViewMenu = CreateMenu(), hHelpMenu = CreateMenu();
    WCHAR buf[128];

    // File Menu items
    LoadStringW(g_hinst, IDS_FE_MENU_OPEN, buf, 128); AppendMenuW(hFileMenu,MF_STRING,IDM_FE_FILE_OPEN,buf);
    LoadStringW(g_hinst, IDS_FE_MENU_UPLOAD, buf, 128); AppendMenuW(hFileMenu,MF_STRING,IDM_FE_FILE_UPLOAD,buf); // New Upload item
    AppendMenuW(hFileMenu,MF_SEPARATOR,0,NULL);
    LoadStringW(g_hinst, IDS_FE_MENU_EXIT, buf, 128); AppendMenuW(hFileMenu,MF_STRING,IDM_FE_FILE_EXIT,buf);

    // View Menu items
    LoadStringW(g_hinst, IDS_FE_MENU_REFRESH, buf, 128); AppendMenuW(hViewMenu,MF_STRING,IDM_FE_VIEW_REFRESH,buf);

    // Help Menu items
    LoadStringW(g_hinst, IDS_FE_MENU_ABOUT, buf, 128); AppendMenuW(hHelpMenu,MF_STRING,IDM_FE_HELP_ABOUT,buf);

    // Main Menu
    LoadStringW(g_hinst, IDS_FE_MENU_FILE, buf, 128); AppendMenuW(hMenu,MF_POPUP,(UINT_PTR)hFileMenu,buf);
    LoadStringW(g_hinst, IDS_FE_MENU_VIEW, buf, 128); AppendMenuW(hMenu,MF_POPUP,(UINT_PTR)hViewMenu,buf);
    LoadStringW(g_hinst, IDS_FE_MENU_HELP, buf, 128); AppendMenuW(hMenu,MF_POPUP,(UINT_PTR)hHelpMenu,buf);
    SetMenu(hWnd, hMenu);

    pData->hToolBar = CreateWindowEx(0, TOOLBARCLASSNAMEW, NULL,
        WS_CHILD|WS_VISIBLE|TBSTYLE_FLAT|TBSTYLE_TOOLTIPS|CCS_NODIVIDER,
        0,0,0,0, hWnd, (HMENU)IDC_FE_TOOLBAR, g_hinst, NULL);
    SendMessage(pData->hToolBar, TB_SETBUTTONSIZE, 0, MAKELONG(36, 36));
    SendMessage(pData->hToolBar, WM_SETFONT, (WPARAM)g_hFont, TRUE);

    HIMAGELIST hImgList = ImageList_Create(24, 24, ILC_COLOR32 | ILC_MASK, 3, 0);
    SetWindowLongPtr(pData->hToolBar, GWLP_USERDATA, (LONG_PTR)hImgList); // Store for later cleanup

    HBITMAP hBmpBack = LoadBitmap(g_hinst, MAKEINTRESOURCE(IDB_BACKWARD));
    HBITMAP hBmpFwd = LoadBitmap(g_hinst, MAKEINTRESOURCE(IDB_FORWARD));
    HBITMAP hBmpSettings = LoadBitmap(g_hinst, MAKEINTRESOURCE(IDB_SETTINGS));
    int iBack = ImageList_Add(hImgList, hBmpBack, NULL);
    int iFwd = ImageList_Add(hImgList, hBmpFwd, NULL);
    int iGear = ImageList_Add(hImgList, hBmpSettings, NULL);
    DeleteObject(hBmpBack); DeleteObject(hBmpFwd); DeleteObject(hBmpSettings);
    SendMessage(pData->hToolBar, TB_SETIMAGELIST, 0, (LPARAM)hImgList);

    TBBUTTON tbButtons[] = {
        {iBack, IDC_FE_NAV_BACK, TBSTATE_ENABLED, BTNS_BUTTON},
        {iFwd,  IDC_FE_NAV_FORWARD, TBSTATE_ENABLED, BTNS_BUTTON},
        {0, 0, 0, BTNS_SEP},
        {iGear, IDC_FE_SHOW_SETTINGS, TBSTATE_ENABLED, BTNS_BUTTON}
    };
    SendMessage(pData->hToolBar, TB_BUTTONSTRUCTSIZE, (WPARAM)sizeof(TBBUTTON), 0);
    SendMessage(pData->hToolBar, TB_ADDBUTTONS, (WPARAM)ARRAYSIZE(tbButtons), (LPARAM)&tbButtons);

    pData->hAddressBar = CreateWindowExW(WS_EX_CLIENTEDGE, L"Edit", L"",
        WS_CHILD|WS_VISIBLE|ES_AUTOHSCROLL,
        0,0,0,0, hWnd, (HMENU)IDC_FE_ADDRESSBAR, g_hinst, NULL);
    SendMessage(pData->hAddressBar, WM_SETFONT, (WPARAM)g_hFont, TRUE);
    pData->pfnOldAddressProc = (WNDPROC)SetWindowLongPtr(pData->hAddressBar, GWLP_WNDPROC, (LONG_PTR)AddressBarProc);

    pData->hTreeView = CreateWindowEx(0, WC_TREEVIEWW, NULL,
        WS_CHILD|WS_VISIBLE|WS_BORDER|TVS_HASLINES|TVS_HASBUTTONS|TVS_LINESATROOT,
        0,0,0,0, hWnd, (HMENU)IDC_FE_TREEVIEW, g_hinst, NULL);
    SendMessage(pData->hTreeView, WM_SETFONT, (WPARAM)g_hFont, TRUE);

    pData->hListView = CreateWindowEx(0, WC_LISTVIEWW, NULL,
        WS_CHILD|WS_VISIBLE|WS_BORDER|LVS_REPORT|LVS_EDITLABELS,
        0,0,0,0, hWnd, (HMENU)IDC_FE_LISTVIEW, g_hinst, NULL);
    SendMessage(pData->hListView, WM_SETFONT, (WPARAM)g_hFont, TRUE);
    ListView_SetExtendedListViewStyle(pData->hListView, LVS_EX_FULLROWSELECT|LVS_EX_GRIDLINES|LVS_EX_DOUBLEBUFFER);

    HIMAGELIST hImgListSmall;
    SHFILEINFOW sfi = {0};
    hImgListSmall = (HIMAGELIST)SHGetFileInfoW(
        L"C:\\",
        0,
        &sfi,
        sizeof(sfi),
        SHGFI_SYSICONINDEX | SHGFI_SMALLICON
    );

    if (hImgListSmall) {
        ListView_SetImageList(pData->hListView, hImgListSmall, LVSIL_SMALL);
    }

    LVCOLUMNW lvc = {0}; lvc.mask = LVCF_TEXT|LVCF_WIDTH;
    LoadStringW(g_hinst, IDS_LV_COL_NAME, buf, 128); lvc.pszText = buf; lvc.cx = 250; ListView_InsertColumn(pData->hListView, 0, &lvc);
    LoadStringW(g_hinst, IDS_LV_COL_MODIFIED, buf, 128); lvc.pszText = buf; lvc.cx = 150; ListView_InsertColumn(pData->hListView, 1, &lvc);
    LoadStringW(g_hinst, IDS_LV_COL_TYPE, buf, 128); lvc.pszText = buf; lvc.cx = 120; ListView_InsertColumn(pData->hListView, 2, &lvc);
    LoadStringW(g_hinst, IDS_LV_COL_SIZE, buf, 128); lvc.pszText = buf; lvc.cx = 100; lvc.fmt = LVCFMT_RIGHT; ListView_InsertColumn(pData->hListView, 3, &lvc);

    pData->hStatusBar = CreateWindowEx(0, STATUSCLASSNAMEW, NULL,
        SBARS_SIZEGRIP|WS_CHILD|WS_VISIBLE,
        0,0,0,0, hWnd, (HMENU)IDC_FE_STATUSBAR, g_hinst, NULL);
    SendMessage(pData->hStatusBar, WM_SETFONT, (WPARAM)g_hFont, TRUE);
    if (DarkMode_isEnabled()) {
        DarkMode_setDarkWndNotifySafe_Default(pData->hToolBar);
        DarkMode_setDarkWndNotifySafe_Default(pData->hAddressBar);
        DarkMode_setDarkWndNotifySafe_Default(pData->hTreeView);
        DarkMode_setDarkWndNotifySafe_Default(pData->hListView);
        DarkMode_setDarkWndNotifySafe_Default(pData->hStatusBar);

        // Explicit color fixes
        ListView_SetBkColor(pData->hListView, DarkMode_getBackgroundColor());
        ListView_SetTextBkColor(pData->hListView, DarkMode_getBackgroundColor());
        ListView_SetTextColor(pData->hListView, DarkMode_getTextColor());

        TreeView_SetBkColor(pData->hTreeView, DarkMode_getBackgroundColor());
        TreeView_SetTextColor(pData->hTreeView, DarkMode_getTextColor());

        // ✅ Correct way: force redraw with dark brush (Edit controls)
        SetWindowTheme(pData->hAddressBar, L"DarkMode_Explorer", NULL);
        InvalidateRect(pData->hAddressBar, NULL, TRUE);
    }
    return TRUE;
}

void ResizeExplorerControls(HWND hWnd) {
    FileExplorerData* pData = (FileExplorerData*)GetWindowLongPtr(hWnd, GWLP_USERDATA);
    if (!pData) return;

    RECT rcClient; GetClientRect(hWnd, &rcClient);
    int toolbarH = 40, addressH = 24, statusH = 22, treeW = 250, toolbarButtonsW = 140;

    SetWindowPos(pData->hToolBar, NULL, 0, 0, rcClient.right, toolbarH, SWP_NOZORDER);
    SetWindowPos(pData->hAddressBar, NULL, toolbarButtonsW, (toolbarH-addressH)/2, rcClient.right - toolbarButtonsW - 5, addressH, SWP_NOZORDER);
    int mainY = toolbarH; int mainH = rcClient.bottom - mainY - statusH;
    SetWindowPos(pData->hTreeView, NULL, 0, mainY, treeW, mainH, SWP_NOZORDER);
    SetWindowPos(pData->hListView, NULL, treeW, mainY, rcClient.right - treeW, mainH, SWP_NOZORDER);
    SetWindowPos(pData->hStatusBar, NULL, 0, rcClient.bottom - statusH, rcClient.right, statusH, SWP_NOZORDER);
    SendMessage(pData->hStatusBar, WM_SIZE, 0, 0);
}

void ShowExplorerContextMenu(FileExplorerData* pData, POINT pt) {
    int iItem = ListView_GetNextItem(pData->hListView, -1, LVNI_SELECTED);
    if (iItem == -1) return;

    LVITEMW lvi = {0}; lvi.mask = LVIF_PARAM; lvi.iItem = iItem;
    ListView_GetItem(pData->hListView, &lvi);
    LV_ITEM_DATA* pItemData = (LV_ITEM_DATA*)lvi.lParam;
    if (!pItemData) return;

    HMENU hMenu = CreatePopupMenu();
    WCHAR buf[128];

    LoadStringW(g_hinst, IDS_FE_MENU_OPEN, buf, 128); AppendMenuW(hMenu, MF_STRING, IDM_FE_CONTEXT_OPEN, buf);
    AppendMenuW(hMenu, MF_SEPARATOR, 0, NULL);
    LoadStringW(g_hinst, IDS_FE_CONTEXT_RENAME, buf, 128); AppendMenuW(hMenu, MF_STRING, IDM_FE_CONTEXT_RENAME, buf);
    LoadStringW(g_hinst, IDS_FE_CONTEXT_DELETE, buf, 128); AppendMenuW(hMenu, MF_STRING, IDM_FE_CONTEXT_DELETE, buf);

    // Only allow download if it's a file, not a directory
    if (!pItemData->isDirectory) {
        LoadStringW(g_hinst, IDS_FE_CONTEXT_DOWNLOAD, buf, 128); AppendMenuW(hMenu, MF_STRING, IDM_FE_CONTEXT_DOWNLOAD, buf); // New Download context menu item
    }


    ClientToScreen(pData->hListView, &pt);
    TrackPopupMenu(hMenu, TPM_RIGHTBUTTON, pt.x, pt.y, 0, pData->hMain, NULL);
    DestroyMenu(hMenu);
}
void UpdateDrivesInTreeView(FileExplorerData* pData, cJSON* drivesArray) {
    if (!drivesArray || !cJSON_IsArray(drivesArray)) {
        if (drivesArray) cJSON_Delete(drivesArray);
        SetExplorerControlsEnabled(pData, TRUE);
        return;
    }

    TreeView_DeleteAllItems(pData->hTreeView);
    cJSON* item;
    cJSON_ArrayForEach(item, drivesArray) {
        if (!cJSON_IsString(item)) continue;
        wchar_t* driveW = ConvertCHARToWCHAR(item->valuestring);
        if (!driveW) continue;
        PathRemoveBackslashW(driveW);
        TVINSERTSTRUCTW tvs = {0};
        tvs.hParent = TVI_ROOT; tvs.hInsertAfter = TVI_SORT;
        tvs.item.mask = TVIF_TEXT | TVIF_PARAM | TVIF_CHILDREN;
        tvs.item.pszText = driveW;
        tvs.item.lParam = (LPARAM)_wcsdup(driveW);
        tvs.item.cChildren = 1;
        TreeView_InsertItem(pData->hTreeView, &tvs);
        free(driveW);
    }
    cJSON_Delete(drivesArray);
    SetExplorerControlsEnabled(pData, TRUE);
}

void UpdateListView(FileExplorerData* pData, cJSON* response) {
    if (!pData || pData->bShutdown || !IsWindow(pData->hListView)) {
        if (response) cJSON_Delete(response);
        return;
    }

    SendMessageW(pData->hListView, WM_SETREDRAW, FALSE, 0);
    int count = ListView_GetItemCount(pData->hListView);
    for (int i = 0; i < count; i++) {
        LVITEMW lvi = {0}; lvi.mask = LVIF_PARAM; lvi.iItem = i;
        ListView_GetItem(pData->hListView, &lvi);
        if (lvi.lParam) free((LV_ITEM_DATA*)lvi.lParam);
    }
    ListView_DeleteAllItems(pData->hListView);

    int itemCount = 0;
    if (response) {
        AddToCache(pData, pData->szCurrentPath, response);
        cJSON* items = cJSON_GetObjectItem(response, "items");
        if (cJSON_IsArray(items)) {
            cJSON* entry;
            cJSON_ArrayForEach(entry, items) {
                cJSON* nameItem = cJSON_GetObjectItem(entry, "name");
                cJSON* typeItem = cJSON_GetObjectItem(entry, "type");
                cJSON* modifiedItem = cJSON_GetObjectItem(entry, "date_modified");

                if (!cJSON_IsString(nameItem) || !cJSON_IsString(typeItem)) continue;

                wchar_t* nameW = ConvertCHARToWCHAR(nameItem->valuestring);
                if (!nameW) continue;

                wchar_t* modifiedW = NULL;
                if (modifiedItem && cJSON_IsString(modifiedItem)) {
                    modifiedW = ConvertCHARToWCHAR(modifiedItem->valuestring);
                }

                LV_ITEM_DATA* pItemData = (LV_ITEM_DATA*)malloc(sizeof(LV_ITEM_DATA));
                if (!pItemData) {
                    free(nameW);
                    free(modifiedW);
                    continue;
                }
                PathCombineW(pItemData->fullPath, pData->szCurrentPath, nameW);
                pItemData->isDirectory = (strcmp(typeItem->valuestring, "dir") == 0);

                SHFILEINFOW sfi = {0};
                DWORD dwFileAttributes = pItemData->isDirectory ? FILE_ATTRIBUTE_DIRECTORY : FILE_ATTRIBUTE_NORMAL;
                SHGetFileInfoW(
                    nameW,
                    dwFileAttributes,
                    &sfi,
                    sizeof(sfi),
                    SHGFI_ICON | SHGFI_SMALLICON | SHGFI_TYPENAME | SHGFI_USEFILEATTRIBUTES
                );

                LVITEMW lvi = {0};
                lvi.mask = LVIF_TEXT | LVIF_PARAM | LVIF_IMAGE;
                lvi.iItem = itemCount++;
                lvi.pszText = nameW;
                lvi.lParam = (LPARAM)pItemData;
                lvi.iImage = sfi.iIcon;

                int idx = ListView_InsertItem(pData->hListView, &lvi);

                ListView_SetItemText(pData->hListView, idx, 1, modifiedW ? modifiedW : L"");

                if (pItemData->isDirectory) {
                    ListView_SetItemText(pData->hListView, idx, 2, L"File Folder");
                } else {
                    ListView_SetItemText(pData->hListView, idx, 2, sfi.szTypeName);
                }

                if (!pItemData->isDirectory) {
                    cJSON* sz = cJSON_GetObjectItem(entry, "size");
                    if (cJSON_IsNumber(sz)) {
                        WCHAR sizeStr[64];
                        StrFormatByteSizeW((LONGLONG)sz->valuedouble, sizeStr, ARRAYSIZE(sizeStr));
                        ListView_SetItemText(pData->hListView, idx, 3, sizeStr);
                    }
                }

                free(nameW);
                free(modifiedW);
            }
        }
        cJSON_Delete(response);
    }

    SendMessageW(pData->hListView, WM_SETREDRAW, TRUE, 0);

    WCHAR statusText[256], fmt[128];
    LoadStringW(g_hinst, IDS_FE_STATUS_ITEMS_FMT, fmt, _countof(fmt));
    swprintf_s(statusText, ARRAYSIZE(statusText), fmt, pData->szCurrentPath, itemCount);
    SendMessageW(pData->hStatusBar, SB_SETTEXTW, 0, (LPARAM)statusText);

    SetExplorerControlsEnabled(pData, TRUE);
}

void UpdateTreeChildren(FileExplorerData* pData, HTREEITEM hParent, cJSON* itemsArray) {
    if (!itemsArray) return;

    HTREEITEM hChild = TreeView_GetChild(pData->hTreeView, hParent);
    if(hChild) TreeView_DeleteItem(pData->hTreeView, hChild);

    LPCWSTR pszParentPath = (LPCWSTR)TreeView_GetItemParam(pData->hTreeView, hParent);

    cJSON* entry;
    cJSON_ArrayForEach(entry, itemsArray) {
        cJSON* typeItem = cJSON_GetObjectItem(entry, "type");
        if (!cJSON_IsString(typeItem) || strcmp(typeItem->valuestring, "dir") != 0) continue;

        wchar_t* nameW = ConvertCHARToWCHAR(cJSON_GetObjectItem(entry, "name")->valuestring);
        if (!nameW) continue;

        WCHAR szFullPath[MAX_PATH]; PathCombineW(szFullPath, pszParentPath, nameW);
        TVINSERTSTRUCTW tvs = {0};
        tvs.hParent = hParent; tvs.hInsertAfter = TVI_SORT;
        tvs.item.mask = TVIF_TEXT | TVIF_PARAM | TVIF_CHILDREN;
        tvs.item.pszText = nameW;
        tvs.item.lParam = (LPARAM)_wcsdup(szFullPath);
        tvs.item.cChildren = 1;
        TreeView_InsertItem(pData->hTreeView, &tvs);
        free(nameW);
    }
}

void UpdateNavButtonsState(FileExplorerData* pData) {
    if (!pData || !IsWindow(pData->hToolBar)) return;
    SendMessage(pData->hToolBar, TB_SETSTATE, IDC_FE_NAV_BACK, MAKELONG(pData->iBackStackTop > -1 ? TBSTATE_ENABLED : 0, 0));
    SendMessage(pData->hToolBar, TB_SETSTATE, IDC_FE_NAV_FORWARD, MAKELONG(pData->iForwardStackTop > -1 ? TBSTATE_ENABLED : 0, 0));
}
```

**File: `src\gui\file_explorer\fe_networking.c`**

```c
// src/gui/file_explorer/fe_networking.c
#include <gui/file_explorer/fe_networking.h>
#include <gui/file_explorer/fe_ui.h>
#include <gui/file_explorer/fe_cache.h>
#include <service/network_service.h>
#include <helpers/utils.h>
#include <gui/log_page.h>
#include <stdlib.h> // For malloc, free
#include <Shlwapi.h> // For PathFindFileNameW

// FIX: New thread data structure specifically for file transfers
typedef struct {
    int clientIndex;
    HWND hNotifyWnd;
    BOOL isUpload;
    WCHAR remotePath[MAX_PATH]; // Remote file path (for download) or remote destination directory (for upload)
    WCHAR localPath[MAX_PATH];  // Local file path (for upload) or local destination path (for download)
    cJSON* jsonMetadata;        // Additional JSON to send with file
} FileTransferThreadData;

// FIX: Callback for network_service progress
static void FileTransferProgressCallback(long long bytes_transferred, long long total_bytes, void* userData) {
    FileTransferProgressInfo* progress = (FileTransferProgressInfo*)userData;
    if (progress) {
        progress->current = bytes_transferred;
        progress->total = total_bytes;
        // Post a copy of the progress info, as this callback might be called frequently.
        // The GUI thread is responsible for freeing it.
        FileTransferProgressInfo* progressCopy = (FileTransferProgressInfo*)malloc(sizeof(FileTransferProgressInfo));
        if (progressCopy) {
            *progressCopy = *progress; // Copy all fields from the live progress state
            PostMessage(progress->hNotifyWnd, WM_APP_FE_TRANSFER_PROGRESS, 0, (LPARAM)progressCopy);
        }
    }
}

// FIX: New thread for file uploads and downloads
static DWORD WINAPI FileTransferThread(LPVOID lpParam) {
    FileTransferThreadData* data = (FileTransferThreadData*)lpParam;
    if (!data) return 1;

    // Check if the main File Explorer window is still valid and not shutting down
    FileExplorerData* pDataCheck = (FileExplorerData*)GetWindowLongPtr(data->hNotifyWnd, GWLP_USERDATA);
    if (!IsWindow(data->hNotifyWnd) || !pDataCheck || pDataCheck->bShutdown) {
        PostLogMessage(L"FE: File transfer thread aborted (window closed). Cleaning up data.");
        if (data->jsonMetadata) cJSON_Delete(data->jsonMetadata);
        free(data);
        return 1;
    }

    EnterCriticalSection(&g_cs_clients);
    BOOL bClientDataValid = FALSE;
    SOCKET clientSocket = INVALID_SOCKET;
    LPCRITICAL_SECTION pClientSocketLock = NULL;

    // Make a local copy of the client data to avoid holding the lock
    if (data->clientIndex >= 0 && data->clientIndex < g_active_client_count) {
        clientSocket = g_active_clients[data->clientIndex].socket;
        pClientSocketLock = &g_active_clients[data->clientIndex].cs_socket_access;
        bClientDataValid = (clientSocket != INVALID_SOCKET);
    }
    LeaveCriticalSection(&g_cs_clients);

    if (!bClientDataValid) {
        PostLogMessage(L"FE: File transfer aborted, client at index %d no longer valid.", data->clientIndex);
        FileTransferProgressInfo* progressResult = (FileTransferProgressInfo*)malloc(sizeof(FileTransferProgressInfo));
        if (progressResult) {
            ZeroMemory(progressResult, sizeof(FileTransferProgressInfo));
            progressResult->hNotifyWnd = data->hNotifyWnd;
            progressResult->isUpload = data->isUpload;
            wcscpy_s(progressResult->fileName, MAX_PATH, data->isUpload ? PathFindFileNameW(data->localPath) : PathFindFileNameW(data->remotePath));
            PostMessage(data->hNotifyWnd, WM_APP_FE_TRANSFER_COMPLETE, 0, (LPARAM)progressResult);
        }
        goto cleanup_transfer_thread_data;
    }

    // Acquire lock for this specific client's socket for communication
    EnterCriticalSection(pClientSocketLock);

    // Re-check for shutdown *after* acquiring the socket lock
    pDataCheck = (FileExplorerData*)GetWindowLongPtr(data->hNotifyWnd, GWLP_USERDATA);
    if (!IsWindow(data->hNotifyWnd) || !pDataCheck || pDataCheck->bShutdown) {
        PostLogMessage(L"FE: File transfer thread aborted (window closed while holding socket lock). Cleaning up data.");
        LeaveCriticalSection(pClientSocketLock); // Release the lock before exiting
        goto cleanup_transfer_thread_data;
    }

    BOOL transferSuccess = FALSE;
    FileTransferProgressInfo progressState; // Live state to be updated by callback
    ZeroMemory(&progressState, sizeof(FileTransferProgressInfo));
    progressState.clientIndex = data->clientIndex;
    progressState.hNotifyWnd = data->hNotifyWnd;
    progressState.isUpload = data->isUpload;
    wcscpy_s(progressState.fileName, MAX_PATH, data->isUpload ? PathFindFileNameW(data->localPath) : PathFindFileNameW(data->remotePath));

    if (data->isUpload) {
        // Upload Logic
        PostLogMessage(L"FE: Starting upload of local '%s' to remote directory '%s'", data->localPath, data->remotePath);
        cJSON* uploadCmd = cJSON_CreateObject();
        if (!uploadCmd) {
            PostLogMessage(L"FE: Failed to create JSON object for upload command.");
            goto end_transfer;
        }
        cJSON_AddStringToObject(uploadCmd, "command", "UPLOAD_FILE");
        char* remoteDestDirUtf8 = ConvertWCHARToCHAR(data->remotePath);
        char* localFileNameUtf8 = ConvertWCHARToCHAR(PathFindFileNameW(data->localPath));
        if (remoteDestDirUtf8 && localFileNameUtf8) {
            cJSON_AddStringToObject(uploadCmd, "dest_dir", remoteDestDirUtf8);
            cJSON_AddStringToObject(uploadCmd, "file_name", localFileNameUtf8);
            free(remoteDestDirUtf8);
            free(localFileNameUtf8);
        } else {
             PostLogMessage(L"FE: Path conversion failed for upload command.");
             cJSON_Delete(uploadCmd);
             goto end_transfer;
        }

        // Send JSON metadata and the local file data as binary part
        if (SendFileRequest(clientSocket, uploadCmd, data->localPath, FileTransferProgressCallback, &progressState)) {
             PostLogMessage(L"FE: Upload of '%s' initiated.", data->localPath);
             // Client should send a response indicating success/failure of saving the file
             PacketType responseType;
             cJSON* responseJson = NULL;
             // Expect a simple JSON response back
             if (ReceiveRequest(clientSocket, &responseType, &responseJson, NULL, NULL, NULL, NULL) && responseType == PACKET_TYPE_JSON) {
                 cJSON* status = cJSON_GetObjectItem(responseJson, "status");
                 if (cJSON_IsString(status) && strcmp(status->valuestring, "success") == 0) {
                     transferSuccess = TRUE;
                     PostLogMessage(L"FE: Client reported successful upload of '%s'.", data->localPath);
                 } else {
                     PostLogMessage(L"FE: Client reported upload failed for '%s'. Status: %s", data->localPath, GetJsonString(responseJson, "message"));
                 }
                 cJSON_Delete(responseJson);
             } else {
                 PostLogMessage(L"FE: No valid response from client after upload initiation for '%s'.", data->localPath);
             }
        } else {
             PostLogMessage(L"FE: SendFileRequest failed for upload of '%s'. Network error or client disconnected.", data->localPath);
        }
        cJSON_Delete(uploadCmd); // Free the JSON created for the command
    } else {
        // Download Logic
        PostLogMessage(L"FE: Starting download of remote '%s' to local '%s'", data->remotePath, data->localPath);
        cJSON* downloadCmd = cJSON_CreateObject();
        if (!downloadCmd) {
            PostLogMessage(L"FE: Failed to create JSON object for download command.");
            goto end_transfer;
        }
        cJSON_AddStringToObject(downloadCmd, "command", "DOWNLOAD_FILE");
        char* remotePathUtf8 = ConvertWCHARToCHAR(data->remotePath);
        if (remotePathUtf8) {
            cJSON_AddStringToObject(downloadCmd, "path", remotePathUtf8);
            free(remotePathUtf8);
        } else {
            PostLogMessage(L"FE: Path conversion failed for download command.");
            cJSON_Delete(downloadCmd);
            goto end_transfer;
        }

        // Send JSON request for download, then expect a binary response
        if (SendJsonRequest(clientSocket, downloadCmd)) {
            PostLogMessage(L"FE: Download request for '%s' sent.", data->remotePath);
            PacketType responseType;
            cJSON* responseJson = NULL;
            char* fileData = NULL;
            uint32_t fileSize = 0;

            // Receive the response, which might include JSON metadata and the file itself
            if (ReceiveRequest(clientSocket, &responseType, &responseJson, &fileData, &fileSize, FileTransferProgressCallback, &progressState)) {
                cJSON* status = cJSON_GetObjectItem(responseJson, "status");
                if (cJSON_IsString(status) && strcmp(status->valuestring, "success") == 0) {
                    if (fileData) {
                        FILE* outFile;
                        if (_wfopen_s(&outFile, data->localPath, L"wb") == 0 && outFile) {
                            if (fwrite(fileData, 1, fileSize, outFile) == fileSize) {
                                transferSuccess = TRUE;
                                PostLogMessage(L"FE: Successfully downloaded '%s' to '%s'.", data->remotePath, data->localPath);
                            } else {
                                PostLogMessage(L"FE: Failed to write downloaded data to local file '%s'.", data->localPath);
                            }
                            fclose(outFile);
                        } else {
                            PostLogMessage(L"FE: Failed to open local file '%s' for writing.", data->localPath);
                        }
                    } else { // Handle 0-byte file (fileData will be NULL, fileSize 0)
                        FILE* outFile;
                        if (_wfopen_s(&outFile, data->localPath, L"wb") == 0 && outFile) {
                            fclose(outFile); // Create an empty file
                            transferSuccess = TRUE;
                            PostLogMessage(L"FE: Successfully downloaded 0-byte file '%s' to '%s'.", data->remotePath, data->localPath);
                        } else {
                            PostLogMessage(L"FE: Failed to create 0-byte local file '%s'.", data->localPath);
                        }
                    }
                } else {
                    PostLogMessage(L"FE: Client reported download failed for '%s'. Status: %s", data->remotePath, GetJsonString(responseJson, "message"));
                }
                if (responseJson) cJSON_Delete(responseJson);
                if (fileData) free(fileData);
            } else {
                PostLogMessage(L"FE: No valid response from client after download request for '%s'. Network error or client disconnected.", data->remotePath);
            }
        } else {
            PostLogMessage(L"FE: SendJsonRequest failed for download of '%s'. Network error or client disconnected.", data->remotePath);
        }
        cJSON_Delete(downloadCmd);
    }

end_transfer:
    LeaveCriticalSection(pClientSocketLock); // Release the socket lock

    // Post final completion message to the GUI thread
    FileTransferProgressInfo* progressResult = (FileTransferProgressInfo*)malloc(sizeof(FileTransferProgressInfo));
    if (progressResult) {
        *progressResult = progressState; // Copy the last known state
        progressResult->total = transferSuccess ? progressState.total : 0; // If failed, total=0 is a convention to indicate failure
        PostMessage(data->hNotifyWnd, WM_APP_FE_TRANSFER_COMPLETE, 0, (LPARAM)progressResult);
    } else {
         PostLogMessage(L"FE: Failed to allocate memory for final transfer result message.");
    }

cleanup_transfer_thread_data:
    if (data->jsonMetadata) cJSON_Delete(data->jsonMetadata);
    free(data);
    return 0;
}


// FIX: This function now populates the safer, decoupled thread data struct.
// GenericRequestThread is for JSON-only requests (listings, commands without file data).
static DWORD WINAPI GenericRequestThread(LPVOID lpParam) {
    NetworkThreadData* data = (NetworkThreadData*)lpParam;
    if (!data) return 1;

    FileExplorerData* pDataCheck = (FileExplorerData*)GetWindowLongPtr(data->hNotifyWnd, GWLP_USERDATA);
    if (!IsWindow(data->hNotifyWnd) || !pDataCheck || pDataCheck->bShutdown) {
        PostLogMessage(L"FE: Worker thread aborted (window closed). Cleaning up request data.");
        if (data->jsonRequest) cJSON_Delete(data->jsonRequest);
        free(data);
        return 1;
    }

    EnterCriticalSection(&g_cs_clients);
    BOOL bClientDataValid = FALSE;
    SOCKET clientSocket = INVALID_SOCKET;
    LPCRITICAL_SECTION pClientSocketLock = NULL;

    // Make a local copy of the client data to avoid holding the lock
    if (data->clientIndex >= 0 && data->clientIndex < g_active_client_count) {
        clientSocket = g_active_clients[data->clientIndex].socket;
        pClientSocketLock = &g_active_clients[data->clientIndex].cs_socket_access;
        bClientDataValid = (clientSocket != INVALID_SOCKET);
    }
    LeaveCriticalSection(&g_cs_clients);

    if (!bClientDataValid) {
        PostLogMessage(L"FE: Network request aborted, client at index %d no longer valid.", data->clientIndex);
        // Post NULL to signal UI to re-enable controls
        if (IsWindow(data->hNotifyWnd) && ((FileExplorerData*)GetWindowLongPtr(data->hNotifyWnd, GWLP_USERDATA))->bShutdown == FALSE) {
            PostMessage(data->hNotifyWnd, data->uNotifyMsg, 0, (LPARAM)NULL);
        }
        goto cleanup_generic_thread_data;
    }

    // --- Step 2: Perform the network operation using the obtained socket ---
    LPARAM resultLParam = 0; // Default to 0 (failure/no data)

    // Acquire lock for this specific client's socket for communication
    EnterCriticalSection(pClientSocketLock);

    // Re-check for shutdown *after* acquiring the socket lock
    pDataCheck = (FileExplorerData*)GetWindowLongPtr(data->hNotifyWnd, GWLP_USERDATA);
    if (!IsWindow(data->hNotifyWnd) || !pDataCheck || pDataCheck->bShutdown) {
        PostLogMessage(L"FE: Worker thread aborted (window closed while holding socket lock). Cleaning up request data.");
        LeaveCriticalSection(pClientSocketLock); // Release the lock before exiting
        goto cleanup_generic_thread_data;
    }

    BOOL success = SendJsonRequest(clientSocket, data->jsonRequest);
    if (success) {
        PacketType packetType;
        cJSON* jsonResponse = NULL;
        char* binaryData = NULL;
        uint32_t binarySize = 0;

        // Re-check for shutdown *before* receiving, to avoid blocking on a dead window
        pDataCheck = (FileExplorerData*)GetWindowLongPtr(data->hNotifyWnd, GWLP_USERDATA);
        if (!IsWindow(data->hNotifyWnd) || !pDataCheck || pDataCheck->bShutdown) {
            PostLogMessage(L"FE: Worker thread aborted (window closed before receiving response). Cleaning up request data.");
            goto cleanup_socket_lock; // Skip receive, go to cleanup
        }

        if (ReceiveRequest(clientSocket, &packetType, &jsonResponse, &binaryData, &binarySize, NULL, NULL)) {
            if (packetType == PACKET_TYPE_JSON && jsonResponse) {
                // Prepare the result to be sent back to the GUI thread
                switch (data->uNotifyMsg) {
                    case WM_APP_FE_UPDATE_DRIVES:
                        // "drives" array will be detached and sent directly
                        resultLParam = (LPARAM)cJSON_DetachItemFromObjectCaseSensitive(jsonResponse, "drives");
                        break;
                    case WM_APP_FE_UPDATE_LISTVIEW:
                        // The entire jsonResponse will be sent
                        resultLParam = (LPARAM)jsonResponse;
                        jsonResponse = NULL; // Detach to prevent double free
                        break;
                    case WM_APP_FE_UPDATE_TREE_CHILDREN: {
                        // Create a wrapper JSON object to hold both items and parent item handle
                        cJSON* wrapper = cJSON_CreateObject();
                        cJSON* items = cJSON_DetachItemFromObjectCaseSensitive(jsonResponse, "items");
                        cJSON_AddItemToObject(wrapper, "items", items);
                        cJSON_AddNumberToObject(wrapper, "hParent", (double)(ULONG_PTR)data->hParentItem);
                        resultLParam = (LPARAM)wrapper;
                        break;
                    }
                    default: break; // For requests that don't need a UI update (like EXEC, DELETE, RENAME)
                }
            }
            if (jsonResponse) cJSON_Delete(jsonResponse); // Free if not detached
            if (binaryData) free(binaryData);
        } else {
            PostLogMessage(L"FE: ReceiveRequest failed for client %hs.", clientSocket);
        }
    } else {
        PostLogMessage(L"FE: SendJsonRequest failed for client %hs.", clientSocket);
    }

cleanup_socket_lock:
    LeaveCriticalSection(pClientSocketLock); // Unlock the socket

    // --- Step 3: Post results back to the GUI thread safely ---
    // Check if the window is still valid AND not shutting down before posting.
    pDataCheck = (FileExplorerData*)GetWindowLongPtr(data->hNotifyWnd, GWLP_USERDATA);

    if (IsWindow(data->hNotifyWnd) && pDataCheck && !pDataCheck->bShutdown) {
        // Post the result. If resultLParam is 0 (e.g., operation failed or no data expected),
        // the GUI handler should still re-enable controls.
        PostMessage(data->hNotifyWnd, data->uNotifyMsg, 0, resultLParam);
    } else {
        // Window destroyed or shutting down during/after network operation;
        // clean up the result data here to prevent leaks as PostMessage won't be processed.
        PostLogMessage(L"FE: Window closed before UI update could be posted. Cleaning up result data.");
        if (resultLParam != 0) {
            cJSON_Delete((cJSON*)resultLParam);
        }
    }

cleanup_generic_thread_data: // Jump target for early exit if jsonRequest needs freeing
    if (data->jsonRequest) cJSON_Delete(data->jsonRequest);
    free(data);
    return 0;
}


// Helper to launch JSON-only request thread
static void LaunchRequestThread(FileExplorerData* pData, UINT uNotifyMsg, cJSON* jsonRequest, HTREEITEM hParent) {
    if (pData->bShutdown) { // Don't launch new threads if window is shutting down
        cJSON_Delete(jsonRequest);
        return;
    }

    NetworkThreadData* data = calloc(1, sizeof(NetworkThreadData));
    if (!data) {
        cJSON_Delete(jsonRequest);
        PostLogMessage(L"FE: Failed to allocate memory for network thread data.");
        return;
    }

    data->clientIndex = pData->clientIndex;
    data->hNotifyWnd = pData->hMain;
    data->uNotifyMsg = uNotifyMsg;
    data->jsonRequest = jsonRequest;
    data->hParentItem = hParent;

    SetExplorerControlsEnabled(pData, FALSE); // Disable UI while request is in progress
    WCHAR loadingMsg[64];
    LoadStringW(g_hinst, IDS_FE_STATUS_LOADING, loadingMsg, _countof(loadingMsg));
    SendMessage(pData->hStatusBar, SB_SETTEXT, 0, (LPARAM)loadingMsg);

    HANDLE hThread = CreateThread(NULL, 0, GenericRequestThread, data, 0, NULL);
    if(hThread) {
        CloseHandle(hThread); // We don't need to wait for this thread, so close its handle.
    } else {
        PostLogMessage(L"FE: Failed to create network request thread.");
        free(data);
        cJSON_Delete(jsonRequest);
        SetExplorerControlsEnabled(pData, TRUE); // Re-enable UI if thread creation fails
        SendMessage(pData->hStatusBar, SB_SETTEXT, 0, (LPARAM)L"Error: Failed to launch request.");
    }
}

// Helper to launch file transfer threads
static void LaunchFileTransferThread(FileExplorerData* pData, BOOL isUpload, LPCWSTR path1, LPCWSTR path2) {
    if (pData->bShutdown) return;

    FileTransferThreadData* data = calloc(1, sizeof(FileTransferThreadData));
    if (!data) {
        PostLogMessage(L"FE: Failed to allocate memory for file transfer thread data.");
        return;
    }

    data->clientIndex = pData->clientIndex;
    data->hNotifyWnd = pData->hMain;
    data->isUpload = isUpload;

    if (isUpload) { // path1 = local file, path2 = remote dest dir
        wcscpy_s(data->localPath, MAX_PATH, path1);
        wcscpy_s(data->remotePath, MAX_PATH, path2);
        PostLogMessage(L"FE: Preparing upload of '%s' to '%s'", data->localPath, data->remotePath);
    } else { // path1 = remote file, path2 = local dest file
        wcscpy_s(data->remotePath, MAX_PATH, path1);
        wcscpy_s(data->localPath, MAX_PATH, path2);
        PostLogMessage(L"FE: Preparing download of '%s' to '%s'", data->remotePath, data->localPath);
    }

    SetExplorerControlsEnabled(pData, FALSE);
    WCHAR initialStatus[512], fmt[128];
    UINT fmt_id = isUpload ? IDS_FE_STATUS_UPLOADING_FMT : IDS_FE_STATUS_DOWNLOADING_FMT;
    LoadStringW(g_hinst, fmt_id, fmt, _countof(fmt));
    swprintf_s(initialStatus, ARRAYSIZE(initialStatus), fmt, PathFindFileNameW(isUpload ? data->localPath : data->remotePath), 0LL, 0LL, 0.0);
    SendMessageW(pData->hStatusBar, SB_SETTEXTW, 0, (LPARAM)initialStatus);


    HANDLE hThread = CreateThread(NULL, 0, FileTransferThread, data, 0, NULL);
    if(hThread) {
        CloseHandle(hThread);
    } else {
        PostLogMessage(L"FE: Failed to create file transfer thread.");
        free(data);
        SetExplorerControlsEnabled(pData, TRUE);
        SendMessage(pData->hStatusBar, SB_SETTEXT, 0, (LPARAM)L"Error: Failed to launch transfer.");
    }
}


void RequestDrives(FileExplorerData* pData) {
    cJSON* cmd = cJSON_CreateObject();
    cJSON_AddStringToObject(cmd, "command", "GET_DRIVES");
    LaunchRequestThread(pData, WM_APP_FE_UPDATE_DRIVES, cmd, NULL);
}

void RequestDirectoryListing(FileExplorerData* pData, LPCWSTR pszPath) {
    // Attempt to retrieve from cache first
    cJSON* cachedJson = GetFromCache(pData, pszPath);
    if (cachedJson) {
        // If data is in cache, post a message to update the UI directly
        // We still disable controls briefly for visual consistency
        SetExplorerControlsEnabled(pData, FALSE);
        SendMessage(pData->hStatusBar, SB_SETTEXT, 0, (LPARAM)L"Loading from cache...");
        PostMessage(pData->hMain, WM_APP_FE_UPDATE_LISTVIEW, 0, (LPARAM)cachedJson);
        return;
    }

    char* pathUtf8 = ConvertWCHARToCHAR(pszPath);
    if (!pathUtf8) {
        PostLogMessage(L"FE: Error converting path to UTF-8: %s", pszPath);
        return;
    }

    cJSON* cmd = cJSON_CreateObject();
    cJSON_AddStringToObject(cmd, "command", "LIST");
    cJSON_AddStringToObject(cmd, "path", pathUtf8);
    cJSON_AddBoolToObject(cmd, "show_hidden", pData->bShowHidden);
    free(pathUtf8);

    LaunchRequestThread(pData, WM_APP_FE_UPDATE_LISTVIEW, cmd, NULL);
}

void RequestTreeChildren(FileExplorerData* pData, HTREEITEM hParentItem) {
    LPCWSTR path = (LPCWSTR)TreeView_GetItemParam(pData->hTreeView, hParentItem);
    if (!path) return;
    char* pathUtf8 = ConvertWCHARToCHAR(path);
    if (!pathUtf8) return;

    cJSON* cmd = cJSON_CreateObject();
    cJSON_AddStringToObject(cmd, "command", "LIST");
    cJSON_AddStringToObject(cmd, "path", pathUtf8);
    cJSON_AddBoolToObject(cmd, "show_hidden", pData->bShowHidden);
    free(pathUtf8);

    LaunchRequestThread(pData, WM_APP_FE_UPDATE_TREE_CHILDREN, cmd, hParentItem);
}

void RequestFileExecution(FileExplorerData* pData, LPCWSTR pszPath) {
    char* pathUtf8 = ConvertWCHARToCHAR(pszPath);
    if (!pathUtf8) return;

    cJSON* cmd = cJSON_CreateObject();
    cJSON_AddStringToObject(cmd, "command", "EXEC");
    cJSON_AddStringToObject(cmd, "path", pathUtf8);
    free(pathUtf8);

    // No UI update is expected for execution, so the message ID is 0.
    LaunchRequestThread(pData, 0, cmd, NULL);
}

void RequestFileDeletion(FileExplorerData* pData, LPCWSTR pszPath) {
    char* pathUtf8 = ConvertWCHARToCHAR(pszPath);
    if (!pathUtf8) return;

    cJSON* cmd = cJSON_CreateObject();
    cJSON_AddStringToObject(cmd, "command", "DELETE");
    cJSON_AddStringToObject(cmd, "path", pathUtf8);
    free(pathUtf8);

    // Invalidate the cache for the current path, as its contents have changed
    RemoveFromCache(pData, pData->szCurrentPath);

    LaunchRequestThread(pData, WM_APP_FE_REFRESH_VIEW, cmd, NULL); // Refresh view after deletion
}

void RequestFileRename(FileExplorerData* pData, LPCWSTR pszOldPath, LPCWSTR pszNewPath) {
    char* oldPathUtf8 = ConvertWCHARToCHAR(pszOldPath);
    char* newPathUtf8 = ConvertWCHARToCHAR(pszNewPath);
    if (!oldPathUtf8 || !newPathUtf8) { free(oldPathUtf8); free(newPathUtf8); return; }

    cJSON* cmd = cJSON_CreateObject();
    cJSON_AddStringToObject(cmd, "command", "RENAME");
    cJSON_AddStringToObject(cmd, "old_path", oldPathUtf8);
    cJSON_AddStringToObject(cmd, "new_path", newPathUtf8);
    free(oldPathUtf8); free(newPathUtf8);

    // Invalidate the cache for the current path, as its contents have changed
    RemoveFromCache(pData, pData->szCurrentPath);

    LaunchRequestThread(pData, WM_APP_FE_REFRESH_VIEW, cmd, NULL); // Refresh view after rename
}

// FIX: RequestFileUpload implementation
void RequestFileUpload(FileExplorerData* pData, LPCWSTR pszLocalPath, LPCWSTR pszRemoteDestDir) {
    LaunchFileTransferThread(pData, TRUE, pszLocalPath, pszRemoteDestDir);
}

// FIX: RequestFileDownload implementation
void RequestFileDownload(FileExplorerData* pData, LPCWSTR pszRemotePath, LPCWSTR pszLocalDestPath) {
    LaunchFileTransferThread(pData, FALSE, pszRemotePath, pszLocalDestPath);
}
```

**--- End of Changes ---**

**To Compile and Run:**

1.  **Replace Files:** Replace the original versions of these files with the modified content.
2.  **Recompile:** Recompile your entire project. Ensure all new headers and function signatures are correctly linked.
3.  **Client-Side Implementation:** This implementation assumes your remote client application has corresponding logic to handle the new commands:
    *   `{"command": "UPLOAD_FILE", "dest_dir": "...", "file_name": "..."}` (client should receive a file after this JSON).
    *   `{"command": "DOWNLOAD_FILE", "path": "..."}` (client should read the specified file and send it as binary data).
    *   The client also needs to use the same `compression_service` and `encryption_service` logic to correctly handle the data packets.
    *   The client needs to respond with a JSON indicating `{"status": "success"}` or `{"status": "failed", "message": "..."}` after upload/download operations to inform the server.

After these changes, your File Explorer window will have "Upload" and "Download" options, display progress in the status bar, and support drag-and-drop file uploads.
