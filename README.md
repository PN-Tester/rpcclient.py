# rpcclient.py

A Python implementation of Samba's `rpcclient` interactive shell, built with [impacket](https://github.com/fortra/impacket).
This version was specifically designed for compatibility with other impacket components including `ntlmrelayx`.
Additional features have been added to facilitate local account creation and subsequent usage for exploitation.

## Usage

```bash
./rpcclient.py [domain/]user[:password]@target
./rpcclient.py domain/user@target -hashes LM:NT           # pass-the-hash
./rpcclient.py domain/user@target -no-pass                # anonymous
./rpcclient.py domain/user@target  -k                     # Kerberos
proxychains rpcclient.py user@target -no-pass             # SOCKS tunnel
```

### Rationale
The tool exists to offer more granular control of RPC interaction to penetration testers during internal network engagements. When operators relay NTLM authentication over SMB and create a SOCKS tunnel, a common method
of solidifying this foothold is through creation of a local account on the target computer. Subsequently escalating this account to local administrator of the victim machine is trivial, but *actually using* the account is not!
Specifically, a common issue arises when using net.py through an ntlmrelay SOCKS to create users or reset passwords. The RPC method call used to set the password requires a session key to encrypt the value, but because we are in a SOCKS tunnel, there is no session key we can access. Thus, the password of the created account is always blank.
While a blank password is perfectly fine from an authentication standpoint (we can provide '' or the NT hash ```31D6CFE0D16AE931B73C59D7E0C089C0``` for credentials), it presents all sorts of problems in windows land.
Whats worse, you cannot reset or change the blank password with :
1) the original relayed admin session (because we dont have a session key)
2) the created account itself (because its password must change, so it can't authenticate, stuck in a loop)
3) a null session (because we dont have permission)

rpcclient.py does not have this issue, because we have modified the account creation to process to clear the "password must change at next logon" flag.
Even so, the account will remain unuseable by default due to restrictions on network logons using accounts with blank password (default policy).
The fix is to enable remote registry on the target, and change the value of the key in HKLM to allow us to authenticate with the blank password having user. 
This can be done with ```fixblankpasswordpolicy```. Now the user can authenticate but only to IPC$, not C$ or ADMIN$, or do anything spicy. The reason, despite being a local admin, is that there is another default policy that
limits the token privileges a local admin gets over the network. If we want an admin session, we need to change this registry value as well with ```fixuactokenfilter```.
When combined with the ```fixblankpasswordpolicy``` and ```fixuactokenfilter``` commands, the created account is ready for usage out of the box.

### Demo
![](https://github.com/PN-Tester/rpcclient.py/blob/main/RPCCLIENT_EXAMPLE.png)
*In the above example, we use rpcclient.py through a SOCKS tunnel to create and escalate the local "hacker", before dumping SAM and LSA secrets from the target with the new account*

### Connection flags

| Flag | Description |
|---|---|
| `-hashes LMHASH:NTHASH` | NTLM pass-the-hash authentication |
| `-no-pass` | Don't prompt for a password (useful for -k or relays) |
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


## License

No license specified — treat as source-available for personal/internal use unless you know otherwise.
