# rpcclient.py

A Python reimplementation of Samba's `rpcclient` interactive shell, built entirely on [impacket](https://github.com/fortra/impacket).
This version was specifically designed for compatibility with other impacket components including ntlmrelayx.
The need arose for proper RPC interactions via relayed SOCKS tunnels when I noticed that existing impacket components such as net.py were not behaving properly to create local user accounts.
Notably, a specific known issue arises when creating local users for subsequent exploitation. Performing these operations with rpcclient.py instead circumvents the issue and enables exploitation via the created accounts.
> **Authorized testing only.** This tool creates/deletes accounts, changes passwords, modifies group membership, and edits the registry on remote Windows systems. Use it only against systems you own or are explicitly authorized to test.

---
## Usage

```bash
./rpcclient.py [domain/]user[:password]@target
./rpcclient.py -hashes LM:NT domain/user@target          # pass-the-hash
./rpcclient.py -no-pass domain/user@target                # anonymous / prompt
./rpcclient.py -k domain/user@target                      # Kerberos
proxychains rpcclient.py user@target -no-pass             # SOCKS tunnel
```

### Connection flags

| Flag | Description |
|---|---|
| `-hashes LMHASH:NTHASH` | NTLM pass-the-hash authentication |
| `-no-pass` | Don't prompt for a password (anonymous, or used with `-hashes`) |
| `-k` | Kerberos authentication (pulls credentials from ccache) |
| `-aesKey <hex key>` | AES key for Kerberos (128 or 256 bit) |
| `-dc-ip <ip>` | IP of a domain controller, for Kerberos |
| `-port {139,445}` | Destination port (default 445) |
| `-timeout <seconds>` | Connection timeout (raise this over slow SOCKS/proxychains tunnels) |
| `-socks-proxy host:port` | Route all traffic through a SOCKS5 proxy natively, without needing `proxychains` (requires PySocks) |
| `-dialect {1,2.02,2.1,3.0,3.0.2,3.1.1}` | Force a specific SMB dialect instead of letting impacket negotiate. Mostly useful for diagnosing session-key issues (see `smbinfo` below) — `-dialect 1` only helps if the target still has legacy SMB1 enabled, which most current Windows builds do not |
| `-c "cmd1;cmd2"` | Run a semicolon-separated list of commands non-interactively and exit |
| `-debug` | Print full Python tracebacks on errors, not just the summary message |

---

## Command reference

### SAMR — users

| Command | Description |
|---|---|
| `enumdomusers` | Enumerate domain/local users |
| `queryuser <rid\|username>` | Query full user info (name, RID, account flags, etc.) |
| `createdomuser <username> [password] [--enable] [--must-change]` | Create a user. Optionally sets a password and enables the account in the same step. **Unlike stock rpcclient**, the account is guaranteed *not* to require a password change at next logon unless `--must-change` is explicitly passed — see [Beyond stock rpcclient](#beyond-stock-rpcclient) |
| `deletedomuser <username>` | Delete a user |
| `enableuser <rid\|username>` | Clear the account-disabled flag |
| `disableuser <rid\|username>` | Set the account-disabled flag |
| `setuserpass <username> [newpassword] [--must-change]` | Admin-side password reset (no knowledge of the old password required) |
| `chgpasswd <username> [old_nthash]` | Self-service password change (you must know the current password, or its NT hash) |

### SAMR — groups & aliases (local groups)

| Command | Description |
|---|---|
| `enumdomgroups` | Enumerate domain global groups |
| `querygroup <rid>` | Query group info |
| `querygroupmem <rid>` | List a group's members (RIDs) |
| `createdomgroup <name>` | Create a domain global group |
| `deletedomgroup <rid\|name>` | Delete a domain global group |
| `addgroupmem <group rid\|name> <user rid\|name>` | Add a user to a domain global group |
| `delgroupmem <group rid\|name> <user rid\|name>` | Remove a user from a domain global group |
| `enumalsgroups [builtin\|domain]` | Enumerate aliases (local groups) |
| `createdomalias <name> [--builtin]` | Create a local group |
| `deletedomalias <rid\|name> [--builtin]` | Delete a local group |
| `addaliasmem [--builtin] <alias rid\|name> <member: sid\|rid\|username>` | Add a member to a local group (e.g. the built-in **Administrators** group) |
| `delaliasmem [--builtin] <alias rid\|name> <member: sid\|rid\|username>` | Remove a member from a local group |
| `listaliasmem [--builtin] <alias rid\|name>` | List a local group's members (SIDs) |

> Domain global groups and local groups (aliases) are different SAMR containers. `Administrators`, `Remote Desktop Users`, etc. live in the **BUILTIN** domain — reach them with `--builtin` and the `*aliasmem`/`enumalsgroups` commands, not `*groupmem`/`enumdomgroups`.

### SAMR — misc

| Command | Description |
|---|---|
| `querydominfo` | Domain name, SID, and user/group/alias counts |
| `getdompwinfo` | Domain password policy (min length, complexity flags) |
| `lookupnames <name> [name...]` | Resolve names to SIDs |
| `lookupsids <sid> [sid...]` | Resolve SIDs to names (via LSA) |

### LSA

| Command | Description |
|---|---|
| `lsaquery` | Query the LSA policy for domain name/SID |
| `lsaenumsid` | Enumerate accounts known to LSA |
| `enumprivs` | Enumerate LSA privileges |

### SRVSVC / WKSSVC

| Command | Description |
|---|---|
| `srvinfo` | Server info (name, OS version, type) |
| `netshareenum` | Enumerate shares |
| `netsharegetinfo <share>` | Detailed info on one share |
| `netserverdiskenum` | Enumerate disks |
| `wkstainfo` | Workstation info |

### Remote Registry (WINREG/SVCCTL)

| Command | Description |
|---|---|
| `regsetdword <HKLM\key\path> <ValueName> <dword>` | Set a `REG_DWORD` value under `HKEY_LOCAL_MACHINE`. Automatically starts the Remote Registry service if it's stopped or disabled |
| `fixblankpasswordpolicy [on\|off]` | Toggle *"Accounts: Limit local account use of blank passwords to console logon only"*. `off` (default) lets a blank-password local account authenticate over the network; `on` restores the secure default |
| `fixuactokenfilter [on\|off]` | Toggle UAC remote restrictions (`LocalAccountTokenFilterPolicy`). `off` (default) makes a local admin account get a **full** admin token over the network instead of a UAC-filtered one; `on` restores the secure default |

Both `fix*` commands track whether *this session* had to auto-enable the Remote Registry service to reach the registry at all, and their `on` (undo) form puts the service back to `Disabled` if so — they won't touch it if it was already enabled/running beforehand.

### Diagnostics

| Command | Description |
|---|---|
| `smbinfo` | Show the negotiated SMB dialect and session-key length for the underlying connection — the first thing to check if `setuserpass`/`createdomuser <user> <pass>` fails |

### Generic / arbitrary RPC

| Command | Description |
|---|---|
| `bind <\pipe\name> <interface-uuid> [major.minor]` | Bind to an arbitrary named pipe/interface, for interfaces impacket has no typed stub for |
| `rawcall <opnum> [hex-payload]` | Send a raw opnum + NDR-encoded hex body on the interface from `bind`, and print the raw hex response |

---

## Beyond stock rpcclient

A few things here exist specifically because real `rpcclient` doesn't do them, or does them in a way that leaves accounts unusable in practice:

- **`createdomuser` doesn't set passord_must_change** Real rpcclient's `createdomuser` only calls `SamrCreateUser2` — no password parameter exists at all, leaving every new account disabled and passwordless. This version optionally sets the password, enables the account, and clears the disabled flag in one command.

- **full SOCKS tunnel compatibility** This version will work using your relay authentication in the SOCKS tunnel without problem.
  
- **`fixblankpasswordpolicy` / `fixuactokenfilter`** exist because two extremely common, on-by-default Windows policies otherwise make a perfectly correctly-created local account look broken over the network:
  - A local account with a **blank password** cannot authenticate over the network at all by default (console/physical logon only) — regardless of whether it's enabled, in Administrators, etc.
  - A local account that genuinely **is** a member of Administrators still gets a UAC-filtered, non-admin token on network logon by default — it can establish a session and browse `IPC$`, but gets denied on `C$`/`ADMIN$` and can't do remote command execution, with no error suggesting why. (The built-in RID-500 `Administrator` account is exempt from this by default, which is why it never shows this symptom.)

  Both are genuine Windows security controls, not bugs — these commands just make it possible to toggle them from an existing admin RPC session instead of needing console/RDP access to the target.

---

## A note on `-dialect` / `smbinfo`

Some SAMR password-set operations encrypt their payload using the underlying **SMB session key**. impacket only derives that key for SMB3 connections that actually negotiated message encryption — if the session lands on SMB 2.x, or encryption capability doesn't come through cleanly (this can happen over some proxied/tunneled paths), the key stays empty and those calls fail with `STATUS_WRONG_PASSWORD` rather than succeeding. Run `smbinfo` to check the negotiated dialect and session-key length if you hit this. `-dialect 1` (forcing legacy SMB1) sidesteps the dependency entirely, but only works if the target still has the SMB1 server component enabled — most current Windows builds do not, and forcing it against one that doesn't will just break the connection with a protocol-mismatch error instead of falling back gracefully.

## License

No license specified — treat as source-available for personal/internal use unless you know otherwise.
