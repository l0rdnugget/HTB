oBfsC4t10n2

Category: Forensics  
Difficulty: Hard  
Date:  5/1/26

<img width="2264" height="168" alt="image" src="https://github.com/user-attachments/assets/af1326e5-9f41-485b-b1b4-6532b8662bc6" />  

<br>
Summary:
The goal of the challenge is to analyze a phishing document and figure out what it executes

<br>
<br>

Recon:
Downloaded and extracted the challenge file on PwnBox

```bash
cd ~
mkdir -p challenge1
wget "<HTB_DOWNLOAD_LINK>" -O challenge1.zip
unzip -P hackthebox challenge1.zip
```

This gave me `oBfsC4t10n2.xls` an older `.xls` format, which is common in phishing docs because it supports embedded macros more easily

<br>
Steps:
1. Opened the file in LibreOffice Calc

Nothing visible in the spreadsheet. No data, no hidden sheets. Saw "Enable Editing" and "Enable Content" prompts. Macro scripts??
<img width="1006" height="840" alt="image" src="https://github.com/user-attachments/assets/8f5fb554-f8c0-47ca-b842-1b9b1c63183e" />

2. Ran olevba against the file
Since there was nothing to see visually, I ran olevba to extract any hidden macros

```bash
olevba oBfsC4t10n2.xls
```

<img width="1242" height="710" alt="image" src="https://github.com/user-attachments/assets/148afeee-668a-46d6-b43c-87e83cfe416a" />

<br>
This pulled out a bunch of obfuscated Excel 4.0 XLM macros scattered across hidden cells. Key findings:

- Cell N545
- Cell N547
- Cells R1188-R1191
- Cell A1338

3. Piecing together the flag:
The flag was split across multiple cells and embedded inside the `ShellExecuteA` call in cell R1191. Three pieces.

<img width="1246" height="24" alt="image" src="https://github.com/user-attachments/assets/24cb1580-9363-4fb1-9d06-54fe8ffb7e20" />
<br>

Flag
`HTB{n0w_xxxxx_xxx_xxxxxx_x_xxxx}`

Lessons Learned:
- `.xls` files use Excel 4.0 XLM macros which hide formulas across individual cells rather than in a macro editor 
- olevba is the go to for pulling these out without executing anything
- Flags can be buried inside function arguments, not just in formula outputs

Tools Used:
- olevba: part of the oletools suite. Extracts VBA and XLM macros from office files, detects obfuscation, and flags suspicious behavior like file downloads, shell execution, or registry modifications.
