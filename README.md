# bincapz

![bincapz logo](./images/logo_small.jpg)

Experimental tool to enumerate capabilities and detect malicious behavior within binaries through fragment analysis.

![screenshot](./images/screenshot.png)

## Features

- Analyzes binaries from any architecture - arm64, amd64, riscv, ppc64, sparc64
- Supports scripting languages such as bash, PHP, Perl, Ruby, NodeJS, and Python
- 12,000+ rules, for detecting everything from ioctl access to malware
- Integrates [YARA forge](https://yarahq.github.io/), with rules by Avast, Elastic, FireEye, Google, Mandiant, Nextron, and more.
- Diff-friendly JSON output to detect when capabilities change over time
- CI/CD friendly

## Shortcomings

- This tool is in early development with unstable output
- Does not attempt to process archive files (jar, zip, apk)
- Minimal rule support for Windows executables (help wanted!)

## Installation

```shell
go install github.com/chainguard-dev/bincapz@latest
```

## Usage

To inspect a binary, pass it as an argument to dump a list of predicted capabilities:

```shell
bincapz /bin/ping
```

```
/sbin/ping
--------------------------------------------------------------------------------------------------
  RISK  |          KEY          |                           DESCRIPTION
--------+-----------------------+-------------------------------------------------------------------
  meta  | sha256                | 1eec23e4189171ea689c7fe6a133e5f22b9683f633e414bde9ca47b9644f090b
  meta  | entitlements          | com.apple.private.network.management.data.development
        |                       | com.apple.security.network.client
        |                       | com.apple.security.network.server
        |                       |
  1/LOW | net/hostname/resolve  | resolves network hosts via name
  1/LOW | net/http/request      | Makes HTTP (Hypertext Transport Protocol) requests
  1/LOW | net/icmp              | ICMP (Internet Control Message Protocol), aka ping
  1/LOW | net/interface/get     | get network interfaces by name or index
  1/LOW | net/interface/list    | list network interfaces and their associated addresses
  1/LOW | net/ip                | access the internet
  1/LOW | net/ip/multicast/send | send data to multiple nodes simultaneously
  1/LOW | net/ip/resolve        | resolves network hosts via IP address
  1/LOW | net/ip/send/unicast   | send data to the internet
  1/LOW | net/socket/connect    | initiate a connection on a socket
  1/LOW | net/socket/receive    | receive a message from a socket
  1/LOW | net/socket/send       | send a message to a socket
  1/LOW | process/userid/set    | set real and effective user ID of current process
  2/MED | combo/net/scan_tool   | may scan networks:
        |                       | connect gethostbyname port scan socket
  2/MED | net/ip/string         | converts IP address from byte to string
```

That seems low-risk to me. Now, let's analyze a suspected malicious binary:

```log
bpfdoor_2022.x86_64
------------------------------------------------------------------------------------------------------------------------------
RISK  |                 KEY                 |                                 DESCRIPTION
---------+-------------------------------------+------------------------------------------------------------------------------
meta   | sha256                              | fd1b20ee5bd429046d3c04e9c675c41e9095bea70e0329bd32d7edd17ebaf68a
|                                     |
1/LOW  | exec/program/background             | Waits for a process to exit
1/LOW  | fd/multiplex                        | monitor multiple file descriptors
1/LOW  | fs/file/delete                      | deletes files
1/LOW  | net/socket/connect                  | initiate a connection on a socket
1/LOW  | net/socket/listen                   | listen on a socket
1/LOW  | net/socket/receive                  | receive a message from a socket
1/LOW  | net/socket/send                     | send a message to a socket
1/LOW  | process/chroot                      | change the location of root for the process
1/LOW  | process/create                      | Create a new child process using fork
1/LOW  | random/insecure                     | generate random numbers insecurely
1/LOW  | ref/path/usr/sbin                   | References paths within /usr/sbin:
|                                     | /usr/sbin/console-kit-daemon
1/LOW  | tty/vhangup                         | virtually hangup the current terminal
2/MED  | device/pseudo_terminal              | pseudo-terminal access functions
2/MED  | exec/program                        | executes another program
2/MED  | exec/shell_command                  | execute a shell command
2/MED  | fs/file/times/set                   | change file last access and modification times
2/MED  | net/ip/byte/order                   | convert values between host and network byte order
2/MED  | net/ip/string                       | converts IP address from byte to string
3/HIGH | combo/backdoor/net_term             | Listens, provides a terminal, runs program:
|                                     | /dev/ptmx execve grantpt listen
3/HIGH | combo/backdoor/sys_cmd              | multiple sys commands:
|                                     | auditd systemd/systemd
3/HIGH | ref/program/ancient_gcc             | built by archaic gcc version:
|                                     | GCC: (GNU) 4.4.7
4/CRIT | 3P/elastic/bpfdoor                  | Detects Linux Trojan Bpfdoor (Linux.Trojan.BPFDoor), by Elastic Security
4/CRIT | 3P/signature_base/redmenshen/bpfd.. | Detects BPFDoor implants used by Chinese actor Red Menshen, by Florian Roth
|                                     | (Nextron Systems)
```

If you want to focus on the most suspicious behaviors, you can pass `--min-level=3`, which will remove a lot of the noise by only showing "HIGH" or "CRITICAL" risk behaviors.

## Detecting supply-chain compromises

Let's say you are a company that is sensitive to supply-chain compromises. You can use the JSON output to store a record of known capabilities for each file, for instance:

```

bincapz --format=json 3CX_clean_ffmpeg.dylib | jq  '.Files.[].Behaviors | keys'

```

which outputs:

```json

[

"combo/exfil/connect_glob_exec",

"crypto/aes",

"encoding/base64",

"fs/directory/create",

"net/http/post",

"net/url",

"process/thread/create",

"ref/path/tmp"

]

```

Now you can detect during PR review or release time if you have suffered a supply-chain attack. Here's what the diff would look like if you suffered the same compromise as 3CX did:

```diff

[

+  "3P/godmoderules/iddqd/god/mode",

+  "3P/signature_base/3cxdesktopapp/macos",

+  "3P/signature_base/macos/nk",

+  "3P/signature_base/susp/xored",

+  "3P/volexity/mac/iconic",

"combo/exfil/connect_glob_exec",

+  "compression/gzip",

"crypto/aes",

"encoding/base64",

+  "env/HOME",

+  "exec/pipe",

"fs/directory/create",

+  "fs/lock/update",

+  "fs/permission/modify",

+  "kernel/dispatch/semaphore",

+  "kernel/hostname/get",

+  "net/http/accept/encoding",

+  "net/http/cookies",

"net/http/post",

"net/url",

+  "net/url/request",

"process/thread/create",

-  "ref/path/tmp"

+  "random/insecure",

+  "ref/path/hidden",

+  "ref/path/tmp",

+  "shell/arbitrary_command/dev_null",

+  "sync/semaphore/user"

]

```

That is a lot of unusual new capabilities for a video processing library.

## Supported Flags

* `-all` - Don't filter anything out, include even harmless capabilities
* `-alsologtostderr` - log to standard error
* `-format string` - Output type. Valid values are: table, simple, json, yaml (default "table")
*  `-ignore-tags` - Rule tags to ignore
* `-include-data-files` - include files that are detected to as non-program (binary or source) files
* `-min-level` - minimum suspicion level to report (1=low, 2=medium, 3=high, 4=critical) (default 1)
* `-omit-empty` - don't report on files that have no matches
* `-third-party` - include third-party rules, which may have licensing restrictions (default true)

## FAQ

### How does it work?

Through the execution of 12,000+ YARA rules.

bincapz automates the initial triage step used by any security analyst when faced with an unknown binary:  `strings`. While this seems absurdly simple, it is exceptionally effective, as every binary leaves traces of its capabilities in its contents, particularly on UNIX platforms. These fragments are typically  `libc` or `syscall` references or error codes. Due to the C-like background of many scripting languages such as PHP or Perl, the same fragment detection rules often apply.

### Why not properly reverse-engineer binaries?

Mostly because fragment analysis is so effective. Capability analysis through reverse engineering is challenging to get right, particularly for programs that execute other programs, such as malware that executes `/bin/rm`. Capability analysis through reverse engineering that supports a wide array of file formats also requires significant engineering investment.

### Why not just observe binaries in a sandbox?

The most exciting malware only triggers when the right conditions are met. Nation-state actors in particular are fond of time bombs and locale detection. bincapz will enumerate the capabilities, regardless of conditions.

### Why not just analyze the source code?

Sometimes you don't have it! Sometimes your CI/CD infrastructure is the source of compromise. Source-code-based capability analysis is also complicated for polyglot programs, or programs that execute external binaries, such as `/bin/rm`.

### How does bincapz work for packed binaries?

bincapz alerns when an obfuscated or packed binary is detected. Depending on the packer used, fragment analysis may still work to a lesser degree. For the full story, we recommend unpacking binaries first.

### What related software is out there?

Much of bincapz's functionality is inspired by <https://github.com/mandiant/capa>. While capa is a fantastic tool, it only works on x86-64 binaries (ELF/PE), and does not work for macOS programs, arm64 binaries, or scripting languages. <https://karambit.ai/> and <https://www.reversinglabs.com/> offer capability analysis through reverse engineering as a service. If you require more than what bincapz can offer, such as Windows binary analysis, you should check them out.
