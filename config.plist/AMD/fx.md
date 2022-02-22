# Bulldozer(15°) e Jaguar(16°)

| Supporto | Versione |
| :--- | :--- |
| Supporto di macOS iniziale | macOS 10.13, High Sierra |

## Punto d'Inizio

Fare un config.plist potrebbe sembrare difficile, ma non lo è. Ci metterai solo un po' di tempo ma questa guida ti dice come configurare il tutto, non rimarrai a bocca asciutta. Questo significa anche che se hai problemi, rivedi come hai impostato il config per essere sicuro che siano corrette. Le cose principali da definire con OpenCore:

* **Tutte le proprietà devono essere definite**, non c'è un fallback di default, perciò **non cancellare sezioni a meno che non ti sia esplicitamente richiesto**. Se la guida non parla di quella sezione, lascia come il predefinito.
* **Il Sample.plist non può essere usato così com'è**, devi configurarlo per il tuo sistema
* **NON USARE CONFIGURATORI**, questi raramente rispettano la configurazione di OpenCore e alcuni di quelli come Mackie aggiungeranno proprietà di Clover o potrebbero corrompere il plist!

Ora che hai letto questo, un piccolo reminder degli strumenti necessari

* [ProperTree](https://github.com/corpnewt/ProperTree)
  * Plist editor universale
* [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS)
  * Per geneare i dati del nostro SMBIOS
* [Sample config.plist](https://github.com/acidanthera/OpenCorePkg/releases)
  * Vedi la sezione precedente per capire come ottenerlo: [Setup del config.plist](../)
* [AMD Kernel Patches](https://github.com/AMD-OSX/AMD_Vanilla/tree/master)
  * Necessarie per avviare macOS nell'hardware AMD
  * Supporto delle generazioni: 15h, 16h, 17h and 19h

**E leggi questa guida una volta prima di impostare OpenCore e sii sicuro di aver impostato tutto correttamente. Nota che le immagini non potranno essere sempre aggiornatissime, perciò leggi le didascalie sotto, se nulla viene menzionato, lascia com'è di default.**

## ACPI

![ACPI](../../images/config/AMD/acpi-fx.png)

### Add

::: tip Informazioni

Qui aggiungerai i tuoi SSDT al sistema, sono molto importanti per **avviare macOS** e hanno molti usi come [USB maps (EN)](/OpenCore-Post-Install/usb/), [disabilitare GPU non supportate](/extras/spoof.md) e altro. E con il nostro sistema, **è soprattutto richiesto per l'avvio**. Guide per farli può essere trovata qui: **[Iniziamo con ACPI](/Getting-Started-With-ACPI/)**

Nota che **non dovresti** aggiungere `DSDT.aml` qui, è aggiunto già dal tuo firmware. Perciò se presente, toglilo dal tuo `config.plist` e da EFI/OC/ACPI.

Gli SSDT hanno l'estensione **.aml** (Assembled) e andranno dentro la cartella `EFI/OC/ACPI` e **devono** essere specificati nel config anche nella sezione `ACPI -> Add`.

| SSDT Richiesti | Descrizione |
| :--- | :--- |
| **SSDT-EC-USBX** | Sistema il controller integrato e l'energia dei USB |

:::

### Delete

Questa sezione previene il caricamento di certe tabelle ACPI, per noi è ignorabile.

### Patch

Questa sezione ci permette di modificare dinamicamente le parti di ACPI (DSDT, SSDT, ecc.) tramite OpenCore. Per noi, le nostre patch sono legate agli SSDT. Questa è una soluzione più pulita perché ci permetterà di avviare Windows e altri sistemi con OpenCore

### Quirks

Impostazioni relative a ACPI, lascia tutto come default dato che non useremo questi quirk.

## Booter

![Booter](../../images/config/config-universal/aptio-iv-booter.png)

This section is dedicated to quirks relating to boot.efi patching with OpenRuntime, the replacement for AptioMemoryFix.efi

### MmioWhitelist

This section is allowing spaces to be passthrough to macOS that are generally ignored, useful when paired with `DevirtualiseMmio`

### Quirks

::: tip Info
Settings relating to boot.efi patching and firmware fixes, for us, we leave it as default
:::
::: details More in-depth Info

* **AvoidRuntimeDefrag**: YES
  * Fixes UEFI runtime services like date, time, NVRAM, power control, etc
* **EnableSafeModeSlide**: YES
  * Enables slide variables to be used in safe mode.
* **EnableWriteUnprotector**: YES
  * Needed to remove write protection from CR0 register.
* **ProvideCustomSlide**: YES
  * Used for Slide variable calculation. However the necessity of this quirk is determined by `OCABC: Only N/256 slide values are usable!` message in the debug log. If the message `OCABC: All slides are usable! You can disable ProvideCustomSlide!` is present in your log, you can disable `ProvideCustomSlide`.
* **SetupVirtualMap**: YES
  * Fixes SetVirtualAddresses calls to virtual addresses, required for Gigabyte boards to resolve early kernel panics

:::

## DeviceProperties

![DeviceProperties](../../images/config/config-universal/DP-no-igpu.png)

### Add

Sets device properties from a map.

By default, the Sample.plist has this section set for Audio. We'll be setting Audio the layout in the Argomenti di avvio section, for us we can ignore this

### Delete

Removes device properties from the map, for us we can ignore this

## Kernel

| Kernel | Kernel Patches |
| :--- | :--- |
| ![Kernel](../../images/config/AMD/kernel.png) | ![](../../images/config/AMD/kernel-patch.png) |

### Add

Here's where we specify which kexts to load, in what specific order to load, and what architectures each kext is meant for. By default we recommend leaving what ProperTree has done, however for 32-bit CPUs please see below:

::: details More in-depth Info

The main thing you need to keep in mind is:

* Load order
  * Remember that any plugins should load *after* its dependencies
  * This means kexts like Lilu **must** come before VirtualSMC, AppleALC, WhateverGreen, etc

A reminder that [ProperTree](https://github.com/corpnewt/ProperTree) users can run **Cmd/Ctrl + Shift + R** to add all their kexts in the correct order without manually typing each kext out.

* **Arch**
  * Architectures supported by this kext
  * Currently supported values are `Any`, `i386` (32-bit), and `x86_64` (64-bit)
* **BundlePath**
  * Name of the kext
  * ex: `Lilu.kext`
* **Enabled**
  * Self-explanatory, either enables or disables the kext
* **ExecutablePath**
  * Path to the actual executable is hidden within the kext, you can see what path your kext has by right-clicking and selecting `Show Package Contents`. Generally, they'll be `Contents/MacOS/Kext` but some have kexts hidden within under `Plugin` folder. Do note that plist only kexts do not need this filled in.
  * ex: `Contents/MacOS/Lilu`
* **MinKernel**
  * Lowest kernel version your kext will be injected into, see below table for possible values
  * ex. `12.00.00` for OS X 10.8
* **MaxKernel**
  * Highest kernel version your kext will be injected into, see below table for possible values
  * ex. `11.99.99` for OS X 10.7
* **PlistPath**
  * Path to the `info.plist` hidden within the kext
  * ex: `Contents/Info.plist`

::: details Kernel Support Table

| OS X Version | MinKernel | MaxKernel |
| :--- | :--- | :--- |
| 10.4 | 8.0.0 | 8.99.99 |
| 10.5 | 9.0.0 | 9.99.99 |
| 10.6 | 10.0.0 | 10.99.99 |
| 10.7 | 11.0.0 | 11.99.99 |
| 10.8 | 12.0.0 | 12.99.99 |
| 10.9 | 13.0.0 | 13.99.99 |
| 10.10 | 14.0.0 | 14.99.99 |
| 10.11 | 15.0.0 | 15.99.99 |
| 10.12 | 16.0.0 | 16.99.99 |
| 10.13 | 17.0.0 | 17.99.99 |
| 10.14 | 18.0.0 | 18.99.99 |
| 10.15 | 19.0.0 | 19.99.99 |
| 11 | 20.0.0 | 20.99.99 |
| 12 | 21.0.0 | 21.99.99 |
| 12 | 21.0.0 | 21.99.99 |

:::

### Emulate

::: tip Info

Needed for spoofing unsupported CPUs like Pentiums and Celerons and to disable CPU power management on unsupported CPUs (such as AMD CPUs)

| Quirk | Enabled |
| :--- | :--- |
| DummyPowerManagement | YES |

:::

::: details More in-depth Info

* **CpuidMask**: Leave this blank
  * Mask for fake CPUID
* **CpuidData**: Leave this blank
  * Fake CPUID entry
* **DummyPowerManagement**: YES
  * New alternative to NullCPUPowerManagement, required for all AMD CPU based systems as there's no native power management. Intel can ignore
* **MinKernel**: Leave this blank
  * Lowest kernel version the above patches will be injected into, if no value specified it'll be applied to all versions of macOS. See below table for possible values
  * ex. `12.00.00` for OS X 10.8
* **MaxKernel**: Leave this blank
  * Highest kernel version the above patches will be injected into, if no value specified it'll be applied to all versions of macOS. See below table for possible values
  * ex. `11.99.99` for OS X 10.7

::: details Kernel Support Table

| OS X Version | MinKernel | MaxKernel |
| :--- | :--- | :--- |
| 10.4 | 8.0.0 | 8.99.99 |
| 10.5 | 9.0.0 | 9.99.99 |
| 10.6 | 10.0.0 | 10.99.99 |
| 10.7 | 11.0.0 | 11.99.99 |
| 10.8 | 12.0.0 | 12.99.99 |
| 10.9 | 13.0.0 | 13.99.99 |
| 10.10 | 14.0.0 | 14.99.99 |
| 10.11 | 15.0.0 | 15.99.99 |
| 10.12 | 16.0.0 | 16.99.99 |
| 10.13 | 17.0.0 | 17.99.99 |
| 10.14 | 18.0.0 | 18.99.99 |
| 10.15 | 19.0.0 | 19.99.99 |
| 11 | 20.0.0 | 20.99.99 |
| 12 | 21.0.0 | 21.99.99 |
| 12 | 21.0.0 | 21.99.99 |

:::

### Force

Used for loading kexts off system volume, only relevant for older operating systems where certain kexts are not present in the cache(ie. IONetworkingFamily in 10.6).

For us, we can ignore.

### Block

Blocks certain kexts from loading. Not relevant for us.

### Patch

This is where the AMD kernel patching magic happens. Please do note that `KernelToPatch` and `MatchOS` from Clover becomes `Kernel` and `MinKernel`/ `MaxKernel` in OpenCore, you can find pre-made patches by [AlGrey](https://amd-osx.com/forum/memberlist.php?mode=viewprofile&u=10918&sid=e0feb8a14a97be482d2fd68dbc268f97)(algrey#9303).

Kernel patches:

* [Bulldozer/Jaguar(15h/16h)](https://github.com/AMD-OSX/AMD_Vanilla/tree/master) (10.13, 10.14, 10.15, 11.x and 12.x)

To merge:

* Open both files,
* Delete the `Kernel -> Patch` section from config.plist
* Copy the `Kernel -> Patch` section from patches.plist
* Paste into where old patches were in config.plist

![](../../images/config/AMD/kernel.gif)

You will also need to modify three patches, all named `algrey - Force cpuid_cores_per_package`. You only need to change the `Replace` value. You should change:

* `B8000000 0000` => `B8 <core count> 0000 0000`
* `BA000000 0000` => `BA <core count> 0000 0000`
* `BA000000 0090` => `BA <core count> 0000 0090`

Where `<core count>` is replaced with the physical core count of your CPU in hexadecimal. For example, an 8-Core 5800X would have the new Replace value be:

* `B8 08 0000 0000`
* `BA 08 0000 0000`
* `BA 08 0000 0090`

::: details Core Count => Hexadecimal Table

| Core Count | Hexadecimal |
| :--------- | :---------- |
| 4 Core | `04` |
| 6 Core | `06` |
| 8 Core | `08` |
| 12 Core | `0C` |
| 16 Core | `10` |
| 24 Core | `18` |
| 32 Core | `20` |
| 64 Core | `40` |

:::

### Quirks

::: tip Info

Settings relating to the kernel, for us we'll be enabling the following:

| Quirk | Enabled | Comment |
| :--- | :--- | :--- |
| PanicNoKextDump | YES | |
| PowerTimeoutKernelPanic | YES | |
| ProvideCurrentCpuInfo | YES | |
| XhciPortLimit | YES | Disable if running macOS 11.3+ |

:::

::: details More in-depth Info

* **AppleCpuPmCfgLock**: NO
  * Only needed when CFG-Lock can't be disabled in BIOS. AMD users can ignore
* **AppleXcpmCfgLock**: NO
  * Only needed when CFG-Lock can't be disabled in BIOS. AMD users can ignore
* **AppleXcpmExtraMsrs**: NO
  * Disables multiple MSR access needed for unsupported CPUs like Pentiums and certain Xeons
* **CustomSMBIOSGuid**: NO
  * Performs GUID patching for UpdateSMBIOSMode set to `Custom`. Usually relevant for Dell laptops
  * Enabling this quirk in tandem with `PlatformInfo -> UpdateSMBIOSMode -> Custom` will disable SMBIOS injection into "non-Apple" OSes however we do not endorse this method as it breaks Bootcamp compatibility. Use at your own risk.
* **DisableIoMapper**: NO
  * AMD doesn't have DMAR or VT-D support so irrelevant
* **DisableLinkeditJettison**: YES
  * Allows Lilu and others to have more reliable performance without `keepsyms=1`
* **DisableRtcChecksum**: NO
  * Prevents AppleRTC from writing to primary checksum (0x58-0x59), required for users who either receive BIOS reset or are sent into Safe mode after reboot/shutdown
* **ExtendBTFeatureFlags** NO
  * Helpful for those having continuity issues with non-Apple/non-Fenvi cards
* **LapicKernelPanic**: NO
  * Disables kernel panic on AP core lapic interrupt, generally needed for HP systems. Clover equivalent is `Kernel LAPIC`
* **LegacyCommpage**: NO
  * Resolves SSSE3 requirement for 64 Bit CPUs in macOS, mainly relevant for 64-Bit Pentium 4 CPUs(ie. Prescott)
* **PanicNoKextDump**: YES
  * Allows for reading kernel panics logs when kernel panics occur
* **PowerTimeoutKernelPanic**: YES
  * Helps fix kernel panics relating to power changes with Apple drivers in macOS Catalina, most notably with digital audio.
* **ProvideCurrentCpuInfo**: YES
  * Provides the kernel with CPU frequency values for AMD.
* **SetApfsTrimTimeout**: `-1`
  * Sets trim timeout in microseconds for APFS filesystems on SSDs, only applicable for macOS 10.14 and newer with problematic SSDs.
* **XhciPortLimit**: YES
  * This is actually the 15 port limit patch, don't rely on it as it's not a guaranteed solution for fixing USB. A more proper solution for AMD can be found here: [AMD USB Mapping](/OpenCore-Post-Install/usb/)
:::

### Scheme

Settings related to legacy booting(ie. 10.4-10.6), for majority you can skip however for those planning to boot legacy OSes you can see below:

::: details More in-depth Info

* **FuzzyMatch**: True
  * Used for ignoring checksums with kernelcache, instead opting for the latest cache available. Can help improve boot performance on many machines in 10.6
* **KernelArch**: x86_64
  * Set the kernel's arch type, you can choose between `Auto`, `i386` (32-bit), and `x86_64` (64-bit).
  * If you're booting older OSes which require a 32-bit kernel(ie. 10.4 and 10.5) we recommend to set this to `Auto` and let macOS decide based on your SMBIOS. See below table for supported values:
    * 10.4-10.5 — `x86_64`, `i386` or `i386-user32`
      * `i386-user32` refers 32-bit userspace, so 32-bit CPUs must use this(or CPUs missing SSSE3)
      * `x86_64` will still have a 32-bit kernelspace however will ensure 64-bit userspace in 10.4/5
    * 10.6 — `i386`, `i386-user32`, or `x86_64`
    * 10.7 — `i386` or `x86_64`
    * 10.8 or newer — `x86_64`

* **KernelCache**: Auto
  * Set kernel cache type, mainly useful for debugging and so we recommend `Auto` for best support

:::

## Misc

![Misc](../../images/config/config-universal/misc.png)

### Boot

Settings for boot screen (Leave everything as default).

### Debug

::: tip Info

Helpful for debugging OpenCore boot issues(We'll be changing everything *but* `DisplayDelay`):

| Quirk | Enabled |
| :--- | :--- |
| AppleDebug | YES |
| ApplePanic | YES |
| DisableWatchDog | YES |
| Target | 67 |

:::

::: details More in-depth Info

* **AppleDebug**: YES
  * Enables boot.efi logging, useful for debugging. Note this is only supported on 10.15.4 and newer
* **ApplePanic**: YES
  * Attempts to log kernel panics to disk
* **DisableWatchDog**: YES
  * Disables the UEFI watchdog, can help with early boot issues
* **DisplayLevel**: `2147483650`
  * Shows even more debug information, requires debug version of OpenCore
* **SerialInit**: NO
  * Needed for setting up serial output with OpenCore
* **SysReport**: NO
  * Helpful for debugging such as dumping ACPI tables
  * Note that this is limited to DEBUG versions of OpenCore
* **Target**: `67`
  * Shows more debug information, requires debug version of OpenCore

These values are based of those calculated in [OpenCore debugging](/troubleshooting/debug.md)

:::

### Security

::: tip Info

Security is pretty self-explanatory, **do not skip**. We'll be changing the following:

| Quirk | Enabled | Comment |
| :--- | :--- | :--- |
| AllowNvramReset | YES | |
| AllowSetDefault | YES | |
| BlacklistAppleUpdate | YES | |
| ScanPolicy | 0 | |
| SecureBootModel | Default | Leave this as `Default` if running macOS Big Sur or newer. The next page goes into more detail about this setting. |
| Vault | Optional | This is a word, it is not optional to omit this setting. You will regret it if you don't set it to Optional, note that it is case-sensitive |

:::

::: details More in-depth Info

* **AllowNvramReset**: YES
  * Allows for NVRAM reset both in the boot picker and when pressing `Cmd+Opt+P+R`
* **AllowSetDefault**: YES
  * Allow `CTRL+Enter` and `CTRL+Index` to set default boot device in the picker
* **ApECID**: 0
  * Used for netting personalized secure-boot identifiers, currently this quirk is unreliable due to a bug in the macOS installer so we highly encourage you to leave this as default.
* **AuthRestart**: NO
  * Enables Authenticated restart for FileVault 2 so password is not required on reboot. Can be considered a security risk so optional
* **BlacklistAppleUpdate**: YES
  * Used for blocking firmware updates, used as extra level of protection as macOS Big Sur no longer uses `run-efi-updater` variable

* **DmgLoading**: Signed
  * Ensures only signed DMGs load
* **ExposeSensitiveData**: `6`
  * Shows more debug information, requires debug version of OpenCore
* **Vault**: `Optional`
  * We won't be dealing vaulting so we can ignore, **you won't boot with this set to Secure**
  * This is a word, it is not optional to omit this setting. You will regret it if you don't set it to `Optional`, note that it is case-sensitive
* **ScanPolicy**: `0`
  * `0` allows you to see all drives available, please refer to [Security](/OpenCore-Post-Install/universal/security.md) section for further details. **Will not boot USB devices with this set to default**
* **SecureBootModel**: Disabled
  * Controls Apple's secure boot functionality in macOS, please refer to [Security](/OpenCore-Post-Install/universal/security.md) section for further details.
  * Note: Users may find upgrading OpenCore on an already installed system can result in early boot failures. To resolve this, see here: [Stuck on OCB: LoadImage failed - Security Violation](/troubleshooting/extended/kernel-issues.md#stuck-on-ocb-loadimage-failed-security-violation)

:::

### Tools

Used for running OC debugging tools like the shell, ProperTree's snapshot function will add these for you.

### Entries

Used for specifying irregular boot paths that can't be found naturally with OpenCore.

Won't be covered here, see 8.6 of [Configuration.pdf](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/Configuration.pdf) for more info

## NVRAM

![NVRAM](../../images/config/config-universal/nvram.png)

### Add

::: tip 4D1EDE05-38C7-4A6A-9CC6-4BCCA8B38C14

Used for OpenCore's UI scaling, default will work for us. See in-depth section for more info

:::

::: details More in-depth Info

Booter Path, mainly used for UI Scaling

* **UIScale**:
  * `01`: Standard resolution
  * `02`: HiDPI (generally required for FileVault to function correctly on smaller displays)

* **DefaultBackgroundColor**: Background color used by boot.efi
  * `00000000`: Syrah Black
  * `BFBFBF00`: Light Gray

:::

::: tip 4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102

OpenCore's NVRAM GUID, mainly relevant for RTCMemoryFixup users

:::

::: details More in-depth Info

* **rtc-blacklist**: <>
  * To be used in conjunction with RTCMemoryFixup, see here for more info: [Fixing RTC write issues](/OpenCore-Post-Install/misc/rtc.html#finding-our-bad-rtc-region)
  * Most users can ignore this section

:::

::: tip 7C436110-AB2A-4BBB-A880-FE41995C9F82

System Integrity Protection bitmask

* **General Purpose Argomenti di avvio**:

| Argomenti di avvio | Description |
| :--- | :--- |
| **-v** | This enables verbose mode, which shows all the behind-the-scenes text that scrolls by as you're booting instead of the Apple logo and progress bar. It's invaluable to any Hackintosher, as it gives you an inside look at the boot process, and can help you identify issues, problem kexts, etc. |
| **debug=0x100** | This disables macOS's watchdog which helps prevents a reboot on a kernel panic. That way you can *hopefully* glean some useful info and follow the breadcrumbs to get past the issues. |
| **keepsyms=1** | This is a companion setting to debug=0x100 that tells the OS to also print the symbols on a kernel panic. That can give some more helpful insight as to what's causing the panic itself. |
| **npci=0x2000** | This disables some PCI debugging related to `kIOPCIConfiguratorPFM64`, alternative is `npci=0x3000` which disables debugging related to `gIOPCITunnelledKey` in addition. Required for when getting stuck on `[PCI configuration begin]` as there are IRQ conflicts relating to your PCI lanes. **Not needed if Above 4G Decoding is enabled**. [Source](https://opensource.apple.com/source/IOPCIFamily/IOPCIFamily-370.0.2/IOPCIBridge.cpp.auto.html) |

* **GPU-Specific Argomenti di avvio**:

| Argomenti di avvio | Description |
| :--- | :--- |
| **agdpmod=pikera** | Used for disabling board ID checks on Navi GPUs (RX 5000 & 6000 series), without this you'll get a black screen. **Don't use if you don't have Navi** (ie. Polaris and Vega cards shouldn't use this) |
| **nvda_drv_vrl=1** | Used for enabling Nvidia's Web Drivers on Maxwell and Pascal cards in Sierra and High Sierra |

* **csr-active-config**: `00000000`
  * Settings for 'System Integrity Protection' (SIP). It is generally recommended to change this with `csrutil` via the recovery partition.
  * csr-active-config by default is set to `00000000` which enables System Integrity Protection. You can choose a number of different values but overall we recommend keeping this enabled for best security practices. More info can be found in our troubleshooting page: [Disabilitare SIP](/troubleshooting/extended/post-issues.md#disabilitare-sip)

* **run-efi-updater**: `No`
  * This is used to prevent Apple's firmware update packages from installing and breaking boot order; this is important as these firmware updates (meant for Macs) will not work.

* **prev-lang:kbd**: <>
  * Needed for non-latin keyboards in the format of `lang-COUNTRY:keyboard`, recommended to keep blank though you can specify it(**Default in Sample config is Russian**):
  * American: `en-US:0`(`656e2d55533a30` in HEX)
  * Full list can be found in [AppleKeyboardLayouts.txt](https://github.com/acidanthera/OpenCorePkg/blob/master/Utilities/AppleKeyboardLayouts/AppleKeyboardLayouts.txt)
  * Hint: `prev-lang:kbd` can be changed into a String so you can input `en-US:0` directly instead of converting to HEX

| Key | Type | Value |
| :--- | :--- | :--- |
| prev-lang:kbd | String | en-US:0 |

:::

### Delete

::: tip Info

Forcibly rewrites NVRAM variables, do note that `Add` **will not overwrite** values already present in NVRAM so values like `Argomenti di avvio` should be left alone. For us, we'll be changing the following:

| Quirk | Enabled |
| :--- | :--- |
| WriteFlash | YES |

:::

::: details More in-depth Info

* **LegacyEnable**: NO
  * Allows for NVRAM to be stored on nvram.plist, needed for systems without native NVRAM

* **LegacyOverwrite**: NO
  * Permits overwriting firmware variables from nvram.plist, only needed for systems without native NVRAM

* **LegacySchema**
  * Used for assigning NVRAM variables, used with LegacyEnable set to YES

* **WriteFlash**: YES
  * Enables writing to flash memory for all added variables.

:::

## PlatformInfo

![PlatformInfo](../../images/config/config-universal/iMacPro-smbios.png)

::: tip Info

For setting up the SMBIOS info, we'll use CorpNewt's [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) application.

For this example, we'll choose the MacPro7,1 SMBIOS but some SMBIOS play with certain GPUs better than others:

* MacPro7,1: AMD RX Polaris and newer
  * Note that MacPro7,1 is exclusive to macOS 10.15, Catalina and newer
* iMacPro1,1: NVIDIA Maxwell and Pascal or AMD RX Polaris and newer
  * Use if you need High Sierra or Mojave, otherwise use MacPro7,1
* iMac14,2: NVIDIA Maxwell and Pascal
  * Use if you get black screens on iMacPro1,1 after installing Web Drivers with an NVIDIA GPU
* MacPro6,1: AMD GCN GPUs

Run GenSMBIOS, pick option 1 for downloading MacSerial and Option 3 for selecting out SMBIOS.  This will give us an output similar to the following:

```sh
  #######################################################
 #               MacPro7,1 SMBIOS Info                 #
#######################################################

Type:         MacPro7,1
Serial:       F5KZV0JVP7QM
Board Serial: F5K9518024NK3F7JC
SmUUID:       535B897C-55F7-4D65-A8F4-40F4B96ED394
Apple ROM:    001D4F0D5E22
```

The order is `Product | Serial | Board Serial (MLB)`

The `Type` part gets copied to Generic -> SystemProductName.

The `Serial` part gets copied to Generic -> SystemSerialNumber.

The `Board Serial` part gets copied to Generic -> MLB.

The `SmUUID` part gets copied to Generic -> SystemUUID.

The `Apple ROM` part gets copied to Generic -> ROM.

::: danger
Reminder that you want either an invalid serial or valid serial numbers but those not in use, you want to get a message back like: "Invalid Serial" or "Purchase Date not Validated"
:::

[Apple Check Coverage page](https://checkcoverage.apple.com)

**Automatic**: YES

* Generates PlatformInfo based on Generic section instead of DataHub, NVRAM, and SMBIOS sections

:::

### Generic

::: details More in-depth Info

* **AdviseFeatures**: NO
  * Used for when the EFI partition isn't first on the Windows drive

* **MaxBIOSVersion**: NO
  * Sets BIOS version to Max to avoid firmware updates in Big Sur+, mainly applicable for genuine Macs.

* **ProcessorType**: `0`
  * Set to `0` for automatic type detection, however this value can be overridden if desired. See [AppleSmBios.h](https://github.com/acidanthera/OpenCorePkg/blob/master/Include/Apple/IndustryStandard/AppleSmBios.h) for possible values

* **SpoofVendor**: YES
  * Swaps vendor field for Acidanthera, generally not safe to use Apple as a vendor in most case

* **SystemMemoryStatus**: Auto
  * Sets whether memory is soldered or not in SMBIOS info, purely cosmetic and so we recommend `Auto`

* **UpdateDataHub**: YES
  * Update Data Hub fields

* **UpdateNVRAM**: YES
  * Update NVRAM fields

* **UpdateSMBIOS**: YES
  * Updates SMBIOS fields

* **UpdateSMBIOSMode**: Create
  * Replace the tables with newly allocated EfiReservedMemoryType, use `Custom` on Dell laptops requiring `CustomSMBIOSGuid` quirk
  * Setting to `Custom` with `CustomSMBIOSGuid` quirk enabled can also disable SMBIOS injection into "non-Apple" OSes however we do not endorse this method as it breaks Bootcamp compatibility. Use at your own risk

:::

## UEFI

![UEFI](../../images/config/config-universal/aptio-v-uefi.png)

**ConnectDrivers**: YES

* Forces .efi drivers, change to NO will automatically connect added UEFI drivers. This can make booting slightly faster, but not all drivers connect themselves. E.g. certain file system drivers may not load.

### Drivers

Add your .efi drivers here.

Only drivers present here should be:

* HfsPlus.efi
* OpenRuntime.efi

### APFS

By default, OpenCore only loads APFS drivers from macOS Big Sur and newer. If you are booting macOS Catalina or earlier, you may need to set a new minimum version/date.
Not setting this can result in OpenCore not finding your macOS partition!

macOS Sierra and earlier use HFS instead of APFS. You can skip this section if booting older versions of macOS.

::: tip APFS Versions

Both MinVersion and MinDate need to be set if changing the minimum version.

| macOS Version | Min Version | Min Date |
| :------------ | :---------- | :------- |
| High Sierra (`10.13.6`) | `748077008000000` | `20180621` |
| Mojave (`10.14.6`) | `945275007000000` | `20190820` |
| Catalina (`10.15.4`) | `1412101001000000` | `20200306` |
| No restriction | `-1` | `-1` |

:::

### Audio

Related to AudioDxe settings, for us we'll be ignoring(leave as default). This is unrelated to audio support in macOS.

* For further use of AudioDxe and the Audio section, please see the Post Install page: [Add GUI and Boot-chime](/OpenCore-Post-Install/)

### Input

Related to boot.efi keyboard passthrough used for FileVault and Hotkey support, leave everything here as default as we have no use for these quirks. See here for more details: [Security and FileVault](/OpenCore-Post-Install/)

### Output

Relating to OpenCore's visual output,  leave everything here as default as we have no use for these quirks.

### ProtocolOverrides

Mainly relevant for Virtual machines, legacy macs and FileVault users. See here for more details: [Security and FileVault](/OpenCore-Post-Install/)

### Quirks

::: tip Info
Relating to quirks with the UEFI environment, for us we'll be changing the following:

| Quirk | Enabled | Comment |
| :--- | :--- | :--- |
| UnblockFsConnect | NO | Needed mainly by HP motherboards |

:::

::: details More in-depth Info

* **DisableSecurityPolicy**: NO
  * Disables platform security policy in firmware, recommended for buggy firmwares where disabling Secure Boot does not allow 3rd party firmware drivers to load.
  * If running a Microsoft Surface device, recommended to enable this option

* **RequestBootVarRouting**: YES
  * Redirects AptioMemoryFix from `EFI_GLOBAL_VARIABLE_GUID` to `OC_VENDOR_VARIABLE_GUID`. Needed for when firmware tries to delete boot entries and is recommended to be enabled on all systems for correct update installation, Startup Disk control panel functioning, etc.

* **UnblockFsConnect**: NO
  * Some firmware block partition handles by opening them in By Driver mode, which results in File System protocols being unable to install. Mainly relevant for HP systems when no drives are listed

:::

### ReservedMemory

Used for exempting certain memory regions from OSes to use, mainly relevant for Sandy Bridge iGPUs or systems with faulty memory. Use of this quirk is not covered in this guide

## AMD BIOS Settings

* Note: Most of these options may not be present in your firmware, we recommend matching up as closely as possible but don't be too concerned if many of these options are not available in your BIOS

### Disable

* Fast Boot
* Secure Boot
* Serial/COM Port
* Parallel Port
* Compatibility Support Module (CSM)(**Must be off, GPU errors like `gIO` are common when this option in enabled**)

### Enable

* Above 4G decoding(**This must be on, if you can't find the option then add `npci=0x2000` to Argomenti di avvio. Do not have both this option and npci enabled at the same time**)
* EHCI/XHCI Hand-off
* OS type: Windows 8.1/10 UEFI Mode
* SATA Mode: AHCI

> Una volta completato, dobbiamo sistemare ancora un paio di cose. Fai un salto alla pagina riguardo a [Apple Secure Boot](security.md)