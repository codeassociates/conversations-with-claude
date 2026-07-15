# Yubikey-backed file encryption — deployment plan

A cross-platform (Linux/WSL2 + Android) replacement for a Veracrypt workflow, using
OpenPGP public-key encryption with the private **Encryption** subkey held on one or
more Yubikeys. Encrypted data lives on removable/synced storage; plaintext is only
ever produced on demand (pipe / tmpfs), never written to the SD card.

## Design at a glance

- **Primitive:** OpenPGP asymmetric encryption. Files are encrypted *to your public key*;
  the Yubikey's on-card **Encryption (E)** subkey decrypts them. The public key is not
  secret — put it on every device.
- **The Yubikey holds exactly one E subkey.** That single slot decrypts *arbitrarily many*
  files, so there is no per-secret or per-SSH-key limit. (The OpenPGP applet's one-Auth-slot
  limit only bites if you use the Yubikey's Auth subkey *directly* as an SSH identity — we
  don't; see [SSH keys](#ssh-keys-into-ssh-agent).)
- **Keys are generated offline and kept as an exportable backup**, so the same E subkey can
  be provisioned to multiple Yubikeys and used "bare" without a token when needed.
- **Files:** `pass` for passwords/short secrets; plain `gpg -e` files for whole SSH key blobs.
- **Android:** OpenKeychain drives the Yubikey (NFC/USB); Termux delegates to it via
  `okc-gpg`. Termux cannot talk to the Yubikey directly.
- **Desktop/WSL2:** relay the Windows `gpg-agent` (Gpg4win) socket into WSL2 so the Yubikey
  stays shared with Windows.

> **Security tradeoff, stated up front:** keeping the E subkey exportable defeats the
> hardware token's non-extractability guarantee. Your offline backup *is* the plaintext —
> anyone with the backup file **and** its passphrase can decrypt everything, forever, with
> no Yubikey. Guard it accordingly. This is a deliberate choice to support multi-token /
> bare use.

---

## 1. One-time key ceremony (do this offline)

Run on an airgapped or freshly-booted machine. Using a throwaway `GNUPGHOME` keeps it out of
your daily keyring.

```bash
export GNUPGHOME=$(mktemp -d)
chmod 700 "$GNUPGHOME"

# Certify-only master key (kept offline). ed25519 for Yubikey 5; use rsa4096 if any
# token is a NEO/4 or you need maximum interop.
gpg --quick-generate-key 'Your Name <you@example.com>' ed25519 cert never

# Capture the fingerprint
export KEYFPR=$(gpg --list-keys --with-colons | awk -F: '/^fpr:/ {print $10; exit}')

# Subkeys: sign, encrypt, authenticate
gpg --quick-add-key "$KEYFPR" ed25519 sign 2y
gpg --quick-add-key "$KEYFPR" cv25519 encr 2y
gpg --quick-add-key "$KEYFPR" ed25519 auth 2y
```

### Back up everything BEFORE touching a Yubikey

`keytocard` is **destructive** — it replaces the on-disk subkey with a stub. This backup is
both your multi-token source and your "use it bare" copy.

```bash
gpg --armor --export-secret-keys    "$KEYFPR" > master-secret.asc
gpg --armor --export-secret-subkeys "$KEYFPR" > subkeys.asc
gpg --armor --export                "$KEYFPR" > public.asc
gpg --output revoke.asc --gen-revoke "$KEYFPR"

# Encrypt the backups symmetrically before they leave the airgap, and/or use paperkey.
gpg -c master-secret.asc     # -> master-secret.asc.gpg (strong passphrase!)
```

Store `*.asc.gpg` + `revoke.asc` on encrypted offline media. Keep `public.asc` handy — it's
what you distribute.

---

## 2. Provision each Yubikey

Repeat per token. Because `keytocard` destroys the local copy, **start each token from a
fresh import of the backup.**

```bash
export GNUPGHOME=$(mktemp -d); chmod 700 "$GNUPGHOME"
gpg --import master-secret.asc            # from your decrypted backup
export KEYFPR=<your-fingerprint>

# (optional but recommended) set card PINs / metadata first
gpg --card-edit
#   admin
#   passwd        # change default user PIN (123456) and admin PIN (12345678)
#   quit

gpg --edit-key "$KEYFPR"
#   key N         # select the ENCRYPTION subkey (the [E] one in the listing)
#   keytocard     # choose slot (2) Encryption
#   save
```

To also use one flagship SSH identity on-card, repeat `key N` / `keytocard` for the `[A]`
subkey into the Authentication slot. Then discard this `GNUPGHOME` and start clean for the
next token.

Verify: `gpg --card-status` should show the Encryption key fingerprint populated.

---

## 3. Everyday keyring (per device)

On each machine you only need the **public** key plus, for decryption, the Yubikey:

```bash
gpg --import public.asc
# Trust it (ultimately, since it's yours)
echo "$KEYFPR:6:" | gpg --import-ownertrust
```

For "bare" (no-token) use on a trusted machine, import the secret subkeys instead:

```bash
gpg --import subkeys.asc      # decrypt without a Yubikey; protect with the key passphrase
```

---

## 4. Encrypting & decrypting files

```bash
# Encrypt to yourself
gpg --encrypt --recipient "$KEYFPR" secrets.txt          # -> secrets.txt.gpg
gpg -e -r "$KEYFPR" -a notes.txt                         # ASCII-armored -> notes.txt.asc

# Decrypt (Yubikey prompts for PIN/touch as configured)
gpg --decrypt secrets.txt.gpg > secrets.txt
gpg -d secrets.txt.gpg | less                            # straight to a pager, no plaintext file
```

Keep the `*.gpg` files on the SD card / synced folder. Decrypt to a pipe or a tmpfs path,
never onto removable media.

### Using `pass` for passwords / short secrets

`pass` adds no crypto — it's a naming convention + ergonomics (clipboard auto-clear, TOTP,
git history, multi-recipient) over per-file `gpg`. Ideal for many small secrets and the
Android app.

```bash
pass init "$KEYFPR"                # binds the store to your key (writes .gpg-id)
pass git init                      # optional: version + sync the ciphertext

pass insert web/github.com         # prompts, stores encrypted
pass generate web/example.com 30   # make + store a 30-char password
pass -c web/github.com             # copy to clipboard, auto-clears after 45s
pass show email/fastmail           # print (first line = password, rest = metadata)
pass otp totp/some-service         # TOTP, if seed stored (pass-otp)
```

---

## 5. SSH keys into ssh-agent

Decrypt straight into the agent so the private key only ever lives in agent memory —
never on disk. `ssh-add -` reads a key from stdin. No per-key Yubikey slot involved, so
**any number** of legacy RSA keys is fine.

```bash
gpg -d ~/keys/legacy-rsa-a.gpg | ssh-add -t 3600 -      # expires after 1h
gpg -d ~/keys/legacy-rsa-b.gpg | ssh-add -t 3600 -
pass show ssh/hetzner          | ssh-add -              # if stored in pass
```

**Store the SSH keys without their own passphrase** — the GPG/Yubikey layer is your
protection. A passphrase-protected key piped on stdin can't prompt on the terminal and will
fail unless you wire up `SSH_ASKPASS` + `SSH_ASKPASS_REQUIRE=force`.

> On-device alternative (desktop only, non-extractable, touch-per-use): import RSA keys into
> **PIV** slots (9a/9c/9d/9e + retired 82–95, ~24 total) via `yubico-piv-tool -a import-key`
> and use them over PKCS#11 (`ssh -I .../libykcs11.so`). This has **no Android/Termux SSH
> story**, so it splits your workflow — only worth it if on-device non-extractability
> outranks cross-platform uniformity.

---

## 6. Desktop / WSL2 setup

Let **Windows** own the Yubikey and relay its `gpg-agent` socket into WSL2 — the token stays
usable by Windows apps, survives reboots, and needs no custom kernel. This recipe uses
`wsl2-ssh-pageant`.

### Windows side (once)

1. Install **Gpg4win**; confirm the Yubikey is seen:
   ```
   gpg --card-status
   ```
2. Download `wsl2-ssh-pageant.exe` from its
   [releases page](https://github.com/BlackReloaded/wsl2-ssh-pageant/releases/latest) and put
   it on the **Windows** filesystem, e.g. `C:\Users\Public\Downloads\`.
3. Restart the agent so the socket is fresh:
   ```
   gpg-connect-agent killagent /bye
   gpg-connect-agent /bye
   ```

### WSL2 side (once)

```bash
sudo apt install socat iproute2

# Symlink the Windows exe into WSL. Keep the .exe ON the Windows fs — running it from the
# Linux fs makes every operation take 15-25s.
windows_destination="/mnt/c/Users/Public/Downloads/wsl2-ssh-pageant.exe"
linux_destination="$HOME/.ssh/wsl2-ssh-pageant.exe"
chmod +x "$windows_destination"
ln -s "$windows_destination" "$linux_destination"
```

Add to `~/.bashrc` (or `~/.zshrc`):

```bash
export GPG_AGENT_SOCK="$HOME/.gnupg/S.gpg-agent"
if ! ss -a | grep -q "$GPG_AGENT_SOCK"; then
  rm -rf "$GPG_AGENT_SOCK"
  wsl2_ssh_pageant_bin="$HOME/.ssh/wsl2-ssh-pageant.exe"
  if test -x "$wsl2_ssh_pageant_bin"; then
    (setsid nohup socat UNIX-LISTEN:"$GPG_AGENT_SOCK,fork" \
      EXEC:"$wsl2_ssh_pageant_bin --gpg S.gpg-agent" >/dev/null 2>&1 &)
  else
    echo >&2 "WARNING: $wsl2_ssh_pageant_bin is not executable."
  fi
  unset wsl2_ssh_pageant_bin
fi
```

Open a fresh shell, then teach the WSL `gpg` about your key + card so it routes private
operations to the Windows agent:

```bash
gpg --import public.asc
gpg --card-status                              # populates shadowed secret-key stubs via the relay
echo test | gpg -e -r "$KEYFPR" | gpg -d       # round-trip check (Yubikey prompts)
```

The `ssh-add -` workflow uses a normal **local** WSL ssh-agent (the relay only covers `gpg`):

```bash
eval "$(ssh-agent -s)"
gpg -d ~/keys/legacy-rsa-a.gpg | ssh-add -t 3600 -
```

Alternative: **[usbipd-win](https://github.com/dorssel/usbipd-win)** `attach` passes the raw
USB device into the WSL2 VM (run `pcscd` inside Linux). Downside: it's **exclusive** —
Windows loses the token while attached, and grab-order conflicts can require a reboot. Prefer
the relay unless you specifically want the token isolated in the VM.

---

## 7. Android / Termux setup

Termux (non-root) can't reach the Yubikey's USB CCID, and NFC is foreground-app only — so
**OpenKeychain drives the token** and Termux delegates to it through the **OkcAgent** bridge
(a separate app from OpenKeychain).

1. Install three apps (F-Droid / GitHub releases / Play): **OpenKeychain**, **Termux**, and
   **OkcAgent**.
2. In **OpenKeychain**: import `public.asc`, then pair the Yubikey (NFC tap or USB-C). It
   prompts for a tap whenever a private-key op is needed. (The private E key never touches
   the phone — it stays on the Yubikey.)
3. In **OkcAgent**: open it once and select the key to use for crypto operations.
4. In **Termux**, install the bridge and clients:
   ```bash
   pkg install okc-agents gnupg openssh git
   ```
5. Decrypt files through the bridge (tap the Yubikey when prompted):
   ```bash
   okc-gpg -d ~/vault/secrets.txt.gpg > "$TMPDIR/secrets.txt"
   ```
   Keep `$TMPDIR` on **internal app storage**, not the SD card, so plaintext never lands on
   removable media. The `.gpg` files themselves live on the SD card as planned.
6. Load a decrypted SSH key into a normal Termux ssh-agent:
   ```bash
   eval "$(ssh-agent -s)"
   okc-gpg -d ~/vault/id_ed25519.gpg | ssh-add -t 3600 -
   ```
   (`okc-gpg` implements a limited subset of gpg options — enough for `-d` / `-e`; see its
   `GpgArguments.kt` for the full list.) If instead you want to use the Yubikey's *own*
   Authentication subkey as an SSH identity (the single-slot flagship approach), run
   `eval "$(okc-ssh-agent)"` and skip the key file entirely.
7. For passwords, the **Android Password Store** app reads the same `pass` store and talks to
   OpenKeychain + Yubikey directly.

---

## Suggested rollout order

1. Do the offline **key ceremony** (§1) and verify the encrypted backup restores.
2. **Provision one Yubikey** (§2).
3. Get the **desktop/WSL2 relay** working (§6) and validate `gpg -e` / `gpg -d` round-trips
   plus `ssh-add -`.
4. Only then tackle the **Android/okc-agent** leg (§7) — it's the fiddliest.
5. Provision the **second/third Yubikey** from backup once the workflow is proven.

---

## Sources

- [OkcAgent — OpenKeychain ↔ Termux bridge (`okc-gpg` / `okc-ssh-agent`)](https://github.com/DDoSolitary/OkcAgent)
- [drduh YubiKey-Guide](https://github.com/drduh/YubiKey-Guide) · [multi-Yubikey note](https://github.com/drduh/YubiKey-Guide/issues/19)
- [Yubico: Using Your YubiKey with OpenPGP](https://support.yubico.com/hc/en-us/articles/360013790259-Using-Your-YubiKey-with-OpenPGP) · [Yubico PGP developer docs](https://developers.yubico.com/PGP/)
- [wsl2-ssh-pageant (GPG/SSH agent relay into WSL2)](https://github.com/BlackReloaded/wsl2-ssh-pageant) · [WSL2 gpg-agent relay gist](https://gist.github.com/Speedy37/1833b6b8c73a7768a266f0b903ab3678) · [Ultimate guide to Yubikey on WSL2](https://jardazivny.medium.com/the-ultimate-guide-to-yubikey-on-wsl2-part-1-dce2ff8d7e45)
- [usbipd-win](https://github.com/dorssel/usbipd-win)
- [Android Password Store](https://github.com/android-password-store/Android-Password-Store) · [passwordstore.org](https://www.passwordstore.org/)
- [OpenKeychain](https://www.openkeychain.org/)
