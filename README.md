# ASCOM Alpaca driver for Raspberry Pi Camera

This is an ASCOM Alpaca driver which lets you use your Raspberry PI HQ camera in Windows based astrophotography software such as NINA or Sharpcap. The ASCOM installation on your PC will communicate over the network with this driver and make it appear like it's plugged in locally. It uses Python with LibCamera2 and AlpycaDevice. It's compatible with any Raspberry Pi running Bullseye Raspbian or later.

Currently, it will only recognise a Raspberry Pi HQ camera, however it's been coded to allow easy addition of other cameras.

It supports 
* binning
* subframes
* gain
* sensor temperature feedback
* exposure abort
* imagebytes downloads for better performance

Tested with:-
* NINA

Happy if anyone wants to add support for other cameras or test it with other programs.

*Note, Currently (as of 2.2) NINA always assumes RGGB bayer when it displays in the imaging tab for any ASCOM driver, not just this one. You should change NINA settings to force BGGR for HQ camera. However, regardless of the settings, the output FITS files will always be correct*

## Installation and Usage

### Step 1: Raspberry Pi Setup
To get this driver to work, clone this repository onto your Raspberry Pi and install the necessary dependencies:

```bash
# Clone the repository
git clone https://github.com/IanCassTwo/rpicam-ascom-alpaca.git
cd rpicam-ascom-alpaca

# Install Python packages
pip3 install falcon toml orjson

# Install required system packages
sudo apt-get update
sudo apt-get install python3-picamera2 python3-lxml python3-astropy
```

### Step 2: Start the Driver
Run the application on the Raspberry Pi:
```bash
python app.py
```
*(You may want to set this up as a systemd service or inside a `tmux` session to keep it running in the background).*

### Step 3: ASCOM Configuration (Windows PC)
1. Ensure the **ASCOM Platform** is installed on your Windows PC.
2. Open the **ASCOM Diagnostics** tool.
3. Click on **Choose and Connect to a Device**.
4. Select **Camera** from the dropdown menu.
5. In the ASCOM Chooser window, open the **Alpaca** menu and turn on **Alpaca device discovery**.
6. The tool should automatically find the Raspberry PI camera on your network and offer to install it.
7. From that point onwards, the camera will be available as an ASCOM driver in software like NINA, Sharpcap, or PHD2.

This project was made possible by the ASCOM AlpycaDevice SDK https://github.com/ASCOMInitiative/AlpycaDevice and the Python Picamera2 SDK https://github.com/raspberrypi/picamera2
Original rpicam-ascom-alpaca repo: https://github.com/IanCassTwo/rpicam-ascom-alpaca

## Fixes for NINA and ASCOM Alpaca
This fork includes essential stability and format fixes for reliable `ImageBytes` streams to Windows ASCOM clients (especially NINA):

1. **Overbright Images Output Constraints**: The PiHQ `master` code handled raw packed bytes by blowing them out with `* 16`. The LibCamera outputs 16-bit unpacked structs with left-aligned 12-bit sequences. This caused non-exact values. Replaced scaling with `np.right_shift(array.view(np.uint16), 4)` which securely transforms raw MIPI packet streams directly into clean `0 - 4095` value ranges without any datatype overflow upcasting.
2. **NINA Disconnects & BlockCopy .NET crashes**: Previously, `shr.py` attempted to stream an `ImageElementType` of `Int32` (2) zipped alongside a `TransmissionElementType` of `UInt16` (8). The mismatch directly crashed older .NET array parsing back in NINA's `AscomCamera.cs` client dependency due to a `.NET Buffer.BlockCopy` arrays bounds error. This is patched by circumventing dynamic mappings and globally streaming data out natively as flat uncompressed `Int32` chunks back to ASCOM deserializers, keeping connection persistence rock-solid through hour-long imaging runs!
3. **Half-Black Cursed Arrays/Dimensions**: Standard ASCOM image transmission requires a *Column-Major* format matrix where the X axis iterates quickest across the array structure HTTP stream layout. Python inherently flattens arrays in `C-order` (Y varies fastest), so NINA would incorrectly diagonally-slice 2D images resulting in halved noise bounds along 3040x3040 layouts! Explicitly replacing configurations with `self.Value.astype(np.int32, order='F')` forces `numpy` to map bytes by complying exactly with Alpaca dimensionality rules.
