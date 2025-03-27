# cryptboot

Encrypted boot partition manager with UEFI Secure Boot support

## Description

With encrypted boot partition, nobody can see or modify your kernel image or initramfs.
systemd-boot supports booting from encrypted boot partition, but you would be
still vulnerable to [Evil Maid](https://www.schneier.com/blog/archives/2009/10/evil_maid_attac.html)
attacks.

One possible solution is to use UEFI Secure Boot. Get rid of preloaded Secure Boot keys
(you really don't want to trust Microsoft and OEM), enroll your own Secure Boot keys
and sign boot loader with them. Evil maid would be unable to boot modified
boot loader (not signed by your keys) and whole attack is prevented.

`cryptboot` simply makes this easy and manageable.

## Requirements

There might be other packages that are needed for scripts and configs to function.

### Arch repo

- [efibootmgr](https://archlinux.org/packages/core/x86_64/efibootmgr/)
- [efitools](https://archlinux.org/packages/extra/x86_64/efitools/)
- [openssl](https://archlinux.org/packages/core/x86_64/openssl/)
- [sbsigntools](https://archlinux.org/packages/extra/x86_64/sbsigntools/)
- [systemd](https://archlinux.org/packages/core/x86_64/systemd/)

## Installation

0. Before you enroll your own keys, you can backup the ones which are currently deployed
    ```sh
    # Execute as root
    efi-readvar -v PK -o old_PK.esl
    efi-readvar -v KEK -o old_KEK.esl
    efi-readvar -v db -o old_db.esl
    efi-readvar -v dbx -o old_dbx.esl
    ```

1.  Install your favorite Linux distribution according to its documentation.

2.  Boot into UEFI firmware setup utility (frequently but incorrectly referred to as "BIOS"),
    enable _Secure Boot_ and clear all preloaded Secure Boot keys (Microsoft and OEM).
    By clearing all Secure Boot keys, you will enter into _Setup Mode_
    (so you can enroll your own Secure Boot keys later).

    You must also set your UEFI firmware _supervisor password_, so nobody
    can simply boot into UEFI setup utility and turn off Secure Boot.

3.  Generate your new UEFI Secure Boot keys:
    ```sh
    # Execute as root
    cryptboot-efikeys create
    ```

4.  Enroll your newly generated UEFI Secure Boot keys into UEFI firmware:
    ```sh
    # Execute as root
    cryptboot-efikeys enroll
    ```

5.  Sign boot loader with your new UEFI Secure Boot keys:
    ```sh
    # Execute as root
    cryptboot systemd-boot-sign
    ```

6.  Reboot your system, you should be completely secured against evil maid attacks from now on!

## Help

### cryptboot

```
Usage: cryptboot {systemd-boot-sign}

Manage UEFI Secure Boot keys

Commands:
    systemd-boot-sign  Sign kernel with UEFI secure boot keys
```

### cryptboot-efikeys

```
Usage: cryptboot-efikeys {create,enroll,sign,verify,list} [file-to-sign-or-verify]

Manage UEFI Secure Boot keys

Commands:
    create  Generate new UEFI Secure Boot keys
    enroll  Enroll new UEFI Secure Boot keys to your UEFI firmware
            (you have to clear old keys in your UEFI firmware setup utility first)
    sign    Sign EFI boot image file with your UEFI Secure Boot keys
    verify  Verify signature of EFI boot image file with your UEFI Secure Boot keys
    list    List all UEFI Secure Boot keys enrolled in your UEFI firmware
    status  Check if UEFI Secure Boot is active or inactive
```

### Default configuration (`/etc/cryptboot.conf`)

```conf
# EFI System partition mount point (has to be specified in /etc/fstab)
EFI_DIR="/efi"

# List of paths with images to sign
## format: ("a" "b")
TO_SIGN=("EFI/Linux" "EFI/systemd" "EFI/BOOT")

# UEFI Secure Boot keys directory
EFI_KEYS_DIR="/etc/secureboot/keys"

# Option ROM
## See: https://github.com/Foxboron/sbctl/wiki/FAQ#option-rom
##      Setting ENABLE_OPROM="false" might soft brick your device. Make sure that your hardware doesn't need oproms.
ENABLE_OPROM="true"
```

## Limitations

- If there is backdoor in your UEFI firmware, you are out of luck. It is _GAME OVER_.

  Old laptops unfortunately regularly had backdoors in BIOS:

  [BIOS Password Backdoors in Laptops](https://dogber1.blogspot.cz/2009/05/table-of-reverse-engineered-bios.html)

  New laptops (as of 2016) should be hopefully more secure, but I am only sure about
  Lenovo ThinkPads (there are no known backdoor passwords and Lenovo is reportedly
  replacing whole system board if user forgets his supervisor password).

- You should never use same UEFI firmware _supervisor password_ as your encryption password,
  because on some old laptops, supervisor password could be recovered as plaintext
  from EEPROM chip.

  New Lenovo ThinkPads (T440, T450, T540, X1 Carbon gen2/3, X240, X250, W540, W541, W550
  and newer models) should be safe, see e.g. this presentation:

  [ThinkPad BIOS Password Design for UEFI](http://monitor.espec.ws/files/lewnovo_password_399.pdf)

- Attacker can also directly reflash your UEFI firmware with his own modified evil firmware,
  but this can be prevented by physical means (e.g. epoxy resin ;-)).

  There are also [procedures](http://www.allservice.ro/forum/viewtopic.php?t=3044) how to reset
  supervisor password even on modern ThinkPads with SPI serial flash programmer. Again, you can
  use physical means for prevention.

- If you have encrypted boot partition, you can't easily use another TPM-based
  trusted / verified boot solution like [tpmtotp](https://github.com/mjg59/tpmtotp)
  or [anti-evil-maid](https://github.com/QubesOS/qubes-antievilmaid/tree/master/anti-evil-maid).

  This is because these solutions rely on running code from initramfs before you enter
  decryption password. But if you have encrypted boot partition, you have to enter decryption
  password before loading initramfs, so it would be already too late for these solutions to
  have any effect (evil firmware / boot loader will already have your password at that point).

  This can be fixed by implementing TPM support and `tpmtotp` or `anti-evil-maid` like
  functionality directly in the boot loader.

  The question is if this is really needed? If you don't trust UEFI firmware, why should you
  trust TPM? But nevertheless it would be nice to have double-check against evil maids.

## Further reading

- UEFI (Secure_Boot) - https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot
- How to boot Linux using UEFI with Secure Boot? - https://ubs_csse.gitlab.io/secu_os/tutorials/linux_secure_boot.html
