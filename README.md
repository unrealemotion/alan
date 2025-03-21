**Case Summary: NetApp MPIO LUN Disappearance after Reboot**

**1. Problem Description:**

A NetApp LUN (Logical Unit Number) presented to a Windows Server 2022 host became inaccessible (hidden) after a planned server reboot.  The server was configured to use Microsoft MPIO (Multipath I/O) with the Microsoft DSM (Device Specific Module) for managing the multipath connections to the NetApp storage array. The LUN was visible and working correctly before the reboot.

**2. Environment:**

*   **Operating System:** Windows Server 2022
*   **Storage Array:** NetApp ONTAP (specific version not provided, but assumed to be a supported version)
*   **Multipathing:** Microsoft MPIO, using the built-in Microsoft DSM (MSDSM)
*   **Connectivity:** iSCSI (The initial problem description strongly suggests iSCSI, but this should be confirmed if possible. If it's Fibre Channel, that would change some details.)
*   **Target Port Groups (TPGs):**
    *   Initially, two pairs of TPGs were configured: 1000/1001 (with Target Ports 1000:1 and 1001:2) and 5096/5097 (with Target Ports 5096:4097 and 5097:4098).
    *   The first pair (1000/1001) was *removed* before the reboot, leaving only 5096/5097 active.
    *   The root cause involves stale TPG identifiers (13e8 and 13e9) found in the VPD data.  These *do not* correspond to the initially configured TPGs.

**3. Sequence of Events:**

1.  **Initial Configuration:** The server was successfully connected to the NetApp LUN using both TPG pairs (1000/1001 and 5096/5097). The LUN was accessible.
2.  **TPG Removal:** The first TPG pair (1000/1001) was removed from the configuration. The LUN *remained accessible* through the remaining TPG pair (5096/5097). `mpclaim -v` was used to verify that only TPGs 5096/5097 were reported as active.
3.  **Planned Reboot:** The Windows Server was rebooted as part of a planned maintenance activity.
4.  **Post-Reboot Failure:** After the reboot, the NetApp LUN was no longer visible in Disk Management and was inaccessible to applications.
5. **Trouble shooting:**
By capturing MPIO and MSDSM trace, along with kernel dump, stale TPGs ID 13e8 and 13e9 were found.

**4. Key Findings and Root Cause Analysis:**

*   **MPIO Event Logs:** Two Event ID 47 entries from MPIO were logged immediately after the reboot. These events, by themselves, are *not* necessarily errors. They indicate that MPIO attempted to add the device, but a DSM did not claim it. These events corresponded to the two *expected* Target Ports (5096:4097 and 5097:4098).  The *absence* of subsequent successful device claiming events is the key indicator of a problem.
*   **MPIO Tracing:**  When MPIO tracing was enabled, and the iSCSI connections were disconnected and reconnected, the following critical error messages were captured:
    ```
    MPIOAddSingleDevice (FFFFA38E24D91050): PDO was not claimed by any DSM
    MPIODeviceRegistration() - MPIODeviceRegistration (FFFFAB83343C2050): Failed to add device (FFFFA38E24D91050) with status (c000000e)
    ```
    These errors definitively show that the Microsoft DSM (MSDSM) *failed to claim the device*, even though MPIO presented it.  The `c000000e` status code is a generic "NTSTATUS" value indicating a device hardware error, which, in this context, is a *consequence* of the DSM not claiming the device, not the root cause.
*   **VPD Page 83 Data:** The VPD (Vital Product Data) page 83 data returned by the NetApp storage array was examined. This data, which provides device identification information, was found to contain *stale* Target Port Group (TPG) identifiers: `13e8` and `13e9`. These identifiers were *not* valid at the time of the reboot (and were not part of the original, working two-TPG configuration).
*    **!scsikd.scsiinquiry Output (Harddisk1 - Relevant Snippet):**

    ```
     000: 00 83 00 70  02 01 00 20  4e 45 54 41  50 50 20 20 | ...p ... NETA PP
     010: 20 4c 55 4e  20 77 4f 6a  34 5a 3f 56  6b 7a 44 37 |  LUN  wOj 4Z?V kzD7
     020: 38 20 20 20  20 20 20 20  01 03 00 10  60 0a 09 80 | 8         .... `...
     030: 77 4f 6a 34  5a 3f 56 6b  7a 44 37 38  01 02 00 10 | wOj4 Z?Vk zD78 ....
     040: 3f 56 6b 7a  44 37 38 00  00 a0 98 77  4f 6a 34 5a | ?Vkz D78. ...w Oj4Z
     050: 01 13 00 10  60 0a 09 80  00 00 00 02  c0 a8 00 8e | .... `... .... ....
     060: 00 00 0c bc  01 14 00 04  00 00 10 01  01 15 00 04 | .... .... .... ....
     070: 00 00 13 e8                                        | ....
    ```

*    **!scsikd.scsiinquiry Output (Harddisk3 - Relevant Snippet):**

    ```
     000: 00 83 00 70  02 01 00 20  4e 45 54 41  50 50 20 20 | ...p ... NETA PP
     010: 20 4c 55 4e  20 77 4f 6a  34 5a 3f 56  6b 7a 44 37 |  LUN  wOj 4Z?V kzD7
     020: 38 20 20 20  20 20 20 20  01 03 00 10  60 0a 09 80 | 8         .... `...
     030: 77 4f 6a 34  5a 3f 56 6b  7a 44 37 38  01 02 00 10 | wOj4 Z?Vk zD78 ....
     040: 3f 56 6b 7a  44 37 38 00  00 a0 98 77  4f 6a 34 5a | ?Vkz D78. ...w Oj4Z
     050: 01 13 00 10  60 0a 09 80  00 00 00 02  c0 a8 00 8f | .... `... .... ....
     060: 00 00 0c bc  01 14 00 04  00 00 10 02  01 15 00 04 | .... .... .... ....
     070: 00 00 13 e9                                        | ....
    ```
*   **Root Cause:** The stale TPG identifiers (13e8 and 13e9) in the VPD page 83 data are the root cause. The Microsoft DSM, upon encountering these invalid identifiers during device enumeration after the reboot, failed to correctly process the device and did not claim it. This resulted in the LUN being inaccessible.
*   **DSM Behavior:** The Microsoft DSM uses the VPD page 83 information to build and maintain its internal representation of Target Port Groups and Target Ports.  The presence of incorrect TPG identifiers caused a conflict or inconsistency within the DSM's data structures, preventing it from correctly associating the presented device with the valid and active TPGs (5096/5097).

**5. Current Status:**

The resolution requires correcting the VPD page 83 data on the NetApp storage array.  The stale TPG identifiers (13e8 and 13e9) *must be removed*. The VPD data should *only* contain information about the currently active and valid TPGs (in this case, 5096/5097).

**Important Note:** The specific procedure for modifying VPD data on a NetApp ONTAP array is a NetApp-specific task and is outside the scope of this Windows-focused investigation.  This will be addressed in a separate ticket assigned to the NetApp administration team.

