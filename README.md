# Password Manager (command line)

Final project for ICS0022 Secure Programming, autumn 2026.

A password manager that is running in the terminal. All saved passwords are kept in one
encrypted vault file on the computer, opened with a master password. No server and no
internet connection.

Design and threat model: [`docs/checkpoint1.md`](docs/checkpoint1.md)

## Running it

Needs Python 3.11 or newer.

```
pip install -r requirements.txt
python main.py
```

The first run asks you to create a master password and makes the vault file in the
same folder. After unlocking, the menu is:

```
1. Add a new password
2. List saved entries
3. Show one entry
4. Delete an entry
5. Quit
```

## Progress

Checkpoint 1 (design and threat model) is done
