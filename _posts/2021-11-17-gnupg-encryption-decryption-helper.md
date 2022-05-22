---
title: 'GnuPG encryption/decryption helper'
date: '2021-11-17T18:13:00+00:00'
author: deaves
layout: post
permalink: /2021/11/gnupg-encryption-decryption-helper/
categories:
    - Linux
    - 'Linux Admin'
    - 'Linux Scripts'
    - 'Linux Security'
tags:
    - encryption
    - files
    - GPG
    - privacy
    - security
---

I have had many ruminations on the best way to encrypt and secure files. A good reason for wanting to encrypt your files might be; you keep a USB stick on your keychain with your taxes or customer data on it so its accesable to you, but you’re are worried about loosing it. In situations like that, file encryption is exactly what you want! I previously have written a [scripts](https://www.davideaves.com/2010/11/handle-block-devices-that-use-the-device-mapper-driver/) and jotted [notes](https://www.davideaves.com/2014/08/luks-quick-examples/) on encrypting block devices; of which I am not even convinced is best practice. Encrypting block devices is all well and good but it’s not very practical for encrypting individual files. I recently noticed that Cisco uses [GPG](https://gnupg.org) to encrypt their backup files on their ISE appliance, which convinced me that GPG is probabbly the smartest way to encrypt files. Unfortunately there is a lot of commandline options and reading you have to do inorder to get it to work properly. I figured this was perfect thing to write a script for and make public. My goal is to make GnuPG a little bit more accesable to a broader audience.

The following script is intended to make it simple for a lay Linux user to use GPG to encrypt/decrypt files and directories. As a bonus I’m keeping the debug block that I used, and can be removed, in order to test encryption and decryption while I was writting the script and testing commands. If you do not already have a public-private key pair, the script will generate one for you and ask you if you want to do a backup. If you allow the script to create backup files of your keys I strongly recommend you remove them as soon as possible! The script can also perform a restore of your keys from backup if you put them back in the root of your home directory. Additionally the script doesn’t see any keys it will perform symmetric encryption against the follow directory. Symmetric encryption just uses a simple passphrase to perform the encryption and decryption. If you create the following variable:

```bash
export gpg_method="symmetric"
```

With the above variable set; the script will ignore your keys and will do symmetric only encryption.

#### **gpg\_crypt.sh**: *a /bin/env bash script, ASCII text executable, with very long lines (382)*

```bash
#!/bin/env -S bash
## GnuPG encryption/decryption helper.
## 2021 - Script from www.davideaves.com

export GPG_TTY=$(tty)

# Verify script requirements
for req in curl gpg jq; do
    type ${req} >/dev/null 2>&1 || {
        echo >&2 "$(basename "${0}"): I require ${req} but it's not installed. Aborting."
        exit 1
    }
done && umask 0077

# Fetch existing keys
keyid=( `gpg --list-keys --keyid-format 0xLONG | awk '/^sub.*[E]/{gsub("[]|[]|/", " "); print $3,$NF}'` )

# Help manage keys
if [ -z "${keyid}" ] && [ -z "${gpg_method}" ]; then
    [ -f "${HOME}/bin/rc_files/gpg.conf" -a ! -f "${HOME}/.gnupg/gpg.conf" ] && {
        cat "${HOME}/bin/rc_files/gpg.conf" > "${HOME}/.gnupg/gpg.conf"
    }

    # Generate a new keys
    read -p "(G)enerate new or (R)estore keys? (g/r) " -n 1 -r; echo
        if [[ "${REPLY}" =~ ^[Gg]$ ]]; then
            gpg --full-generate-key || gpg --gen-key

            # Backup keys
            read -p "Export a backup of keys? " -n 1 -r; echo
            if [[ "${REPLY}" =~ ^[Yy]$ ]]; then
                gpg --armor --export-secret-key > "${HOME}/gpg_secret-key.asc"
                gpg --armor --export-secret-subkeys > "${HOME}/gpg_secret-subkeys.asc"
                gpg --armor --export > "${HOME}/gpg_public-key.asc"
                gpg --armor --export-ownertrust > "${HOME}/gpg_ownertrust.txt"
            fi

        # Restore existing keys
        elif [[ "${REPLY}" =~ ^[Rr]$ ]]; then
            for asc in ${HOME}/gpg_*.asc; do
                gpg --import "${asc}"
            done && gpg --import-ownertrust "${HOME}/gpg_ownertrust.txt"
        fi && unset ${REPLY}

elif [ -n "${gpg_method}" ]; then
    unset keyid
fi

# Exit if no user input or not debugging.
if [[ ! "$SHELLOPTS" =~ "xtrace" ]] && [[ -z "${@}" ]]; then
    echo "$(basename "${0}"): GnuPG encryption/decryption helper."
    echo "File or Directory input is required to continue!"
    exit 0
fi

# Create debug test file
if [[ "$SHELLOPTS" =~ "xtrace" ]] && [[ -z "${@}" ]]; then
    debug_b64="/9j/4AAQSkZJRgABAQAAZABkAAD/2wCEABQQEBkSGScXFycyJh8mMi4mJiYmLj41NTU1NT5EQUFBQUFBREREREREREREREREREREREREREREREREREREREQBFRkZIBwgJhgYJjYmICY2RDYrKzZERERCNUJERERERERERERERERERERERERERERERERERERERERERERERERERP/AABEIAAEAAQMBIgACEQEDEQH/xABMAAEBAAAAAAAAAAAAAAAAAAAABQEBAQAAAAAAAAAAAAAAAAAABQYQAQAAAAAAAAAAAAAAAAAAAAARAQAAAAAAAAAAAAAAAAAAAAD/2gAMAwEAAhEDEQA/AJQA9Yv/2Q=="

    # Create temp file
    if debug_file="$(mktemp)"; then
        trap "{ if [ -e "${debug_file}*" ]; then rm -rf "${debug_file}*"; fi }" SIGINT SIGTERM ERR EXIT
    else
        echo "Failure, exit status: ${?}"
        exit ${?}
    fi && echo -n "${debug_b64}" | base64 -d > "${debug_file}"

    MimeDB=( "https://raw.githubusercontent.com/jshttp/mime-db/master/db.json" "${HOME}/.cache/mime.json" )
    [ ! -s "${MimeDB[1]}" ] && {
        curl -s "${MimeDB[0]}" | jq > "${MimeDB[1]}"
    }

    debug_mime="$(file -b --mime-type "${debug_file}")"
    debug_hash="$(sha1sum "${debug_temp}" | awk '{print $1}')"
    debug_ext="$(jq -r --arg MIME "${debug_mime}" '.[$MIME].extensions[0] // empty' "${MimeDB[1]}")"
    debug_output="${debug_hash}.${debug_ext}"

    mv "${debug_file}" "${debug_file}.${debug_ext}"
    set -- "$@" "${debug_file}.${debug_ext}"
    file "${debug_file}.${debug_ext}"
fi

### BEGIN ###
for input in "${@}"; do

    file_ext="$(basename "${input##*.}")"

    ## symmetric encryption: file ##
    if [ -z "${keyid[0]}" ] && [ -f "${input}" ]; then
        if [ "${file_ext}" != "gpg" ]; then
            gpg --batch --yes --quiet --output "$(basename "${input:-'null'}").gpg" --symmetric "${input}"
        ## decrypt file ##
        elif [[ "${input%.*}" =~ ".tgz"$ ]]; then
            gpg --batch --yes --quiet --decrypt "${input}" | tar xzfv -
        else
            gpg --batch --yes --quiet --output "$(basename "${input%.*}")" --decrypt "${input}"
        fi

    ## symmetric encryption: directory ##
    elif [ -z "${keyid[0]}" ] && [ -d "${input}" ]; then
        tar czfv - "${input}" | gpg --batch --yes --quiet --output "$(basename "${input:-'null'}").tgz.gpg" --symmetric

    ## key encryption: file ##
    elif [ -n "${keyid[0]}" ] && [ -f "${input}" ]; then
        if [ "${file_ext}" != "gpg" ]; then
            gpg --batch --yes --quiet --output "$(basename "${input:-'null'}").gpg" --recipient ${keyid[0]} --encrypt "${input}"

        ## decrypt file ##
        elif [[ "${input%.*}" =~ ".tgz"$ ]]; then
            gpg --batch --yes --quiet --recipient ${keyid[0]} --decrypt "${input}" | tar xzfv -
        else
            gpg --batch --yes --quiet --output "$(basename "${input%.*}")" --recipient ${keyid[0]} --decrypt "${input}"
        fi

    ## key encryption: directory ##
    elif [ -n "${keyid[0]}" ] && [ -d "${input}" ]; then
        tar czfv - "${input}" | gpg --batch --yes --quiet --recipient ${keyid[0]} --encrypt > "$(basename "${input:-'null'}").tgz.gpg"
    fi

done
### FINISH ###
```

## GnuPG user configuration options

If you use GnuPG I recommend updating your configuration to give it an affinity for stronger ciphers. [Riseup.net](https://help.riseup.net/en/security/message-security/openpgp/best-practices) created a very good best practice config that is a good starting place. I’ve made a few modifications to it, but have left it relatively unchanged. If the above script sees the user config missing and can find the following config file in a repo direcory, it will go ahead and copy the configuration to where it needs to be… Feel free to use the below config as an optional reference.

#### **gpg.conf**: *ASCII text*

```conf
## GnuPG Options

# Assume that command line arguments are given as UTF8 strings.
utf8-strings

#
# This is an implementation of the Riseup OpenPGP Best Practices
# https://help.riseup.net/en/security/message-security/openpgp/best-practices
#

#-----------------------------
# default key
#-----------------------------

# The default key to sign with. If this option is not used, the default key is the first key found in the secret keyring
#default-key 0xD8692123C4065DEA5E0F3AB5249B39D24F25E3B6

#-----------------------------
# behavior
#-----------------------------

# Disable inclusion of the version string in ASCII armored output
no-emit-version

# Disable comment string in clear text signatures and ASCII armored messages
no-comments

# Display long key IDs
keyid-format 0xlong

# List all keys (or the specified ones) along with their fingerprints
with-fingerprint

# Display the calculated validity of user IDs during key listings
list-options show-uid-validity
verify-options show-uid-validity

# Try to use the GnuPG-Agent. With this option, GnuPG first tries to connect to the agent before it asks for a passphrase.
use-agent

#-----------------------------
# algorithm and ciphers
#-----------------------------

# list of personal digest preferences. When multiple digests are supported by all recipients, choose the strongest one
personal-cipher-preferences AES256 AES192 AES CAST5

# list of personal digest preferences. When multiple ciphers are supported by all recipients, choose the strongest one
personal-digest-preferences SHA512 SHA384 SHA256 SHA224

# message digest algorithm used when signing a key
cert-digest-algo SHA512

# This preference list is used for new keys and becomes the default for "setpref" in the edit menu
default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed

# Use a specified algorithm as the symmetric cipher
cipher-algo AES256
```
