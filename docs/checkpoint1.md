# Checkpoint 1: Design and Threat Model

ICS0022 "Secure Programming", Autumn 2026

## 1. What I am developing

A terminal-based password manager. A single encrypted vault file is stored on the computer. Accessing it requires entering a master password, a simple menu then allows for saving and searching for passwords:

```
1. Add new password
2. List entries 
3. Show entry
4. Delete entry
5. Exit
```

I chose this option over a general file encryption project because the data volume is small and the structure is fixed (website, username, password, note). This allows all data to be encrypted at once, eliminating the need to manage large files or user-defined file paths.

Out of scope: no internet or server connection, one user per vault, terminal-only operation. I also do not provide protection against an attacker with administrator privileges on the machine, as they could read any program's memory.

## 2. Program structure

The program consists of six files; they are separated in such a way that the component writing data to disk never "sees" the plaintext password.

| File | Function |
| --- | --- |
| `main.py` | Program startup loop and menu processing. |
| `auth.py` | **User management:** Prompts for a master password and unlocks the vault. |
| `crypto.py` | **Encryption:** Generates a key from the master password; handles encryption and decryption. |
| `storage.py` | **Storage:** Reads from and writes to the vault file. Works only with encrypted data. |
| `entries.py` | Stores entries in RAM while the program is running. |
| `logger.py` | Writes events to a log file. |



The idea is for `storage.py` to be the only file that interacts with the disk,
and for the data to be already encrypted when written to it. Therefore, even if I
later make a mistake in the file-handling code, I won't be able to accidentally write the password to the disk
in plain text.

A vault file consists of three concatenated parts:

```
[salt: 16 bytes][nonce: 12 bytes][encrypted data: remainder]
```

The salt and nonce are not secret data, but they must be stored;
otherwise, the file will be impossible to decrypt. The encrypted part is a JSON list
of entries; each entry contains a title, username, password, and a note.

Since the entire list is encrypted as a single block, an external observer examining the file
would not even be able to determine the number of entries it contains.

Step-by-step process:

1. `auth.py` prompts for the master password (characters are not displayed while typing).

2. `storage.py` reads the file and takes the first 16 bytes as the salt.

3. `crypto.py` applies the Argon2id algorithm to the combination of password and salt to derive a 256-bit key, and then attempts to decrypt the remaining data. 

4. If it fails (incorrect password or file modification), the program notifies the user and terminates.

5. If successful, the entries are loaded into memory and a menu appears.

6. With every change, the entire list is re-encrypted using a **new** nonce and saved.

## 3. What exactly am I protecting and from whom?

First: the master password; then, the key generated from it; and finally, the stored passwords.

* **Someone who gains access to the vault file** - for example, by accessing a stolen laptop, an old backup, or a folder accidentally synced to cloud storage. Such an attacker would have unlimited time to attempt to crack it from home. 

* **Someone using the same computer** - a public computer or a program running under my user account
that can read my files and view running processes.

* **Someone standing behind me** - reading information off my screen or using my computer while
a program is open.

* **Invalid input** - a corrupted vault file or excessively long text entered into a field.

Not considered: an administrator or a virus with full privileges (see Section 1).

## 4. Potential problems and solutions

| # | Area | Problem | Solution |
| --- | --- | --- | --- |
| 1 | Master password | The password is too short or obvious - it can be guessed. | Require a minimum length when creating the vault - warn if the password is too short. |
| 2 | Master password | An attacker who obtains the file attempts to crack the password via brute force. | Use Argon2id with high memory requirements: each attempt consumes time and resources. Salt renders precomputed tables useless. |
| 3 | Master password | The password is displayed on screen or saved in the terminal history. | Input via `getpass` (without echoing characters). It is never displayed on screen or written to the log. |
| 4 | Master password | The password is passed as a startup parameter and is visible to other users in the process list. | The program requests the password only via interactive input. Passing a password via arguments is not possible. |
| 5 | Vault file | Someone opens the file and reads the passwords. | The entire vault is encrypted with AES-256-GCM before saving. |
| 6 | Vault file | Someone modifies bytes in the file to corrupt it or deceive the program. | GCM verifies data integrity. If even a single byte is changed, decryption will fail and the program will terminate. |
| 7 | Vault file | The same nonce is used twice with the same key, compromising GCM. | A new random nonce is generated every time the file is saved. |
| 8 | Vault file | Another user on the same computer reads the file. | The file is created with read and write permissions restricted to the owner. |
| 9 | Vault file | The program fails during saving, and the vault becomes corrupted. | Data is first written to a temporary file and then renamed to overwrite the previous one. |
| 10 | Memory | Passwords and the key remain in memory while the program is running. | Store data only for as long as necessary and avoid creating unnecessary copies. |
| 11 | Terminal | Someone is reading data from the screen or using the program in my absence. | Display the password only when specific input is requested, not in the general list. |
| 12 | Input | A corrupted file or excessively large input causes the program to crash with a Python error. | Limit input length, catch errors during file reading, and print a clear message instead of a stack trace. |
| 13 | Log file | Someone is deleting lines from the log to hide their presence. | I cannot prevent this locally. Only the owner has access to the log, and I treat it as a source of information, not as evidence. |

## 5. Selected cryptographic solutions

**Argon2id** is used to convert the master password into a key. The master password is short and human-readable, so it cannot serve as a key directly. The Argon2id algorithm intentionally slows down the conversion process and consumes significant memory; slowness alone is insufficient - since graphics cards can perform thousands of slow operations in parallel - but the requirement for substantial memory usage per attempt makes such an attack costly. The salt consists of 16 random bytes, generated once when the vault is created.

**AES-256-GCM** is used to encrypt the vault. I initially considered CBC mode, as I had learned it in a cryptography course, but CBC only hides the content and makes it impossible to determine whether changes have been made. This is insufficient for a password manager. GCM mode provides both encryption and integrity verification, so I do not need to implement a separate check and risk making a mistake.

**Important decision:** I do not store the master password hash. There is no data in the file indicating "this is the correct password"; correctness is confirmed only through successful decryption. Consequently, the file lacks a standalone element that could be targeted separately.

I do not write my own cryptographic code - instead, I use existing libraries.

## 6. Python memory management issue

This is the aspect I am least familiar with, so I will only describe what I know.

After decryption, the passwords and the key end up in the running program's RAM, and I cannot fully control this in Python. Strings (`str`) are immutable once created, so I cannot overwrite them with zeros, and the `del x` command only deletes the variable name without guaranteeing the destruction of the actual data.

For now, I plan to store decrypted data only while the program is running, avoid copying passwords between variables, and store the key in a `bytearray` object (rather than a string), since the contents of a `bytearray` *can* be overwritten directly. I will then test whether this works in practice.

I do not consider this issue resolved, and I will describe the solution that worked in the final report. I acknowledge that this is a limitation of Python: in C, I could overwrite the buffer directly, but then I would have to worry about buffer overflows.

## 7. Language and libraries

Python: because it is the language I know best, and because memory errors such as buffer overflows are not possible in it. The downside is the issue described in Section 6.

| Library | Purpose |
| --- | --- |
| `argon2-cffi` | Argon2id, key generation from the master password |
| `cryptography` | AES-256-GCM, vault encryption |
| `getpass` (built-in) | Input master password without displaying characters |
| `json`, `os`, `secrets` (built-in) | Data formatting, file handling, random salt and nonce generation | 

## 8. Plan for the next checkpoints

* **Checkpoint 2:** Create a vault, unlock it, add entries, and list them. Implement Argon2id and AES-GCM, and maintain a log file. 
* **Checkpoint 3:** View and delete entries, manage file access permissions, save securely using a temporary file, fix error messages, and conduct security audits using specialized tools.