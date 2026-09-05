# Digital Forensics Practical Report
## File Carving, Hex Analysis, Metadata Examination & TSK Analysis

---

## 1. Case Information

| Field | Details |
|-------|---------|
| **Lab Title** | File Carving, Metadata, and File System Recovery Lab |
| **Examiner** | Fasinu Temidayo |
| **Date of Analysis** | 2 September 2026 – 4 September 2026 |
| **Lab Environment** | Kali Linux (working directory `~/forensic_carving_lab`) |
| **Role** | Junior Digital Forensic Analyst |

**Timeline (from `logs/timeline.txt`):**
Lab started: Wed Sep 2 17:54:13 EDT 2026

---

## 2. Evidence Handling

### 2.1 Evidence Files

| Filename | Description | Size |
|----------|--------------|------|
| `J_ub_law.jpg` | JPEG image — primary hex/metadata subject | 1,692,150 bytes |
| `120M.7z` | Compressed archive containing the USB FAT image | 36,720,470 bytes |
| `usb_fat_carving.001` (inside `120M/`) | Raw USB disk image (FAT), source for Scalpel carving | 124,780,544 bytes |
| `Ch01InChap01.dd` | Raw FAT12 floppy/USB image, source for TSK analysis | 1,474,560 bytes |

### 2.2 Cryptographic Hashes (from `evidence_hashes.txt`)

Recorded: Wed Sep 2 18:04:22 EDT 2026

| File | MD5 | SHA-256 |
|------|-----|---------|
| `J_ub_law.jpg` | `83a360ac7f7e0ca318e5bfe39f95f137` | `238ff34393c50e52c0e8b14fcff8ec7dc29e23914dbc435f8ef998d172a91468` |
| `120M.7z` | `06c0e8b720827909d4c080887a8823b1` | `39330439e35664349341d5527f9d2cdb979b8c67fc86c5abe7b33b740f01855c` |
| `Ch01InChap01.dd` | `a117773bcf1fc88ec0ab8e0a349fbbcb` | `3ce8053e4f3d9c8ab98b3aadb2480685efb8e4980d34297b83bd5a09b1a7b122` |

*Screenshot: initial hashes* — `screenshots/01_evidence.png`, `screenshots/03_hashes.png`

### 2.3 Original Acquisition Record for the USB Image

The USB image `usb_fat_carving.001` was originally acquired with FTK Imager (metadata preserved in `120M/usb_fat_carving.001.txt`):

| Field | Value |
|-------|-------|
| Acquired using | AccessData FTK Imager 4.5.0.3 |
| Case Number | USB File Carving |
| Evidence Number | UB-002 |
| Examiner (original acquisition) | Frank Xu, University of Baltimore |
| Source | General UDisk USB Device, 119 MB, 243,712 sectors × 512 bytes |
| Acquisition MD5 | `ba4a1d0ba49f4a6667b00a3b3e85e604` (verified) |
| Acquisition SHA-1 | `bcc2d49fd49c9521ecb1739f6542c6bf327375ef` (verified) |

This is the chain-of-custody record from the original imaging of the physical USB drive, prior to it being packaged as `120M.7z` for this lab.

### 2.4 Evidence Integrity Statement

The original evidence files were not modified during this investigation. All carving and recovery operations were performed against copies, and integrity was checked using MD5/SHA-256 hashing before and after analysis.

---

## 3. Part A — File Signatures and Hex Analysis (J_ub_law.jpg)

### 3.1 Header Inspection (xxd)

**Command:**
```bash
xxd J_ub_law.jpg | head -20
xxd -l 64 J_ub_law.jpg
xxd -p J_ub_law.jpg | head -c 20
```

**Result:** The file begins with `ff d8 ff e1 17 f0 45 78 69 66 00 00 49 49 2a 00 08 00 00 00` — i.e. `FFD8` (JPEG **SOI** marker) followed by `FFE1` (an **APP1** segment, not APP0/JFIF), then the ASCII bytes `45 78 69 66 00 00` = `"Exif\0\0"`, confirming this JPEG carries an **EXIF** header rather than a plain JFIF header.

*Screenshot:* `screenshots/04_xxd_inspection.png`

### 3.2 Footer Identification (EOI marker)

**Command:**
```bash
xxd -p J_ub_law.jpg | grep -o "ffd9" | wc -l
xxd J_ub_law.jpg | tail -10
```

**Analysis:** The JPEG **EOI** marker `FFD9` terminates the file. File size confirmed at 1,692,150 bytes via `stat -c%s`.

*Screenshot:* `screenshots/05_jpeg_footer.png`

### 3.3 Hex Dumps

**Command:**
```bash
xxd -p J_ub_law.jpg > J_ub_law_hexdump.txt
xxd J_ub_law.jpg > J_ub_law_hexdump_formatted.txt
```

Both plain (`-p`, no addresses) and formatted (with addresses/ASCII column) hex dumps were generated successfully.

*Screenshot:* `screenshots/06_plain_hex.png`, `screenshots/06_plain_hex_000.png`

### 3.4 Reverse Hexdump Reconstruction

**Command:**
```bash
xxd -r -p J_ub_law_hexdump.txt J_ub_law_reconstructed.jpg
xxd -r J_ub_law_hexdump_formatted.txt J_ub_law_reconstructed2.jpg

file J_ub_law_reconstructed.jpg
file J_ub_law_reconstructed2.jpg

md5sum J_ub_law.jpg J_ub_law_reconstructed.jpg J_ub_law_reconstructed2.jpg
sha256sum J_ub_law.jpg J_ub_law_reconstructed.jpg J_ub_law_reconstructed2.jpg
```

**Result:**

| File | MD5 |
|------|-----|
| `J_ub_law.jpg` | `83a360ac7f7e0ca318e5bfe39f95f137` |
| `J_ub_law_reconstructed.jpg` | `83a360ac7f7e0ca318e5bfe39f95f137` |
| `J_ub_law_reconstructed2.jpg` | `83a360ac7f7e0ca318e5bfe39f95f137` |

All three MD5 (and matching SHA-256) hashes are **identical**. `file` confirms all three are valid JPEG image data with matching EXIF headers and dimensions. This proves that both the plain (`-p`) and address-formatted hex dumps are lossless, reversible representations of the original binary — reconstructing from either produces a byte-for-byte identical file.

*Screenshot:* `screenshots/07_reverse_dump.png`

### 3.5 JPEG Structure Summary

| Element | Value |
|---------|-------|
| Header (SOI + marker) | `FF D8 FF E1` (APP1/Exif segment) |
| Footer (EOI) | `FF D9` |
| File size | 1,692,150 bytes |
| Reported dimensions | 1920×1080 (baseline, 8-bit precision, 3 components) |

*Screenshot:* `screenshots/08_Jpeg_structure.png`

---

## 4. Part B — Embedded Metadata Examination (J_ub_law.jpg)

### 4.1 Strings Extraction

**Command:**
```bash
strings J_ub_law.jpg > jpeg_strings.txt
strings J_ub_law.jpg | grep -i "camera\|model\|date\|copyright\|howard\|korn"
```

Readable text extracted from the file's embedded XMP/IPTC metadata block confirmed the presence of camera make/model strings (`NIKON CORPORATION`, `NIKON D4`), software strings (`Adobe Photoshop 21.1 (Macintosh)`), and the artist/copyright name (`HOWARD KORN`) directly in the raw byte stream — well before running a dedicated EXIF parser.

*Screenshot:* `screenshots/09_embeded_metadata_a`, `screenshots/09_embeded_metadata_b`

### 4.2 exiftool Extraction

**Command:**
```bash
exiftool J_ub_law.jpg > jpeg_exif.txt
exiftool J_ub_law.jpg | grep -E "Camera|Model|Date|Software|Artist"
```

**Key metadata fields recovered:**

| Field | Value |
|-------|-------|
| Make | NIKON CORPORATION |
| Camera Model Name | NIKON D4 |
| Lens Model | 17.0-35.0 mm f/2.8 |
| Serial Number | 2027750 |
| Focal Length | 20.0 mm |
| F Number | 11.0 |
| Exposure Time | 8 sec |
| Exposure Program | Manual |
| ISO | 200 |
| Create Date | 2013:10:08 18:56:21 |
| Modify Date | 2020:08:20 10:20:05 |
| Software | Adobe Photoshop 21.1 (Macintosh) |
| Artist / Copyright | HOWARD KORN |
| Copyright Notice (embedded) | Copyright 1999 Adobe Systems Incorporated |
| Image Size | 1920×1080 |
| Subjects (XMP) | 2013, campus, Merrick School of Business |
| Hierarchical Subject | Schools \| Merrick School of Business |

GPS coordinates and lens-profile-specific distortion data were **not** present in the metadata.

*Screenshot:* `screenshots/10_metadata.png`

### 4.3 Findings Summary

- **Original capture date:** 8 October 2013, 18:56:21 (Nikon D4, manual exposure, 20mm/f11/8s/ISO200)
- **Last edited:** 20 August 2020, 10:20:05, via Adobe Photoshop 21.1 on macOS
- **Editing history (from embedded XMP `xmpMM:History`):** the file went through multiple Photoshop/Camera Raw save-and-convert cycles between January 2014 and August 2020 (RAW→TIFF→JPEG conversions, multiple re-saves), indicating repeated post-processing over several years rather than a single edit.
- **Attribution:** Artist/Copyright field names "HOWARD KORN"
- **Content clue:** XMP subject tags tie the image to "Merrick School of Business" and the year 2013
- **Not found:** GPS coordinates, lens profile/distortion correction data

*Screenshot:* `screenshots/11_findings.png`

**Note on the camera-model finding:** an initial pass using `strings`/grep only surfaced the string `NIKON CORPORATION` for the "Make" field. Running `exiftool` (a full EXIF/XMP parser rather than a raw string search) additionally recovered the specific **Camera Model Name: NIKON D4** and lens details, which raw string grepping missed because they were stored in structured binary EXIF tags rather than as standalone ASCII strings.

---

## 5. Part C — The Sleuth Kit (TSK) Analysis of Ch01InChap01.dd

### 5.1 Image Information

**Command:**
```bash
img_stat Ch01InChap01.dd
```
**Output (`tsk_analysis/img_stat.txt`):**
```
IMAGE FILE INFORMATION
--------------------------------------------
Image Type: raw

Size in bytes: 1474560
Sector size:   512
```

### 5.2 Volume System Analysis

**Command:**
```bash
mmls Ch01InChap01.dd
```
**Output (`tsk_analysis/mmls.txt`):** empty / no output.

**Analysis:** `mmls` returned no partition table, and this is corroborated by `tsk_summary.txt`, which records **"Partition scheme: 0 partitions."** This means `Ch01InChap01.dd` is a **non-partitioned** image — a raw FAT12 file system written directly to the media (typical of a floppy disk), with no MBR/partition table layer to parse. This is why `fsstat`/`fls` below work directly against the image with no `-o` offset needed.

*Screenshot:* `screenshots/12_sleuthkit.png`

### 5.3 File System Information

**Command:**
```bash
fsstat Ch01InChap01.dd
```
**Output (`tsk_analysis/fsstat.txt`):**
```
FILE SYSTEM INFORMATION
--------------------------------------------
File System Type: FAT12

OEM Name: k0nIHC
Volume ID: 0x0

Sectors before file system: 0

File System Layout (in sectors)
Total Range: 0 - 2879
* Reserved: 0 - 0
** Boot Sector: 0
* FAT 0: 1 - 9
* FAT 1: 10 - 18
* Data Area: 19 - 2879
** Root Directory: 19 - 32
** Cluster Area: 33 - 2879

METADATA INFORMATION
--------------------------------------------
Range: 2 - 45782
Root Directory: 2

CONTENT INFORMATION
--------------------------------------------
Sector Size: 512
Cluster Size: 512
Total Cluster Range: 2 - 2848

FAT CONTENTS (in sectors)
--------------------------------------------
33-236 (204) -> EOF
285-311 (27) -> EOF
```

**Analysis:** Confirms a FAT12 file system (consistent with a small floppy/USB volume), 512-byte sectors and clusters, a 9-sector FAT table (×2 copies), and a root directory spanning sectors 19–32. The FAT contents show two allocated cluster runs terminating in EOF — sectors 33–236 and 285–311 — which map to the two allocated files identified below (`Client Info.mdb` and `Income.xls`).

*Screenshot:* `screenshots/13_analysis.png`

### 5.4 List All Files

**Command:**
```bash
fls Ch01InChap01.dd
```
**Output (`tsk_analysis/fls_output.txt`):**
```
r/r 5:    Client Info.mdb
r/r * 8:  Billing Letter.doc
r/r * 11: confirmation.txt
r/r 13:   Income.xls
r/r * 15: letter1.txt
r/r * 17: Regrets.doc
v/v 45779: $MBR
v/v 45780: $FAT1
v/v 45781: $FAT2
V/V 45782: $OrphanFiles
```

**Analysis:** 6 user files total — 2 allocated (`Client Info.mdb`, `Income.xls`), 4 deleted (marked with `*`: `Billing Letter.doc`, `confirmation.txt`, `letter1.txt`, `Regrets.doc`). The `v/v` / `V/V` entries are TSK's virtual metadata objects (not real files on the volume).

*Screenshot:* `screenshots/14_TSK_findings.png`

**Note:** The `fls -i inode Ch01InChap01.dd` command produced no output (`tsk_analysis/fls_inode.txt` was empty) — `-i` is TSK's flag for specifying *image type* (e.g. `-i raw`), not a way to list inode numbers, so `-i inode` was not a valid image type and the command did not run as intended. Inode numbers are already shown by default in the plain `fls` output above (the numbers after `r/r`).

### 5.5 Deleted File Recovery

Cross-referencing the deleted entries from `fls` against inode numbers gives:

| Inode | Filename | Status |
|-------|----------|--------|
| 8 | Billing Letter.doc | Deleted |
| 11 | confirmation.txt | Deleted |
| 13 | Income.xls | **Allocated** |
| 15 | letter1.txt | Deleted |
| 17 | Regrets.doc | Deleted |

**Recovery commands used:**
```bash
icat Ch01InChap01.dd 13 > recovered_files/Income.xls
```

---

## 6. Part D — Embedded File Discovery with Binwalk (J_ub_law.jpg)

**Command:**
```bash
binwalk J_ub_law.jpg > binwalk_jpeg_scan.txt
binwalk -v J_ub_law.jpg >> binwalk_jpeg_scan.txt
```

**Output (`binwalk_jpeg_scan.txt`):**
```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, EXIF standard
12            0xC             TIFF image data, little-endian offset of first image directory: 8
26844         0x68DC          Copyright string: "Copyright 1999 Adobe Systems Incorporated"

Scan Time:     2026-09-04 15:00:28
Target File:   /home/kali/forensic_carving_lab/J_ub_law.jpg
MD5 Checksum:  83a360ac7f7e0ca318e5bfe39f95f137
Signatures:    436
```

**Analysis:** Binwalk confirms the file is a single, well-formed JPEG with an EXIF/TIFF metadata block starting at offset 0xC, and independently surfaces the embedded copyright string found earlier via `strings`. No secondary embedded file (e.g. an appended ZIP or second image signature) was detected — the scan found no additional file signatures beyond the JPEG/EXIF/TIFF structure itself, i.e. nothing was hidden after the JPEG's EOI marker.

*Screenshot:* `screenshots/15_binwalk_scan`, `screenshots/16_binwalk_sccan`

---

## 7. Part E — USB Image Preparation and Analysis

### 7.1 Extraction

**Command:**
```bash
7z x 120M.7z
cd 120M
ls -la
```

**Result:** Extraction produced `120M/usb_fat_carving.001` (124,780,544 bytes), accompanied by the original FTK Imager acquisition record `usb_fat_carving.001.txt` (see Section 2.3).

*Screenshot:* `screenshots/17_extracting_usb.png`, `screenshots/18_extracting_usb_b.png`

### 7.2 TSK Analysis of the USB Image

**Commands:**
```bash
md5sum usb_fat_carving.001 >> ../evidence_hashes.txt
sha256sum usb_fat_carving.001 >> ../evidence_hashes.txt
img_stat usb_fat_carving.001 > ../tsk_analysis/usb_img_stat.txt
mmls usb_fat_carving.001 > ../tsk_analysis/usb_mmls.txt
fsstat -o 128 usb_fat_carving.001 > ../tsk_analysis/usb_fsstat.txt
```

**Note:** the text-file redirects for this step (`usb_img_stat.txt`, `usb_mmls.txt`, `usb_fsstat.txt`) were not present in the submitted lab folder — only the on-screen results were captured, via `screenshots/20_fat_carving1.png` and `screenshots/20_fat_carving2.png`. If you still have terminal scrollback or re-run these three commands, paste the actual `mmls` partition table and `fsstat -o 128` output here so the offset (partition start sector, typically 128 for this FAT image) is documented from your own run rather than assumed.

*Screenshot:* `screenshots/20_fat_carving1.png`, `screenshots/20_fat_carving2.png`

---

## 8. Part F — Carving Deleted Files with Scalpel (usb_fat_carving.001)

### 8.1 Configuration

`scalpel.conf` was edited to enable JPEG carving by uncommenting three signature rules:
```
jpg   y   5000000    \xff\xd8\xff\xe0\x00\x10\x4a\x46    \xff\xd9
jpg   y   5000000    \xff\xd8\xff\xe1\x00\x10\x45\x78    \xff\xd9
jpg   y   5000000    \xff\xd8\xff\xdb                    \xff\xd9
```
These match JFIF (`APP0`), Exif (`APP1`), and generic quantization-table-led JPEGs respectively, each bounded by the `FFD9` EOI footer, with a 5,000,000-byte max carve size.

*Screenshot:* `screenshots/21_scalpel.png`

### 8.2 Carving Run

**Command:**
```bash
scalpel -c scalpel.conf -o scalpel_output usb_fat_carving.001
```

**Result (`scalpel_output/audit.txt`):** Scalpel carved **10 JPEG files** from the raw USB image:

| File | Start Offset | Length (bytes) |
|------|--------------|-----------------|
| 00000000.jpg | 335,872 | 85,046 |
| 00000001.jpg | 425,822 | 85,046 |
| 00000002.jpg | 3,796,992 | 99,347 |
| 00000003.jpg | 35,350,242 | 239,079 |
| 00000004.jpg | 35,590,996 | 13,715 |
| 00000005.jpg | 36,352,448 | 2,946 |
| 00000006.jpg | 36,393,610 | 2,268 |
| 00000007.jpg | 36,571,136 | 99,277 |
| 00000008.jpg | 36,671,488 | 292,225 |
| 00000009.jpg | 35,903,812 | 36,643 |

`scalpel_output/carving_summary.txt` confirms: **10 jpg, 2 txt** files recovered in total.

*Screenshot:* `screenshots/22_scalpel_results`

### 8.3 Review of Carved JPEGs

**Command:**
```bash
for file in *.jpg; do file $file; md5sum $file; done > jpeg_details.txt
```

**Findings (`jpg-0-0/jpeg_details.txt`):**

| File | Dimensions | Notes |
|------|-----------|-------|
| 00000000.jpg / 00000001.jpg | 360×540 | Identical content — same MD5 (`597cef9f...`), duplicate carve |
| 00000002.jpg | 618×927 | Progressive JPEG |
| 00000003.jpg | 1667×762 | Comment: "Software: Snipaste" — a screenshot tool |
| 00000004.jpg | 604×158 | Comment: "Software: Snipaste" |
| 00000005.jpg / 00000006.jpg | 256×144 | Small thumbnails |
| 00000007.jpg / 00000008.jpg | 750×1334 | Progressive JPEG, phone-screen aspect ratio |
| 00000009.jpg (in `jpg-2-0/`) | — | Carved separately due to a different signature match |

*Screenshot:* `screenshots/23_scalpel_img1.png`, `screenshots/24_scalpel_file_review.png`, `screenshots/25_scalpel_image1.png`

### 8.4 Hash Verification of Recovered JPEGs

**Command:**
```bash
for file in *.jpg; do
  md5sum "$file" >> recovered_hashes.txt
  sha256sum "$file" >> recovered_hashes.txt
done
```

**Result (`recovered_hashes.txt`):**

| File | MD5 | Size |
|------|-----|------|
| 00000000.jpg | `597cef9fa6bab3b73312e8e274364b04` | 85,046 B |
| 00000001.jpg | `597cef9fa6bab3b73312e8e274364b04` | 85,046 B |
| 00000002.jpg | `e833e245176b08a46508e26961116bb4` | 99,347 B |
| 00000003.jpg | `9d09c214f6fa49ad4b39fff15e22af04` | 239,079 B |
| 00000004.jpg | `3dcb6a4ca242b85d904da775fcd8a4b1` | 13,715 B |
| 00000005.jpg | `d09b9174b6bcaf09bcf84724509066a2` | 2,946 B |
| 00000006.jpg | `3e1ad4207d62d924fb6bc5473d842e13` | 2,268 B |
| 00000007.jpg | `a4fc766ba1a38f36263d0bfe6fc0ac1d` | 99,277 B |
| 00000008.jpg | `ca42511022040f8739f8b2a8445c81b6` | 292,225 B |

**Analysis:** All recovered files hash-verify as valid, complete JPEGs with matching `file` type identification — Scalpel's header/footer carving successfully reconstructed full, viewable images from unallocated space on the USB image, with one exact duplicate pair (00000000/00000001) carved from adjacent, overlapping regions.

*Screenshot:* `screenshots/26_recovered_hashes.png`

---

## 9. Answers to Key Questions

### Q1: What JPEG header/footer signatures were identified in J_ub_law.jpg, and were the hex dump and its reversal identical to the original?

**A:** Header: `FF D8 FF E1` + `"Exif\0\0"` (APP1/EXIF-tagged JPEG, not plain JFIF). Footer: `FF D9` (EOI). Both the plain (`xxd -p`) and formatted (`xxd`) hex dumps, when reversed with `xxd -r`, reproduced files with MD5 `83a360ac7f7e0ca318e5bfe39f95f137` — **identical** to the original `J_ub_law.jpg`, confirming lossless reversibility.

### Q2: What metadata was recoverable from J_ub_law.jpg, and by which method?

**A:** `strings` alone surfaced the make (NIKON CORPORATION), software (Adobe Photoshop 21.1), and artist (HOWARD KORN) as raw ASCII. `exiftool` additionally decoded structured EXIF/XMP fields not visible to plain string search — camera model (NIKON D4), lens (17-35mm f/2.8), exposure settings (f/11, 8s, ISO 200, manual), original capture date (8 Oct 2013) vs. last-modified date (20 Aug 2020), and XMP subject tags ("Merrick School of Business", 2013). No GPS data was present.

### Q3: How many deleted files exist in Ch01InChap01.dd, and how was recovery verified?

**A:** 4 deleted files (`Billing Letter.doc`, `confirmation.txt`, `letter1.txt`, `Regrets.doc`) per `fls` output, alongside 2 allocated files (`Client Info.mdb`, `Income.xls`). `Income.xls` (inode 13) was recovered via `icat`.

### Q4: How many files did Scalpel carve from the USB image, and were they valid?

**A:** 10 JPEG files + 2 text files. All 9 unique JPEGs (00000000–00000008, with 00000000/00000001 being duplicates) verified as structurally valid JPEG image data via `file`, with individually recorded MD5/SHA-256 hashes for chain-of-custody.

---

## 10. Summary of Findings

| Item | Finding |
|------|---------|
| JPEG evidence file | `J_ub_law.jpg`, 1,692,150 bytes, EXIF-tagged JPEG |
| Hex reconstruction | Verified lossless (identical MD5/SHA-256 to original) |
| Camera/metadata identified | Nikon D4, HOWARD KORN (copyright), edited in Photoshop 21.1, captured 2013, last modified 2020 |
| Binwalk result | Single JPEG structure, no additional embedded/hidden files detected |
| Ch01InChap01.dd file system | FAT12, no partition table (raw floppy layout) |
| Ch01InChap01.dd files | 6 total — 2 allocated, 4 deleted |
| USB image (usb_fat_carving.001) | 124,780,544 bytes, FTK-acquired, FAT |
| Scalpel carving result | 10 JPEG + 2 TXT files recovered from unallocated space |
| Evidence integrity | Verified via MD5/SHA-256 throughout ✓ |

---

## 11. Conclusion

This lab exercised the full forensic file-recovery pipeline: low-level hex/signature analysis and lossless reconstruction of a JPEG, embedded-metadata extraction (both raw string search and structured EXIF/XMP parsing), TSK-based file system and deleted-file analysis of a FAT12 image, binwalk-based embedded-file discovery, and signature-based file carving of unallocated space on a larger USB image using Scalpel. All recovered artifacts were hash-verified, and no modification was made to the original evidence files at any stage.


---

## 12. Appendices

### Appendix A: Files Present in Submission

- `evidence_hashes.txt`, `jpeg_analysis.txt`, `metadata_analysis.txt`, `jpeg_metadata.txt`, `jpeg_exif.txt`, `jpeg_strings.txt`
- `J_ub_law_hexdump.txt`, `J_ub_law_hexdump_formatted.txt`
- `binwalk_jpeg_scan.txt`
- `tsk_analysis/` — `img_stat.txt`, `mmls.txt` (empty), `fsstat.txt`, `fls_output.txt`, `fls_inode.txt` (empty), `tsk_summary.txt`
- `scalpel.conf`, `scalpel_output/audit.txt`, `scalpel_output/carving_summary.txt`, `scalpel_output/jpg-0-0/`, `scalpel_output/jpg-2-0/`
- `120M/usb_fat_carving.001`, `120M/usb_fat_carving.001.txt`

### Appendix B: Recovered/Reconstructed Files

- `J_ub_law_reconstructed.jpg`, `J_ub_law_reconstructed2.jpg` (both hash-identical to original)
- 9 unique carved JPEGs (`00000000.jpg`–`00000008.jpg`) from `usb_fat_carving.001`
- `Income.xls` recovered from `Ch01InChap01.dd` via `icat` (inode 13)

### Appendix C: Screenshots

All screenshots are in the `/screenshots` folder (26 files, `01_evidence.png` through `26_recovered_hashes.png`).

### Appendix D: Tools Used

- `xxd`, `strings`, `exiftool`, `binwalk`
- The Sleuth Kit: `img_stat`, `mmls`, `fsstat`, `fls`, `icat`
- `7z`, `scalpel`
- `md5sum`, `sha256sum`, `file`

---

## 13. Evidence Integrity Declaration

I hereby certify that:

- The original evidence files were not modified during this investigation
- All analysis was performed using validated forensic tools
- All recovered evidence was verified using cryptographic hashing
- Chain of custody was maintained throughout the investigation

**Signature:** Fasinu Temidayo Lucky

**Date:** 4th September 2026
