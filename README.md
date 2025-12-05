# SSH Key Generator Script

A bash script to streamline the generation of multiple SSH key pairs and automatically update your SSH config file.

## Features

- Generates multiple SSH key pairs using Ed25519 algorithm
- Automatically updates `~/.ssh/config` with host configuration blocks
- Supports optional passphrases for all generated keys
- Adds keys to SSH agent automatically (if running)
- Handles existing keys gracefully
- User-friendly status messages and warnings

## Prerequisites

- Bash shell
- `ssh-keygen` utility (usually included with OpenSSH)
- `ssh-agent` running (optional, but recommended for automatic key loading)

## Installation

1. Download the script or create a file with the script contents
2. Make it executable:
   ```bash
   chmod +x script_name.sh
   ```

## Usage

### Basic Syntax

```bash
./script_name.sh -n <key_name_1> [key_name_2] [...] [-p]
```

### Arguments

- `-n` (MANDATORY): Signals the start of the list of key names
- `<key_name>`: One or more names for the keys (e.g., `github`, `server_a`)
- `-p` (OPTIONAL): Prompts for a passphrase to protect all generated keys

### Examples

Generate two keys without passphrases:
```bash
./script_name.sh -n github_personal prod_eu_access
```

Generate a single key with a passphrase:
```bash
./script_name.sh -n test_server_key -p
```

Generate multiple keys with passphrases:
```bash
./script_name.sh -n github gitlab server_a -p
```

## What the Script Does

1. **Parses arguments** to collect key names and options
2. **Prompts for passphrase** (if `-p` flag is used) with confirmation
3. **Creates `~/.ssh` directory** if it doesn't exist
4. **Initializes SSH config file** if missing
5. **Generates key pairs** using Ed25519 algorithm
6. **Adds keys to SSH agent** automatically (if agent is running)
7. **Updates SSH config** with new host configuration blocks

## SSH Config Updates

The script automatically appends configuration blocks to your `~/.ssh/config` file in the following format:

```
Host <key_name>
    HostName CHANGEME
    User CHANGEME
    IdentityFile ~/.ssh/<key_name>
    PreferredAuthentications publickey
    IdentitiesOnly yes
    AddKeysToAgent yes
```

**Important:** You must manually replace `CHANGEME` values with your actual server hostname and username.

## Output Files

Generated keys are created in the current directory:
- `<key_name>` (private key)
- `<key_name>.pub` (public key)

It's recommended to move these to `~/.ssh/` after generation.

## Checking Your Keys

After running the script, verify your keys are loaded in the SSH agent:
```bash
ssh-add -l
```

## SSH Agent Note

If SSH agent is not running, you'll see a warning. To start it, run:
```bash
eval $(ssh-agent -s)
```

## Permissions

- `.ssh` directory: `700` (created automatically)
- `.ssh/config` file: `600` (created automatically)
- Private keys: `600` (created by ssh-keygen)
- Public keys: `644` (created by ssh-keygen)

## Troubleshooting

- **Keys not added to agent**: Make sure SSH agent is running or check the passphrase
- **Config already exists**: If a `Host` entry already exists, the script skips the update to avoid duplicates
- **Permission denied**: Ensure the script is executable (`chmod +x`) and you have write permissions in the output directory
- **Script won't run**: Verify bash is available and the file isn't corrupted (check for proper line endings)
