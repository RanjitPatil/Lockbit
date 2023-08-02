# LockBit-Malware-Analysis

LockBit ransomware is currently one of the most popular and active ransomware groups in the wild. This ransomware variant was first detected in September 2019 and used by Threat Actors to target multiple sectors and organizations worldwide. The TAs behind LockBit operate under the Ransomware-as-a-Service (RaaS) business model.

## Malware Sample

> **MD5:**   38745539b71cf201bb502437f891d799 

> **SHA256:**  80e8defa5377018b093b5b90de0f2957f7062144c83a09a56bba1fe4eda932ce 

## LockBit Execuation

- Using the packer identifier utility Detect It Easy, we found that this particular LockBit 3.0 sample is a Win32 .exe file with multiple sections packed with an unknown packer. 

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/1673c020-ecb3-48d2-b565-347964bcf98e)

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/1c974d94-5d05-4991-9653-adaa34f87ae1)

- According to the malware bazaars comments, the malware uses below  command line argument for decryption.

> ***`Command line argument - sample.exe -k LocalServiceNetworkRestricted -pass db66023ab2abcb9957fb01ed50cdfa6a`***

- Lockbit sample consists of below 4 steps.

    1). In the Unpack Sections phase, the actual sections are decrypted.

    2). The Reconstruct Import Address Table.

    3). During the Escalate Privilege phase, the sample attempts to evolve by granting itself higher privileges.

    4). Most functions are initiated as child threads, and operations corresponding to the decrypted configuration from the Escalate Privilege phase are executed.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/03c714dc-0f63-4f4d-b257-5b395b51aeb7)


## Packer and Dynamic Imports

- To activate the proper execution of the malware, a decryption key must be provided as a parameter (-pass) when the malicious file is launched.

- Lockbit is well packed, and its imports table is almost empty. Most API functions calls are resolved dynamically by calling a trampoline function which decrypts the API function
using XOR encryption.

- We can see the below snap of PEStudio, Import table is empty.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/a3bb1a99-39f0-4ee5-adbc-cec4f06c484c)

- Check the below snap of Packed Sample.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/b4d52909-f5ad-4247-93f1-95f74c512186)

- Below snap shows the unpacked code after passing the command line argument.***`sample.exe -k LocalServiceNetworkRestricted -pass db66023ab2abcb9957fb01ed50cdfa6a`***

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/a3fa75a8-eebb-44a6-8e5c-4f87b64d3462)

- Below funcation is responsible to perform the decryption of the sections of the binary, by retrieving the decryption key from the execution parameters and by passing it through.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/9c9c1712-c21e-4895-b4f1-f2b637afe95d)

- LockBit 3.0 performs API harvesting by hashing the API names of a DLL, and then comparing it to the list of the APIs that the ransomware needs.

- To make analysis difficult, LockBit 3.0 also uses string obfuscation, which is done through a simple decryption algorithm (XOR) to decrypt the strings.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/8101e039-cf4f-4ddd-b714-3631877075af)

- Below funcation is responsible to load and map the needed APIs into memory is analyzable only on the decrypted/unpacked version of the malware.

- The way APIs are resolved consists in a call to a funcation (sub_407C5C) that receives as input an obfuscated string that is XORed with the key 0x4506DFCA, so to decrypt the Win32 API name to
be resolved.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/b363ff4d-ebd6-4ebb-9957-a3a1d5bbcb40)

- Lockbit 3.0 can load all of the Windows APIs during the execution time, so in order to see the hidden API calls we can execute the sample and see the results under Debugger or we can use HashDB on IDA.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/8aa74032-3e2b-4c97-9de9-93cc22229e08)

## OS Version Check

- LockBit checkes the Windows NT version using below funcation.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/cf446fe0-bbb6-4834-a332-20dd2800e1b9)

## Mutex and Multiple Thread Creation

- After resolving API functions dynamically, it creates a mutex to ensure that only one instance of malware is running on the victim’s system at any given time.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/9a471d8a-56fc-412a-83c4-490413bb2895)

- The ransomware creates multiple threads using the CreateThread() API to perform several tasks in parallel for faster file encryption, as shown in Figure 6.

- Each thread is responsible for querying system information, getting drive details, ransom note creation, getting file attributes, deleting services, file search, encryption, etc.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/482ca985-ef1b-49ef-b66d-e58a368a2ed2)

## Service Deletion

- LockBit deletes a few services to encrypt the files successfully. To delete these services, the ransomware calls the OpenSCManagerA()API to get the service control manager database access.

- After gaining access, checks for the presence of services and deletes them if they are actively running on the victim’s machine. The below are some services  targeted by ransomware.

    | Services      | Services      | Services      | Services      |
    | ------------- | ------------- | ------------- | ------------- |
    | backup        | wordpad	      | winword	      | thebat	      |
    | GxBlr         | tbirdconfig	  | synctime	    | outlook	      |
    | GxCIMgr       | steam		      | sqbcoreservice| oracle	      |
    | GxCVD         | registry	    | powerpnt	    | ocssd		      |
    | GxFWD         | ocomm		      | ocautoupds	  | notepad	      |
    | GxVss         | mydesktopqos	| mspub		      | msaccess	    |
    | memtas        | isqlplussvc	  | infopath	    | firefox	      |
    | mepocs        | excel	        | encsvc		    | svc$          |
    | sophos        | dbsnmp		    | dbeng50	      | agntsvc	      |


## Registry Modification

- After deleting the services, the ransomware drops two files named **HLJkNskOq.ico** and **HLJkNskOq.bmp** in the **c:\programdata** location.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/f371f4de-9f7d-4c49-bb87-923c030a8f90)

- The ransomware creates a **Default** registry key for the extension **HLJkNskOq** shown in the figure below.This operation changes the icons of the encrypted files, which have the extension **HLJkNskOq**.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/363c73d0-6e89-4909-8cc7-b26ac82a2531)

## Killing Windows Defender and Tempering Windows Event Log

- LockBit changes the registry keys to disable all Windows Event Log Messages and kill the Microsoft Defender Process/Service.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/36c713f2-106a-4716-80a2-17c8643b0e1a)

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/c1eda179-2000-435b-871d-b64a540718ef)

- After the registry key change, Enabled key set to 0 and new Security Descriptor **(O:BAG:SYD:(A;;0x1;;;SY)(A;;0x5;;;BA)(A;;0x1;;;LA))** add it to temper the Event Logs.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/2fcda530-80a9-405c-b57c-390b6a2b1e18)

- LockBit clears the event log with **ClearEventLogW()** API function.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/d97be8c4-25ab-458d-9273-6aeab34471db)

## Dropping Ransomware Note

- Before initiating the encryption process, the ransomware drops the below ransom note in multiple folders with the file name **HLJkNskOq.README.txt**

- In the dropped ransom note, victims are instructed on how to pay the ransom to decrypt their encrypted files. 

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/6f0baa85-87c6-429e-ad80-8acdf08425ae)

## Encrypted Files

- The ransomware then encrypts the victim’s files, appends the extension **“.HLJkNskOq,”** and changes the file’s icon as shown below.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/ddb74ccb-9bad-40c8-a2de-227c46ae9929)

- Check the below funcations used for File Encryption.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/c42b4bf1-588a-4a26-86ef-676a621e91ac)

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/07fa4f36-8b80-4955-85cf-44d2b80fa061)

*(Image Credit- txone network blog)*


## Changing Desktop Background

- Finally, the ransomware changes the victim’s wallpaper leveraging the file **“HLJkNskOq.bmp”** using the **systemparametersinfoW()** API function.

![image](https://github.com/RanjitPatil/Lockbit/assets/43460691/01b0b9ef-283f-45c4-b6fd-62bd75ae2a2c)

## Yara Rule

Refrerance - [https://github.com/interprobe/lockbit3.0detect_v2-byInterProbe.yara/blob/main/lockbit3.0detect_v2.yara](https://github.com/interprobe/lockbit3.0detect_v2-byInterProbe.yara/blob/main/lockbit3.0detect_v2.yara)
```
rule Lockbit3Detect_via_SectionPatterns {
    meta:
        description = "Detects new Lockbit 3.0 variants"
        author = "InterProbe Malware-Vulnerability Research Team"
        date = "2022-07-04"
    strings:
        $lstr1 = ".xyz" wide ascii
        $lstr2 = ".rdata$zzzdbg" wide ascii
        $lstr3 = ".text$mn" wide ascii
        $entrypoint = { 90 0f 1f 84 00 00 00 00 00 e8 83 fb ff ff 0f 1f 40 00 e8 ce cd fe ff 66 90 e8 77 03 ff ff 0f 1f 44 00 00 e8 e1 da ff ff 0f 1f 84 00 00 00 00 00 6a 00 ff 15 c0 75 42 00 0f 1f 80 00 00 00 00 e8 49 f4 ff ff e8 26 f4 ff ff e8 45 f4 ff ff e8 22 f4 ff ff e8 11 f4 ff ff e8 36 f4 ff ff e8 25 f4 ff ff e8 32 f4 ff ff e8 15 f4 ff ff e8 fe f3 ff ff e8 05 f4 ff ff e8 12 f4 ff ff e8 fb f3 ff ff e8 ae f3 ff ff e8 c1 f3 ff ff e8 ce f3 ff ff e8 ab f3 ff ff e8 ca f3 ff ff e8 b9 f3 ff ff e8 b4 f3 ff ff e8 91 f3 ff ff e8 86 f3 ff ff e8 93 f3 ff ff e8 a6 f3 ff ff e8 95 f3 ff ff e8 6c f3 ff ff e8 09 df ff ff e8 ec de ff ff e8 f3 de ff ff e8 f4 de ff ff e8 e3 de ff ff 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 }
        $sectionitext = { 55 8b ec 81 ec 7c 03 00 00 53 56 57 8d 9d 84 fc ff ff b9 00 c2 eb 0b e2 fe e8 c6 02 00 00 53 50 e8 23 02 00 00 85 c0 74 79 53 8d 45 a0 50 e8 c1 02 00 00 8d 85 8c fe ff ff 50 8d 45 c0 50 8d 45 a0 50 e8 01 03 00 00 89 45 9c e8 85 02 00 00 8b d8 8b 5b 08 8b 73 3c 03 f3 0f b7 7e 06 8d b6 f8 00 00 00 6a 00 8d 06 50 e8 7f 00 00 00 3d 75 80 91 76 74 0e 3d 1b a4 04 00 74 07 3d 9b b4 84 0b 75 18 8b 4e 0c 03 cb ff 75 9c 8d 85 8c fe ff ff 50 ff 76 10 51 e8 82 03 00 00 83 c6 28 4f 85 ff 75 c1 5f 5e 5b 8b e5 5d c3 8d 40 00 55 8b ec 51 52 56 33 c0 8b 55 0c 8b 75 08 b9 61 00 00 00 66 ad 90 66 83 f8 41 72 0b 66 83 f8 5a 77 05 66 83 c8 20 90 02 f1 2a f1 8b c8 d3 ca 03 d0 90 85 c0 75 d8 8b c2 5e 5a 59 5d c2 08 00 90 55 8b ec 51 52 56 33 c0 8b 55 0c 8b 75 08 b9 61 00 00 00 ac }
    condition:
        ((uint16(0) == 0x5a4d) and (filesize < 200KB and filesize > 150KB)) and (all of them)
}
```

## References 

- [https://www.infinitumit.com.tr/wp-content/uploads/2023/01/lockbit.pdf](https://www.infinitumit.com.tr/wp-content/uploads/2023/01/lockbit.pdf)

- [https://www.txone.com/blog/malware-analysis-lockbit-3-0/](https://www.txone.com/blog/malware-analysis-lockbit-3-0/)

- [https://chuongdong.com/](https://chuongdong.com/)
