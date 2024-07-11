---
title: 'Checksum'
type: docs
---

In computer science, a **checksum** is a value derived from the data contained in a data packet or a file. It is a small-sized piece of data that is calculated from the original data and is used to ensure the integrity of the data during transmission or storage. The purpose of a checksum is to detect errors that may have occurred during data transmission or storage.

The checksum is typically calculated using a mathematical algorithm, often a hash function or a cyclic redundancy check (CRC). The resulting checksum is then sent or stored along with the data. When the data is received or retrieved, the checksum is recalculated, and the new checksum is compared to the original checksum. If the two checksums match, it indicates that the data is likely to be intact. If they don't match, it suggests that errors may have occurred, and the data integrity is compromised.

Checksums are commonly used in networking protocols, file storage, error-checking mechanisms, and various data verification processes. They provide a simple and efficient way to detect errors without the need for extensive redundancy or complex error-correction algorithms. However, it's important to note that checksums are not foolproof and are primarily designed to catch accidental errors, not malicious tampering. For more robust error detection and correction, techniques like error-correcting codes are used.