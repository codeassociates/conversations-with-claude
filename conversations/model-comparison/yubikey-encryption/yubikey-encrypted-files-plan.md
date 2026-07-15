# YubiKey-backed encrypted files and SSH

This runbook describes a cross-platform arrangement for:

- OpenPGP-encrypted files stored on removable or synchronized storage.
- An OpenPGP encryption key held on multiple YubiKeys.
- Existing file-based SSH keys, without writing their plaintext to disk.
- A native OpenPGP authentication subkey for newly provisioned SSH hosts.
- Android access through OpenKeychain, Termux, and OkcAgent.
- Windows-native GPG access from WSL2, without assigning the entire YubiKey to WSL.

## Key layout

Create the keys on an offline Linux system:

- Certification-only primary key: retained offline.
- Encryption subkey: copied to every YubiKey; decrypts files.
- Authentication subkey: copied to every YubiKey; acts as an SSH key.
- Signing subkey: optional; useful for Git and document signing.

Keep a passphrase-protected offline backup containing all private key material. A
key generated on a YubiKey cannot be exported, so generate keys externally if the
same keys must be installed on several tokens.

RSA 3072 is a conservative choice for interoperability across GnuPG,
OpenKeychain, and different YubiKey generations.

## 1. Generate and back up the keys

Run this from an offline live Linux environment. Put `GNUPGHOME` on encrypted
offline media, not in the normal user keyring.

```bash
export GNUPGHOME=/path/on/encrypted-media/gnupg
mkdir -m 700 "$GNUPGHOME"

gpg --quick-generate-key \
  "Personal Vault <vault@example.invalid>" \
  rsa3072 cert 2y

gpg --list-secret-keys --with-subkey-fingerprint
```

Copy the full primary-key fingerprint from the output and add the subkeys:

```bash
gpg --quick-add-key PRIMARY_FINGERPRINT rsa3072 encr 2y
gpg --quick-add-key PRIMARY_FINGERPRINT rsa3072 auth 2y
gpg --quick-add-key PRIMARY_FINGERPRINT rsa3072 sign 2y
```

Export the public key, complete private backup, and a revocation certificate:

```bash
gpg --armor --export PRIMARY_FINGERPRINT > public.asc
gpg --armor --export-secret-keys PRIMARY_FINGERPRINT > secret-full.asc
gpg --armor --export-secret-subkeys PRIMARY_FINGERPRINT > secret-subkeys.asc
gpg --output revocation.asc --gen-revoke PRIMARY_FINGERPRINT
```

Store at least two encrypted backup copies separately. Test `secret-full.asc` by
importing it into a different temporary `GNUPGHOME` before provisioning tokens.

## 2. Provision each YubiKey

Before provisioning, change the OpenPGP user and administrator PINs. Optionally
use YubiKey Manager to require touch for private-key operations.

For **each** token, create a fresh temporary `GNUPGHOME` and restore
`secret-full.asc`. Do not reuse the keyring modified while provisioning the
previous token: GnuPG's `keytocard` operation replaces local private-key material
with a card reference.

```bash
export GNUPGHOME=/secure/temporary/location/token-1
mkdir -m 700 "$GNUPGHOME"
gpg --import public.asc
gpg --import secret-full.asc
gpg --edit-key PRIMARY_FINGERPRINT
```

At the `gpg>` prompt, use `list` to identify the subkey numbers. Select exactly
one subkey at a time with `key N`, then run `keytocard`:

| Subkey | OpenPGP card slot |
|---|---|
| Signing | Signature key |
| Encryption | Encryption key |
| Authentication | Authentication key |

Leave the certification primary key offline. After saving, verify and perform a
test encryption/decryption:

```bash
gpg --card-status
gpg --list-secret-keys --with-subkey-fingerprint
printf 'YubiKey test\n' | gpg --armor --encrypt \
  --recipient PRIMARY_FINGERPRINT > test.txt.asc
gpg --decrypt test.txt.asc
```

Repeat from a fresh restored backup for every additional YubiKey.

## 3. Encrypt files

Encryption requires only `public.asc`; decryption requires the encryption subkey
on a YubiKey or the offline software-key backup.

```bash
gpg --import public.asc
gpg --output secrets.tar.gpg --encrypt \
  --recipient PRIMARY_FINGERPRINT secrets.tar

gpg --output secrets.tar --decrypt secrets.tar.gpg
```

OpenPGP is file/stream encryption, not a mounted filesystem like VeraCrypt.
Archive directories before encrypting them. Outer filenames, sizes, and dates may
remain visible.

Avoid writing decrypted secrets to SD cards, SSDs, or shared Android storage;
reliable secure deletion is generally not possible on flash media.

## 4. Existing file-based SSH keys

Keep each existing private key encrypted, preserving its corresponding public key
and all existing server configuration:

```bash
gpg --output legacy_id_ed25519.gpg --encrypt \
  --recipient PRIMARY_FINGERPRINT ~/.ssh/legacy_id_ed25519
```

After verifying that the encrypted copy can be decrypted, remove the plaintext
original using an appropriate migration procedure. Do not assume that `shred`
reliably erases data from an SSD or SD card.

For use, decrypt directly into a conventional SSH agent rather than into a file:

```bash
eval "$(ssh-agent -s)"
gpg --decrypt legacy_id_ed25519.gpg | ssh-add -
ssh-add -l
ssh user@legacy-host
```

Remove the in-memory key when finished:

```bash
ssh-add -D
ssh-agent -k
```

The server sees the original SSH key and requires no GPG support.

## 5. Native OpenPGP SSH authentication

An OpenPGP authentication subkey is also presented to servers as an ordinary SSH
public key; the server does not run GPG. It is nevertheless a new public key that
must be added to `authorized_keys`:

```bash
gpg --export-ssh-key PRIMARY_FINGERPRINT
```

On Linux, enable GnuPG's SSH-agent interface in `~/.gnupg/gpg-agent.conf`:

```text
enable-ssh-support
```

Then restart the agent and select its socket:

```bash
gpgconf --kill gpg-agent
gpgconf --launch gpg-agent
export SSH_AUTH_SOCK="$(gpgconf --list-dirs agent-ssh-socket)"
ssh-add -L
```

FIDO2 SSH keys are a good future/additional credential, but should not currently
be the only Android access path. Termux still lacks a broadly established bridge
between OpenSSH's security-key provider and Android USB/NFC APIs.

## 6. Android and Termux

Install OpenKeychain, Termux, and both components of OkcAgent. Import only
`public.asc` into OpenKeychain and configure the YubiKey as the security token; do
not import `secret-full.asc` unless intentionally using an Android software key.

In Termux:

```bash
pkg install okc-agents openssh
eval "$(okc-ssh-agent)"
```

Use `okc-ssh-agent` for the native OpenPGP authentication subkey. For a legacy
encrypted SSH private key, use an ordinary Termux `ssh-agent` and pipe OkcAgent's
decryption output into `ssh-add -`. `okc-gpg` supports only a subset of GPG
arguments, so confirm the exact syntax against the installed version:

```bash
eval "$(ssh-agent -s)"
okc-gpg --decrypt legacy_id_ed25519.gpg | ssh-add -
```

If a plaintext file is temporarily unavoidable, place it under Termux's private
home directory with mode `0600`, never on shared storage or the removable SD card.

## 7. Windows-native GPG from WSL2

Install Gpg4win and let its native `gpg-agent`/`scdaemon` own the YubiKey. This
keeps the token available to Windows and avoids `usbipd`, which makes an attached
USB device unavailable to Windows while WSL owns it.

Find the native executable in PowerShell:

```powershell
Get-Command gpg.exe
```

Call it from WSL using the returned installation path. For example:

```bash
GPG_WIN='/mnt/c/Program Files (x86)/GnuPG/bin/gpg.exe'
"$GPG_WIN" --card-status
```

For a legacy SSH key, stream the encrypted data through native Windows GPG into a
WSL SSH agent. This avoids Windows/Linux pathname conversion:

```bash
eval "$(ssh-agent -s)"
"$GPG_WIN" --decrypt < legacy_id_ed25519.gpg | ssh-add -
```

If Windows GPG must open a named file, convert the path:

```bash
encrypted_key=$(wslpath -w "$PWD/legacy_id_ed25519.gpg")
"$GPG_WIN" --decrypt "$encrypted_key" | ssh-add -
```

Calling Windows `gpg.exe` is simpler than forwarding GPG-agent sockets into WSL.
Use an agent bridge only if Linux-native programs specifically need direct access
to the Windows GPG agent. Gpg4win and Yubico Authenticator may occasionally
contend for smart-card access; closing one application or running
`gpgconf.exe --kill scdaemon` normally releases it without detaching the token
from Windows.

## 8. Recovery and maintenance

- Test every YubiKey for both file decryption and SSH authentication.
- Record fingerprints and token serial numbers separately from the tokens.
- Periodically test the offline backup in an isolated temporary keyring.
- Before subkeys expire, use the offline primary key to extend or replace them.
- Keep another working token available before changing or resetting one.
- If a token is lost, revoke/replace its subkeys where practical and provision a
  replacement from the offline backup.

To use the key without a token on a trusted isolated machine:

```bash
export GNUPGHOME=/trusted/private/location
mkdir -m 700 "$GNUPGHOME"
gpg --import secret-full.asc
```

This deliberately gives that machine software access to the private keys and
should be treated as a recovery or controlled offline mode.

## Important references

- [Yubico: Importing OpenPGP keys](https://developers.yubico.com/PGP/Importing_keys.html)
- [Yubico: OpenPGP walkthrough](https://developers.yubico.com/PGP/PGP_Walk-Through.html)
- [GnuPG manual](https://www.gnupg.org/documentation/manuals/gnupg.pdf)
- [OpenKeychain](https://www.openkeychain.org/)
- [OpenKeychain FAQ and security-token notes](https://www.openkeychain.org/faq/)
- [OkcAgent for OpenKeychain and Termux](https://github.com/DDoSolitary/OkcAgent)
- [Termux private and shared storage model](https://github.com/termux/termux-packages/wiki/Termux-file-system-layout)
- [Microsoft: Connect USB devices to WSL](https://learn.microsoft.com/en-us/windows/wsl/connect-usb)
- [Win32 OpenSSH releases](https://github.com/PowerShell/Win32-OpenSSH/releases)
