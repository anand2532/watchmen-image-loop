# OpenMV RT1062 Storage Issue - Image Corruption on SD Card

This repository contains test scripts to reproduce and document image corruption issues when saving and retrieving images from SD card on OpenMV RT1062 board.

## Problem Description

When saving images to SD card on OpenMV RT1062, the images become corrupted and cannot be read back properly. The main issue is demonstrated by:

### Test Case 1: Raw JPG Image Corruption
- **Error**: `ERROR, [IMG] Failed read raw image file:/sdcard/myimages/221_1735689732000_raw.jpg, error: Unsupported format!`
- **Description**: After saving a JPG image to SD card using `img.save()`, the saved file cannot be read back and appears corrupted. The file exists on the SD card but the image data is invalid or incomplete.

### Test Case 2: Binary File Read/Write Corruption
- **Error**: `ERROR, [IMG] Failed to decrypt and save image: key`
- **Description**: Binary files saved to SD card become corrupted when read back, indicating data integrity issues during write/read operations.

## Repository Structure

```
storage-issue/
├── main.py          # Main script: continuous image capture, saves raw images to SD card
├── test1.py         # Test Case 1: Read saved JPG image → verify read → hybrid encryption → save encrypted binary to <filename>.enc 
├── test2.py         # Test Case 2: Read encrypted binary file → process and decrypt image → save raw jpg image to <filename>.jpg 
├── enc.py           # Encryption utilities (used by test scripts for processing)
├── enc_priv.py      # Private key repository (used by test scripts)
├── rsa/             # RSA encryption library (dependency)
│   ├── __init__.py
│   ├── key.py
│   ├── pkcs1.py
│   └── ... (other RSA modules)
└── util/
    └── decode_priv_key.py
```

## Setup Instructions

### 1. Hardware Requirements
- OpenMV RT1062 board
- SD card (formatted, inserted in board)
- Camera sensor (for `main.py` only)

### 2. File Upload to OpenMV Board

Upload all Python files to your OpenMV board:

**Required Files:**
- `enc.py`
- `enc_priv.py`
- `test1.py`
- `test2.py`
- `main.py` (optional, for full capture workflow)
- `rsa/` directory (all files)
- Public key files: `221.pub`, etc. (if using encryption in test scripts)

**Upload Instructions:**
Copy all the required files and folders listed above to your OpenMV board's storage (either `/flash` or `/sdcard`). You can use OpenMV IDE's file transfer feature or any other method to upload the files to the board.

### 3. Public Key Files (if needed)

If using test scripts that require encryption processing, ensure you have the public key file(s) on the board (e.g., `221.pub`) in the root directory.

The public key files should contain the RSA modulus `n` as a single integer on the first line:
```
93707570084882290967203367102885687072410701389989352055861123607039021853225999041041941636338758170666665448059059611663612872782499619381386775526365690664987324671998372135663320340834574407710804137444077738151368134385803551377563991839820527547363888541191583559632075302668890641139065893216092760433
```

## Usage

### Test Case 1: Save and Read JPG Image

This test saves a JPG image to SD card and attempts to read it back, demonstrating the corruption issue.

**Prerequisites:**
- A JPG image file must exist at: `/sdcard/myimages/221_1735689810000_raw.jpg`
- Modify the file path in `test1.py` if needed (lines 66, 68)

**Run the test:**
```python
# In OpenMV IDE, open and run test1.py
exec(open('/flash/test1.py').read())
```

**What it does:**
1. Reads a JPG image from SD card
2. Processes the image bytes (may apply encryption for testing)
3. Saves processed bytes to `.enc` file on SD card
4. Uses `os.sync()` and `utime.sleep_ms(500)` to ensure write completion

**Expected behavior:** File should be saved successfully and readable

**Observed issue:** File may be corrupted when written to SD card, causing read failures

---

### Test Case 2: Save and Read Binary File

This test saves a binary file to SD card and attempts to read it back.

**Prerequisites:**
- A binary `.enc` file must exist at: `/sdcard/myimages/221_1735689700000.enc`
- Modify the file path in `test2.py` if needed (lines 64, 66)
- Update `creator` variable (line 58) if needed

**Run the test:**
```python
# In OpenMV IDE, open and run test2.py
exec(open('/flash/test2.py').read())
```

**What it does:**
1. Reads binary `.enc` file from SD card
2. Processes the file bytes
3. Creates `image.Image` object from processed bytes
4. Saves as JPG file on SD card

**Expected behavior:** Binary file should be read successfully and JPG image should be saved

**Observed issue:** 
- Binary file may be corrupted when read from SD card, causing processing to fail
- If read succeeds but data is corrupted, JPG save fails with "Unsupported format!"

---

### Full Workflow: main.py

The `main.py` script demonstrates the complete workflow:
1. Captures images from camera sensor
2. Saves raw JPG to SD card using `img.save()`
3. Reads back the saved JPG to verify integrity
4. Processes and saves additional file formats

**Run the full workflow:**
```python
# In OpenMV IDE, open and run main.py
exec(open('/flash/main.py').read())
```

**Configuration:**
- Modify device ID mapping (lines 23-44) if your board has different unique ID
- Adjust sensor settings (lines 100-101) as needed
- Modify `capture_count` and timing if needed

## Troubleshooting

### SD Card Not Detected
- Ensure SD card is properly inserted and formatted (FAT32 recommended)
- Check that `/sdcard` directory is accessible: `os.listdir('/sdcard')`
- Script will fall back to `/flash` if SD card not available

### Import Errors
- Ensure all files are uploaded to the board
- Check that `rsa/` directory and all its files are present
- **Logger Module**: The scripts use a `logger` module (imported in `enc.py` and `enc_priv.py`). If you don't have this module, you can create a simple stub:
  ```python
  # Create logger.py with:
  class logger:
      @staticmethod
      def debug(msg): print(f"DEBUG: {msg}")
      @staticmethod
      def info(msg): print(f"INFO: {msg}")
      @staticmethod
      def error(msg): print(f"ERROR: {msg}")
      @staticmethod
      def warning(msg): print(f"WARNING: {msg}")
  ```
  Or modify `enc.py` and `enc_priv.py` to replace `logger.debug/info/error` calls with `print()` statements.

### Memory Issues
- Scripts include explicit garbage collection (`gc.collect()`) after image operations
- If memory errors occur, try reducing image size or adding more `gc.collect()` calls

## Technical Details

### File I/O Operations
- **Image Save**: Uses `img.save(filepath)` method to save JPG images
- **Binary Write**: Uses `open(filepath, "wb")` with `f.write()` for binary data
- **Binary Read**: Uses `open(filepath, "rb")` with `f.read()` for binary data
- **Sync**: Uses `os.sync()` after writes to ensure data is flushed to SD card
- **Delay**: Includes `utime.sleep_ms(500)` delay after sync for filesystem stability

### Image Format
- JPEG format (RGB565 source, JPEG output)
- Image size: 320x240 (HD resolution)
- Uses OpenMV `image.Image` class for encoding/decoding

### Observed Issues
- Files appear to be written to SD card (file exists, has size)
- But when read back, data is corrupted or incomplete
- Issue affects both JPG images saved via `img.save()` and binary files written via `f.write()`
- Problem persists even with `os.sync()` and delays

## Forum Discussion

This repository is created for discussion on the OpenMV forum regarding:
- Image corruption when saving to SD card on RT1062
- File corruption during write/read operations
- Potential filesystem sync issues
- Best practices for reliable SD card I/O on OpenMV RT1062

**Test Results:**
Please share your test results, including:
- OpenMV firmware version
- SD card specifications (size, brand, speed class, filesystem format)
- Whether issues occur with both raw JPG and binary files
- File sizes and corruption patterns observed
- Any workarounds or solutions found

## License

This is a test/demonstration repository for debugging purposes.
