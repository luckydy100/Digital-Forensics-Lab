# Digital Forensics Lab: Analysis of Ch01InChap01.dd

## Overview
This repository contains the complete documentation and evidence from a digital forensics practical investigation conducted at the ICDFA laboratory.

## Case Information
- **Case Name:** InChap01
- **Examiner:** Fasinu Temidayo
- **Date:** 29 August 2026
- **Status:** Complete

## Key Findings
- 4 deleted files identified
- INCOME.XLS successfully recovered using three methods (icat, blkcat, tsk_recover)
- All recovered files verified with matching MD5 hashes
- Keyword search for "George" returned relevant results

## Repository Contents

| Directory | Description |
|-----------|-------------|
| `/outputs` | Text outputs of all TSK commands |
| `/screenshots` | All evidence screenshots |
| `report.pdf` | Final forensic report |
| `report.md` | Report source file |

## Tools Used
- The Sleuth Kit (TSK)
- img_stat, mmls, fsstat, fls, istat, icat, blkcat
- losetup, mount
- md5sum, sha256sum

## Evidence Integrity
- Original image preserved (not modified)
- Chain of custody maintained
- All recovered files verified with cryptographic hashes

## Important Notice
**The original evidence image `Ch01InChap01.dd` is NOT included in this repository** to preserve evidence integrity. Only recovered files and analysis results are stored here.

## How to Reproduce
1. Install The Sleuth Kit: `sudo apt install sleuthkit`
2. Download the evidence image
3. Run the commands documented in the report
4. Verify recovered files using the provided hashes

## Report
The full forensic report is available as `report.pdf`

## License
This project is for educational purposes.
