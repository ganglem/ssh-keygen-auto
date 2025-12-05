#!/bin/bash

# Configuration
KEY_ALGORITHM="ed25519"
KEY_COMMENT="$(whoami)@$(hostname)"
OUTPUT_DIR="."
SSH_CONFIG_FILE="$HOME/.ssh/config"
DEFAULT_SSH_USER="$USER"

# Initialize argument flags and storage
PROMPT_PASSPHRASE=false
KEY_NAMES=()
COLLECTING_NAMES=false

# Function to display usage instructions
usage() {
    echo "Usage: $0 -n <key_name_1> [key_name_2] [...] [-p]"
    echo ""
    echo "This script generates SSH key pairs ($KEY_ALGORITHM) and updates the SSH config file:"
    echo "    $SSH_CONFIG_FILE"
    echo ""
    echo "Arguments:"
    echo "  -n        : MANDATORY. Signals the start of the list of key names."
    echo "  <key_name>: One or more names for the keys (e.g., github, server_a)."
    echo "  -p        : OPTIONAL. Prompts for a passphrase for all generated keys."
    echo ""
    echo "Example: $0 -n github_personal prod_eu_access"
    echo "Example: $0 -n test_server_key another_key -p"
    exit 1
}

# --- Argument Parsing ---
for arg in "$@"; do
    case "$arg" in
        "-p")
            PROMPT_PASSPHRASE=true
            COLLECTING_NAMES=false
            ;;
        "-n")
            COLLECTING_NAMES=true
            ;;
        *)
            if [ "$COLLECTING_NAMES" = true ]; then
                KEY_NAMES+=("$arg")
            else
                echo "Error: Key names must be preceded by the '-n' flag."
                usage
            fi
            ;;
    esac
done

if [ ${#KEY_NAMES[@]} -eq 0 ]; then
    echo "Error: You must provide at least one key name after the '-n' flag."
    usage
fi

# Determine the passphrase to use
PASSPHRASE=""
PASSPHRASE_STATUS="None"
if [ "$PROMPT_PASSPHRASE" = true ]; then
    echo -n "Enter Passphrase for all keys: "
    read -r -s PASSPHRASE
    echo
    echo -n "Confirm Passphrase: "
    read -r -s PASSPHRASE_CONFIRM
    echo

    if [ "$PASSPHRASE" != "$PASSPHRASE_CONFIRM" ]; then
        echo "Error: Passphrases do not match. Aborting."
        exit 1
    fi

    if [ -z "$PASSPHRASE" ]; then
        PASSPHRASE_STATUS="Empty (But requested via -p)"
    else
        PASSPHRASE_STATUS="Set"
    fi
fi


# Function to append the configuration block to ~/.ssh/config
append_config_block() {
    local host_alias="$1"
    local identity_path_rel="~/.ssh/$host_alias"
    local config_file="$2"

    if grep -q "Host $host_alias" "$config_file" 2> /dev/null; then
        echo "  -> Note: Configuration block for Host '$host_alias' already exists in $config_file. Skipping config update."
        return 0
    fi
    
    echo "  -> Appending config block to $config_file..."

    local config_block
    config_block=$(cat <<EOF

Host $host_alias
    HostName CHANGEME
    User CHANGEME
    IdentityFile $identity_path_rel
    PreferredAuthentications publickey
    IdentitiesOnly yes
    AddKeysToAgent yes
EOF
    )

    echo "$config_block" >> "$config_file"
    
    if [ $? -eq 0 ]; then
        echo "  -> Successfully added config block for Host '$host_alias'."
    else
        echo "  -> Failure: Could not write config block to $config_file."
    fi
}

# Create the .ssh directory and config file if they don't exist
mkdir -p "$HOME/.ssh"
if [ ! -f "$SSH_CONFIG_FILE" ]; then
    touch "$SSH_CONFIG_FILE"
    chmod 600 "$SSH_CONFIG_FILE"
    echo "Created new config file: $SSH_CONFIG_FILE"
fi

echo "--- Starting SSH Key Generation ---"
echo "Algorithm: $KEY_ALGORITHM"
echo "Comment:   \"$KEY_COMMENT\""
echo "Passphrase Status: $PASSPHRASE_STATUS"
echo "Output Path: Current Directory ($OUTPUT_DIR)"
echo "Config File: $SSH_CONFIG_FILE"
echo "-----------------------------------"

# Check for SSH Agent status before the main loop
if [[ -z "$SSH_AUTH_SOCK" ]]; then
    echo "Warning: SSH agent (ssh-agent) is not running in this session."
    echo "Keys will be generated but CANNOT be added automatically."
    echo "If you need to start the agent, run: eval \$(ssh-agent -s)"
    AGENT_RUNNING=false
else
    echo "SSH Agent found. Keys will be added automatically."
    AGENT_RUNNING=true
fi
echo "-----------------------------------"

# Loop through all key names
for KEY_NAME in "${KEY_NAMES[@]}"; do
    FILEPATH="$OUTPUT_DIR/$KEY_NAME"

    echo "Processing key: $KEY_NAME"

    GENERATION_SUCCESS=false

    # 1. Check if files already exist
    if [ -f "$FILEPATH" ]; then
        echo "  -> Warning: Private key '$FILEPATH' already exists. Skipping generation."
        
        if [ "$AGENT_RUNNING" = true ]; then
            echo "  -> Attempting to add existing key to agent..."
            ssh-add "$FILEPATH" > /dev/null 2>&1
            if [ $? -eq 0 ]; then
                echo "  -> Successfully added existing key to agent."
            else
                echo "  -> Could not add existing key. It may be loaded or require a passphrase."
            fi
        fi
        GENERATION_SUCCESS=true
    elif [ -f "${FILEPATH}.pub" ]; then
        echo "  -> Warning: Public key file '${FILEPATH}.pub' exists, but private key is missing. Skipping."
    else
        # 2. Key Generation
        echo "  -> Generating new key pair (ssh-keygen)..."
        ssh-keygen -t "$KEY_ALGORITHM" -f "$FILEPATH" -N "$PASSPHRASE" -C "$KEY_COMMENT" > /dev/null 2>&1

        if [ $? -eq 0 ]; then
            echo "  -> Success: Created $FILEPATH and ${FILEPATH}.pub"
            GENERATION_SUCCESS=true

            # 3. Add to Agent
            if [ "$AGENT_RUNNING" = true ]; then
                echo "  -> Adding key to SSH agent (ssh-add)..."
                if [ "$PROMPT_PASSPHRASE" = true ] && [ -n "$PASSPHRASE" ]; then
                    printf '%s\n' "$PASSPHRASE" | ssh-add "$FILEPATH" > /dev/null 2>&1
                else
                    ssh-add "$FILEPATH" > /dev/null 2>&1
                fi
                
                if [ $? -eq 0 ]; then
                    echo "  -> Successfully added to agent."
                else
                    echo "  -> Failure adding key to agent. Check 'ssh-add -l' status manually."
                fi
            fi
        else
            echo "  -> Failure: Could not generate key for $KEY_NAME."
        fi
    fi
    
    # 4. Update Config File
    if [ "$GENERATION_SUCCESS" = true ]; then
        append_config_block "$KEY_NAME" "$SSH_CONFIG_FILE"
    fi
    
    echo "-----------------------------------"
done

echo ""
echo "--- Final Status ---"
echo "Total keys requested: ${#KEY_NAMES[@]}"
echo "Keys are located in the current directory."
echo "Your config file ($SSH_CONFIG_FILE) has been updated."
echo "Remember to REPLACE 'HostName CHANGEME' and 'User CHANGEME' with the actual server details."
echo "Use 'ssh-add -l' to check the keys loaded in your agent."
