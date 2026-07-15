# Encrypted Secrets: Veracrypt → GPG + YubiKey (WSL2 + Android)

**Goal:** per-file OpenPGP encryption of secrets (SSH keys, passwords), decryptable on
WSL2 and Android/Termux, with the encryption key held on multiple YubiKeys *and*
available as an off-token ("bare") backup. Replaces Veracrypt.

**Trade-off vs. Veracrypt:** filenames/sizes are visible and there are no hidden
volumes. Encrypt a tarball where filename metadata matters.

**Order of attack:**

1. Generate the offline key and back it up.
2. Provision YubiKey #1.
3. Prove the Android path end-to-end on the real device (the fragile link).
4. Set up the WSL2 relay.
5. Provision remaining YubiKeys, migrate data, retire Veracrypt.

---

## 1. Key generation (offline machine, throwaway GNUPGHOME)

RSA 4096 throughout — the most-tested path through OpenKeychain (maintenance mode).

```bash
export GNUPGHOME=$(mktemp -d); chmod 700 "$GNUPGHOME"

# Certify-only primary, then one subkey per capability
gpg --quick-generate-key 'David <david@bozemanpass.com>' rsa4096 cert never
FPR=$(gpg --list-options show-only-fpr-mbox --list-secret-keys | awk '{print $1}')
gpg --quick-add-key "$FPR" rsa4096 encrypt 5y
gpg --quick-add-key "$FPR" rsa4096 sign    5y
gpg --quick-add-key "$FPR" rsa4096 auth    5y
```

Back up **before** touching any YubiKey — this backup is also your bare-use key:

```bash
gpg --export-secret-keys --armor "$FPR" > master-secret.asc   # full secret key
gpg --export --armor "$FPR"             > public.asc
gpg --gen-revoke "$FPR"                 > revoke.asc
tar czf gnupghome-backup.tgz -C "$GNUPGHOME" .
# → all four to offline media (2 copies), NOT to the encrypted store itself
```

## 2. Provision each YubiKey

`keytocard` + `save` **moves** the subkey (local copy becomes a card stub), so:
provision card, **restore backup**, provision next card. Each card holds identical
subkeys but has its own PINs.

```bash
gpg --card-edit         # admin → passwd → set PIN + Admin PIN (defaults 123456/12345678)

gpg --edit-key "$FPR"
  key 1                 # select encryption subkey
  keytocard             # → slot 2 (Encryption)
  key 1                 # deselect
  key 2
  keytocard             # → slot 1 (Signature)
  key 2
  key 3
  keytocard             # → slot 3 (Authentication)
  save

# Next card: restore, then repeat the block above
rm -rf "$GNUPGHOME"/*; tar xzf gnupghome-backup.tgz -C "$GNUPGHOME"
```

On each daily-use machine: import `public.asc`, then `gpg --card-status` (creates
stubs), then `gpg --edit-key "$FPR"` → `trust` → 5.

When swapping between YubiKeys on GnuPG 2.2 ("insert card with serial X"):

```bash
gpg-connect-agent "scd serialno" "learn --force" /bye
```

## 3. Existing RSA SSH key → the Authentication slot

An RSA SSH key can be re-wrapped as a GPG auth subkey — **same key material, same
public key**, so `authorized_keys` entries keep working. Caveat: the OpenPGP applet
has **one** Auth slot, so exactly one legacy key gets this treatment per card.
(Extra legacy keys: PIV retired slots on desktop, or the encrypted store on Android.)

```bash
# needs: monkeysphere (for pem2openpgp)
cp ~/.ssh/id_rsa /tmp/key && ssh-keygen -p -m PEM -f /tmp/key   # → PEM format
pem2openpgp 'ssh import' < /tmp/key | gpg --import
gpg --with-keygrip -K                                            # note new key's keygrip

gpg --expert --edit-key "$FPR"
  addkey                # → (13/14) Existing key → paste keygrip
                        # toggle capabilities: Authenticate ONLY (disable S and E)
  save

gpg --export-ssh-key "$FPR" # MUST match ~/.ssh/id_rsa.pub before deleting anything
# then keytocard the new subkey into the Auth slot (per section 2), on every card,
# and shred /tmp/key + the imported orphan primary
```

Imported (non-generated) keys occasionally misbehave on the applet — test SSH
end-to-end, including via OpenKeychain, before trusting it.

## 4. Day-to-day: encrypting and decrypting

```bash
gpg -e -r "$FPR" secret.txt                      # → secret.txt.gpg
gpg -d secret.txt.gpg                            # touch YubiKey / enter PIN
tar czf - somedir | gpg -e -r "$FPR" > somedir.tgz.gpg   # hides filenames
```

Avoid `--throw-keyids` / hidden recipients — OpenKeychain can't route those to the
token.

## 5. Android: Termux + OpenKeychain + OkcAgent

Install: **Termux from F-Droid** (not Play Store), **OpenKeychain** (F-Droid),
then in Termux:

```bash
pkg install okc-agents openssh gnupg
termux-setup-storage        # grants ~/storage/* incl. the SD card
```

OpenKeychain: import `public.asc`, then *Manage my keys → security token* with the
YubiKey on NFC/USB-C so it binds the key to the card.

```bash
okc-gpg -d ~/storage/external-1/secrets/foo.gpg   # OpenKeychain prompts for token
export SSH_AUTH_SOCK=$HOME/.okc-ssh-agent.sock
okc-ssh-agent "$SSH_AUTH_SOCK" &                  # SSH via the card's Auth subkey
ssh host
```

Notes:

- Decrypt into Termux's private `$HOME` (or pipe directly), never onto shared
  storage; Android typically only lets Termux *write* to the removable SD inside
  its own `Android/data` dir — reading the encrypted blobs is unrestricted.
- Health warning: OpenKeychain (last release 6.0.4, Feb 2024) and OkcAgent are both
  in maintenance mode. **Prototype this whole section before migrating.** Fallbacks:
  bare key in Termux `gnupg` (less secure), or the Termius app (only Android SSH
  client with FIDO2 sk-key support — but standalone, no Termux integration).
- FIDO2 sk- SSH keys remain unsupported in Termux (termux-packages #4942, open
  since 2020) — that's why OpenKeychain is in the loop at all.

## 6. WSL2: relay Windows gpg-agent (recommended over usbipd)

Gpg4win on Windows owns the YubiKey; WSL2 gets its sockets. No attach/detach, and
the key works in Windows and WSL2 simultaneously.

1. Windows: install Gpg4win, import `public.asc`, check `gpg --card-status` in
   PowerShell. Sockets live under `%LOCALAPPDATA%\gnupg` (older guides say
   `%APPDATA%` — wrong for current Gpg4win).
2. Put `npiperelay.exe` somewhere on the Windows side (e.g. `C:\bin`).
3. WSL2: `apt install socat gnupg`, import `public.asc`, then relay (from a login
   script — see the Speedy37 gist for the full robust version):

```bash
socat UNIX-LISTEN:"$(gpgconf --list-dirs agent-socket),fork" \
  EXEC:'/mnt/c/bin/npiperelay.exe -ei -ep -s -a "C:/Users/<you>/AppData/Local/gnupg/S.gpg-agent"',nofork &

# SSH through the same key:
export SSH_AUTH_SOCK="$(gpgconf --list-dirs agent-ssh-socket)"
socat UNIX-LISTEN:"$SSH_AUTH_SOCK,fork" \
  EXEC:'/mnt/c/bin/npiperelay.exe -ei -s //./pipe/openssh-ssh-agent',nofork &
```

Alternative: `usbipd-win` USB passthrough + `pcscd` inside WSL2 — pure Linux but
exclusive attach and more friction.

## 7. Slot inventory (why the design looks like this)

| Applet  | Capacity                    | Reachable from                    |
|---------|-----------------------------|-----------------------------------|
| OpenPGP | 3 slots: Sig / Enc / Auth   | Linux, WSL2, **Android (OpenKeychain)** |
| PIV     | 4 + ~20 retired slots       | Linux/WSL2 only (PKCS#11 ssh)     |
| FIDO2   | ~100 resident, ∞ non-resident | Desktop OpenSSH; Android only via Termius |

## Sources

- [drduh YubiKey guide](https://github.com/drduh/YubiKey-Guide) — canonical offline-key + keytocard ceremony
- [Yubico: importing keys to the OpenPGP applet](https://developers.yubico.com/PGP/Importing_keys.html)
- [SSH key → GPG auth subkey walkthrough (artizirk gist)](https://gist.github.com/artizirk/d09ce3570021b0f65469cb450bee5e29) and [opensource.com version](https://opensource.com/article/19/4/gpg-subkeys-ssh-multiples)
- [OkcAgent](https://github.com/DDoSolitary/OkcAgent) — Termux ↔ OpenKeychain bridge
- [OpenKeychain maintenance-mode discussion](https://github.com/open-keychain/open-keychain/discussions/2970)
- [Termux FIDO2 gap: termux-packages #4942](https://github.com/termux/termux-packages/issues/4942)
- [WSL2 gpg-agent relay script (Speedy37 gist)](https://gist.github.com/Speedy37/1833b6b8c73a7768a266f0b903ab3678) and [codingnest guide](https://codingnest.com/how-to-use-gpg-with-yubikey-wsl/)
- [Termius FIDO2 SSH support](https://termius.com/blog/using-fido2-with-ssh) — fallback Android client
