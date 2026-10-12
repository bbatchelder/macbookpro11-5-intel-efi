# Sources and licenses

The helper source is GPL-2.0-only. It uses GNU-EFI headers, startup and libraries;
notices for the linked GNU-EFI components are in `LICENSES/`.

- Indexed gmux transactions and selector values follow the Linux kernel's
  [apple-gmux.c](https://github.com/torvalds/linux/blob/master/drivers/platform/x86/apple-gmux.c),
  including work by Seth Forshee / Canonical, Andreas Heider, Lukas Wunner,
  and Orlando Chamberlain.
- Apple Set OS protocol and identity strings follow Bruno Bierbaumer's
  [apple_set_os.efi](https://github.com/0xbb/apple_set_os.efi) (MIT).
- Haswell Port A/E allocation uses `DDI_A_4_LANES` from
  [intel_display_regs.h](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_display_regs.h).
- Background: [Linux vga-switcheroo documentation](https://docs.kernel.org/gpu/vga-switcheroo.html).

Built and tested with GNU-EFI 4.0.4, GCC and GNU binutils on Omarchy.
This project was prepared with GPT-6 via Codex; observations were verified
on Bryan Batchelder's MacBookPro11,5. It is not an official Omarchy component.
