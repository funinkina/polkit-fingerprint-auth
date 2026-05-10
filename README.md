# Fingerprint Authentication for `sudo` and Polkit

This PAM configuration enables fingerprint authentication for:

- `sudo` in the terminal
- Polkit GUI authentication prompts, such as apps asking for administrator privileges
- KDE Plasma/Gnome lock screen, if fingerprint unlock is already configured

It also avoids slowing down SDDM password login by skipping fingerprint authentication for display-manager login sessions.

> [!NOTE]
> This configuration is intended for **Arch Linux**

It assumes the Arch-style PAM layout with:

```text
/etc/pam.d/system-auth
/etc/pam.d/sudo
/usr/lib/pam.d/polkit-1
```

On Arch, both `sudo` and Polkit normally include `system-auth`, so the main change is made in one central file.

## Requirements

Install and configure fingerprint support first:

```bash
sudo pacman -S fprintd
```

Enroll your fingerprint using KDE Plasma System Settings or with:

```bash
fprintd-enroll
```

Confirm fingerprint auth works before editing PAM.

## File to Edit

Edit:

```text
/etc/pam.d/system-auth
```

Do **not** edit:

```text
/etc/pam.d/sudo
/usr/lib/pam.d/polkit-1
/etc/pam.d/sddm
```

## PAM Change

Find the authentication section in `/etc/pam.d/system-auth`.

Replace this block:

```pam
-auth      [success=2 default=ignore]  pam_systemd_home.so
auth       [success=1 default=bad]     pam_unix.so          try_first_pass nullok
auth       [default=die]               pam_faillock.so      authfail
```

with:

```pam
-auth      [success=5 default=ignore]  pam_systemd_home.so
# Only offer fingerprint for sudo and Polkit prompts; skip display-manager logins.
auth       [success=2 default=ignore]  pam_succeed_if.so    service notin sudo:polkit-1
# Disallow fingerprint in sudo without tty.
auth       [success=1 default=ignore]  pam_succeed_if.so    service in sudo tty in :unknown
auth       [success=2 default=ignore]  pam_fprintd.so
auth       [success=1 default=bad]     pam_unix.so          try_first_pass nullok
auth       [default=die]               pam_faillock.so      authfail
```

> [!NOTE]
> The final file is present in the repo for reference

## How It Works

Arch’s `/etc/pam.d/sudo` includes `system-auth`:

```pam
auth include system-auth
```

Polkit also includes `system-auth` through:

```text
/usr/lib/pam.d/polkit-1
```

Because both services use the shared PAM stack, adding `pam_fprintd.so` to `system-auth` enables fingerprint authentication for both terminal and graphical privilege prompts.

The important part is this line:

```pam
auth [success=2 default=ignore] pam_succeed_if.so service notin sudo:polkit-1
```

It skips fingerprint authentication unless the PAM service is either:

```text
sudo
polkit-1
```

This prevents SDDM and other login services from waiting on fingerprint authentication during normal password login.

The next line:

```pam
auth [success=1 default=ignore] pam_succeed_if.so service in sudo tty in :unknown
```

prevents fingerprint authentication for unsafe or non-interactive `sudo` contexts without a real TTY.

## Expected Behavior

After applying the change:

- `sudo` should offer fingerprint authentication in the terminal.
- GUI admin prompts should offer fingerprint authentication through Polkit.
- SDDM password login should remain fast.
- Password fallback should still work if fingerprint authentication fails or is skipped.

## Safety Notes

Keep a root shell open while editing PAM files.

A mistake in PAM configuration can lock you out of authentication. Test in a second terminal before closing your root session:

```bash
sudo -k
sudo true
```

If something breaks, revert the change from the still-open root shell.
