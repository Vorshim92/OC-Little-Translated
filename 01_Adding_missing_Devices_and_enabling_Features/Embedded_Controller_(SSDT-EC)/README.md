# Adding a fake Embedded Controller (`SSDT-EC`) or (`SSDT-EC-USBX`) 

## About Embedded Controllers
An embedded controller (or `EC`) is a hardware microcontroller inside of computers (especially Laptops) that can handle a lot of different tasks: from receiving and processing signals from keyboards and/or touchpads, thermal measurements, managing the battery, handling switches and LEDs, handling USB power, Bluetooth toggle, etc., etc.

### Why do I need a fake EC?
On Desktop PCs, the `EC` device usually isn't named correctly for what macOS expects, so it can be attached to `AppleACPIEC` driver – which is a good thing in this case since EC devices from PCs are incompatible with macOS and may break at any time so `AppleACPIEC` kext must NOT load on desktops. To work around this issue, we disable the device included in the system's `DSDT` and add a fake `EC` for macOS to play with instead.

### `SSDT-EC` or `SSDT-EC-USBX`: which one do I need?
In order to get USB Power Management working properly on **Skylake and newer CPUs**, we have to add a fake `EC` as well as a `USBX` device to supply USB power properties, so macOS can attach its `AppleBusPowerController` service to it. Both devices are included in `SSDT-EC-USBX`. For older systems, `SSDT-EC` alone is sufficient (if required at all).

On Laptops, the `EC` microcontroller actually really exist but may be incompatible because macOS expects a different name than what's provided by the system's `DSDT`. In this case we just use a fake EC to keep macOS happy.

So, to put it in a nutshell:

- On Desktop PCs, an existing `EC` has to be disabled and a fake EC has to be added.
- On Laptops, we just need an additional fake EC to be present (not always required).
- Skylake and newer CPUs require `SSDT-EC-USBX`, older CPUs only need `SSDT-EC`.

## Adding a fake EC Device
There are 2 methods for adding a fake EC: either by manually by adding the required `SSDT-EC `or `SSDT-EC-USBX` (depending on the used Intel CPU Family). Use either one method or the other, not both! Try NOT to rename `EC0`, `H_EC`, etc. to `EC`. These devices are incompatible with macOS and may break at any time. `AppleACPIEC` kext must NOT load on desktops.

### Method 1: Manual patching method (recommended)
- In the `DSDT`, search for `PNP0C09` 
- If multiple devices with the EisaID `PNP0C09` are discovered, confirm that it belongs to an EC Device (`EC`, `H_EC`, `EC0`, etc.).
- Depending on the type of system you are using, open either `SSDT-EC.dsl`, `SSDT-EC-USBX_Desktop.dsl` or `SSDT-EC-USBX_Laptop.dsl`
- Modify the chosen SSDT as needed so the PCI paths and name of the Low Pin Configuration Bus according to what's used in your `DSDT` (either `LPCB`or `LPC`). Read the comments in the .dsl files for more info as well.
- Export the file as .aml (ACPI Machine Language Binary) and add it to EFI > OC > ACPI and your Config
- Save and Reboot.

#### Additional Steps (Desktop PCs only)
To ensure that the existing EC in your `DSDT` does not attach to the `AppleACPIEC` driver, do the following after rebooting:

- Run IORegistryExlorer
- Search for the name of your real EC controller (`EC0`, `H_EC`, etc.)  If the device is not present, you're done!
- If the device is present (exact match!), you have to disable it. 
- Open the previously used .dsl file and remove the comments `/*` and `*/ `from the following section, so it's no longer displayed in green in MaciASL:
	```swift
    External (_SB_.PCI0.LPCB.EC0, DeviceObj)

    Scope (\_SB.PCI0.LPCB.EC0)
    {
        Method (_STA, 0, NotSerialized)  // _STA: Status
        {
            If (_OSI ("Darwin"))
            {
                Return (0) // Disables EC0 in macOS!
            }
            Else
            {
                Return (0x0F) // Leaves EC0 enabled for all other OSes!
            }
        }
    }
    ```
- Adjust the device name and PCI path accordingly to what's used in you `DSDT` 
- Export the file as .aml
- Replace the existing file in EFI > OC > ACPI
- Reboot
- Check IORegistryExplorer again for `EC0`, `H_EC` or whatever the device is called in your `DSDT`.
- If it's not present, you're done.

### Method 2: automated SSDT generation using SSDTTime
**SSDTTime** is a the python script which can generate various SSDTs from analyzing your system's `DSDT`. Unfortunately, SSDTTime does not generate a Fake EC with an included `USBX` Device. So if you are using a system with a Skylake or newer CPU, you either have to add an addtional SSDT containing the USBX device or add it the to the existing SSDT-EC or use the manual method instead!

**HOW TO:**

1. Download [**SSDTTime**](https://github.com/corpnewt/SSDTTime) and run it
2. Pres "D", drag in your system's DSDT and hit "ENTER"
3. Generate all the SSDTs you need.
4. The SSDTs will be stored under `Results` inside the `SSDTTime-master`Folder along with `patches_OC.plist`.
5. Copy the generated `SSDTs` to EFI > OC > ACPI and your Config using OpenCore Auxiliary Tools
6. Open `patches_OC.plist` and copy the included patches to your `config.plist` (to the same section, of course).
7. Save and Reboot. Done.

**Tip**: If you are editing your config using [**OpenCore Auxiliary Tools**](https://github.com/ic005k/QtOpenCoreConfig/releases), OCAT it will update the list of kexts and .aml files automatically, since it monitors the EFI folder.
