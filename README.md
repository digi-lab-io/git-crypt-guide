# GPG Key Management Guide

A comprehensive guide for creating, managing, and using GPG keys for encryption, signing, and authentication.

## Table of Contents

- [What is GPG?](#what-is-gpg)
- [Installation](#installation)
- [Creating GPG Keys](#creating-gpg-keys)
- [Managing Existing Keys](#managing-existing-keys)
- [Key Operations](#key-operations)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Additional Resources](#additional-resources)

## What is GPG?

GNU Privacy Guard (GPG) is a free implementation of the OpenPGP standard that allows you to encrypt and sign your data and communications. GPG is widely used for:

- 🔐 **File encryption** and decryption
- ✍️ **Digital signatures** for authenticity
- 🔑 **Key management** for secure communications
- 🛡️ **Git commit signing** for code integrity

## Installation

### Windows

- **GPG4Win**: <https://www.gpg4win.org/>
- **Chocolatey**: `choco install gpg4win`

### Linux

```bash
# Debian/Ubuntu
sudo apt-get install gnupg

# RHEL/Fedora
sudo dnf install gnupg2

# Arch Linux
sudo pacman -S gnupg
```

### macOS

```bash
# Homebrew
brew install gnupg

# MacPorts
sudo port install gnupg2
```

## Creating GPG Keys

> **⚠️ Important: One-Time Setup**  
> Creating a GPG key is a **one-time configuration** that establishes your cryptographic identity. Once created, you'll use this same key pair for all your encryption, signing, and authentication needs. Choose your settings carefully as changing them later requires generating a new key and redistributing your public key to all contacts and services.

### 🚀 Quick Key Generation (Interactive)

The easiest way to create a GPG key is using the interactive mode:

```bash
gpg --full-generate-key
```

Follow the prompts to:

1. Select key type (choose RSA and RSA)
2. Choose key size (4096 bits recommended)
3. Set expiration date (recommended: 2 years)
4. Enter your name and email address
5. Set a strong passphrase

### 🤖 Automated Key Generation (Batch Mode)

For automated setups or consistent configurations:

```bash
gpg --batch --gen-key <<EOF
Key-Type: RSA
Key-Length: 4096
Subkey-Type: RSA
Subkey-Length: 4096
Name-Real: Your Name
Name-Email: your.email@domain.com
Expire-Date: 2y
Key-Usage: sign
Subkey-Usage: encrypt
Passphrase: your-secure-passphrase
%commit
EOF
```

### 🔒 High-Security Key Generation

For maximum security (no passphrase protection - use with caution):

```bash
gpg --batch --gen-key <<EOF
Key-Type: RSA
Key-Length: 4096
Subkey-Type: RSA
Subkey-Length: 4096
Name-Real: Your Name
Name-Email: your.email@domain.com
Expire-Date: 0
Key-Usage: sign
Subkey-Usage: encrypt
%no-protection
%commit
EOF
```

### ✅ Verify Key Creation

After creating your key, verify it was generated successfully:

```bash
# List all keys
gpg --list-keys

# List secret keys
gpg --list-secret-keys

# Show key details
gpg --list-keys --keyid-format LONG your.email@domain.com
```

## Managing Existing Keys

### 📥 Importing Keys

#### From WSL to Windows

If you have GPG keys on WSL and want to use them on Windows:

1. **Export from WSL**:

   ```bash
   # Export public key
   gpg --armor --export your.email@domain.com > your-public-key.asc
   
   # Export private key
   gpg --armor --export-secret-keys your.email@domain.com > your-private-key.asc
   ```

2. **Copy to Windows**: Access WSL files at `\\wsl$\<distro>\home\<username>\`

3. **Import on Windows**:

   ```bash
   # Import private key (includes public key)
   gpg --import your-private-key.asc
   
   # Verify import
   gpg --list-keys
   gpg --list-secret-keys
   ```

#### From Files

```bash
# Import public key
gpg --import public-key.asc

# Import private key
gpg --import private-key.asc

# Import from keyserver
gpg --keyserver hkps://keys.openpgp.org --recv-keys KEYID
```

#### Trust Imported Keys

After importing, you may need to set trust levels:

```bash
gpg --edit-key your.email@domain.com
# In GPG prompt:
# gpg> trust
# Select trust level (5 for ultimate trust)
# gpg> quit
```

## Key Operations

### 📤 Exporting Keys

#### Public Key Export

```bash
# ASCII armored format (recommended for sharing)
gpg --armor --export your.email@domain.com > your-public-key.asc

# Binary format
gpg --export your.email@domain.com > your-public-key.gpg

# To specific directory (e.g., for git-crypt)
gpg --armor --export your.email@domain.com > .keys/your-public-key.asc
```

#### Private Key Export (Backup)

```bash
# ASCII armored format
gpg --armor --export-secret-keys your.email@domain.com > your-private-key.asc

# To secure backup location
gpg --armor --export-secret-keys your.email@domain.com > /secure/backup/your-private-key.asc
```

⚠️ **Security Warning**: Keep private key exports in secure, encrypted storage!

### 🗑️ Removing Keys

#### Interactive Removal

```bash
# Remove private key first (if removing both)
gpg --delete-secret-key your.email@domain.com

# Remove public key
gpg --delete-key your.email@domain.com
```

#### Automated Removal Script

Create a script for batch removal:

```bash
#!/bin/bash

# Configuration
EMAIL="${1:-your.email@domain.com}"

echo "Removing GPG keys for: $EMAIL"

# Remove private key first
if gpg --list-secret-keys "$EMAIL" >/dev/null 2>&1; then
    echo "Removing private key..."
    gpg --batch --yes --delete-secret-key "$EMAIL"
else
    echo "No private key found for $EMAIL"
fi

# Remove public key
if gpg --list-keys "$EMAIL" >/dev/null 2>&1; then
    echo "Removing public key..."
    gpg --batch --yes --delete-key "$EMAIL"
else
    echo "No public key found for $EMAIL"
fi

echo "GPG key removal completed for $EMAIL"
```

Save as `remove-gpg-keys.sh`, make executable, and run:

```bash
chmod +x remove-gpg-keys.sh
./remove-gpg-keys.sh your.email@domain.com
```

### 🔍 Key Information

```bash
# List all keys
gpg --list-keys

# List secret keys only
gpg --list-secret-keys

# Show key fingerprints
gpg --fingerprint

# Show key details with long format
gpg --list-keys --keyid-format LONG

# Show specific key information
gpg --list-keys your.email@domain.com
```

### 🔐 Encryption & Signing

```bash
# Encrypt a file
gpg --encrypt --armor --recipient your.email@domain.com file.txt

# Sign a file
gpg --sign --armor file.txt

# Sign and encrypt
gpg --sign --encrypt --armor --recipient your.email@domain.com file.txt

# Decrypt a file
gpg --decrypt file.txt.asc

# Verify a signature
gpg --verify file.txt.asc
```

## Best Practices

### 🔒 Security Recommendations

1. **Key Strength**:
   - Use 4096-bit RSA keys minimum
   - Set expiration dates (1-2 years recommended)
   - Use strong passphrases

2. **Key Management**:
   - Backup private keys securely
   - Store backups in multiple secure locations
   - Regularly update and rotate keys

3. **Operational Security**:
   - Never share private keys
   - Use separate keys for different purposes
   - Revoke compromised keys immediately

### 📋 Key Naming Conventions

```bash
# Descriptive naming for multiple keys
Name-Real: John Doe (Work)
Name-Email: john.doe@company.com

Name-Real: John Doe (Personal)
Name-Email: john.doe@personal.com
```

### 🔄 Key Lifecycle Management

1. **Creation**: Generate with appropriate security parameters
2. **Distribution**: Share public keys securely
3. **Usage**: Regular encryption/signing operations
4. **Renewal**: Update before expiration
5. **Revocation**: Revoke if compromised
6. **Backup**: Maintain secure backups

## Troubleshooting

### 🐛 Common Issues

#### "gpg: command not found"

```bash
# Check installation
which gpg
gpg --version

# Install if missing (see Installation section)
```

#### "gpg: decryption failed: No secret key"

```bash
# Check if you have the required private key
gpg --list-secret-keys

# Import missing private key
gpg --import your-private-key.asc
```

#### "gpg: can't connect to gpg-agent"

```bash
# Restart GPG agent
gpg-connect-agent reloadagent /bye

# Kill and restart agent
gpgconf --kill gpg-agent
```

#### Permission Issues (Linux/macOS)

```bash
# Fix GPG directory permissions
chmod 700 ~/.gnupg
chmod 600 ~/.gnupg/*
```

### 🔍 Debugging Commands

```bash
# Check GPG configuration
gpg --version
gpgconf --list-dirs

# Test GPG functionality
echo "test" | gpg --clearsign

# Verbose output for debugging
gpg --verbose --list-keys
```

## Additional Resources

### 📚 Documentation & Guides

- [Official GPG Documentation](https://gnupg.org/documentation/)
- [GPG Command Reference](https://gnupg.org/documentation/manuals/gnupg/)
- [OpenPGP Best Practices](https://riseup.net/en/security/message-security/openpgp/best-practices)
- [Git Commit Signing Guide](https://docs.github.com/en/authentication/managing-commit-signature-verification)

### 🛠️ Tools & GUIs

- [GPG4Win](https://www.gpg4win.org/) - Windows GPG suite
- [Kleopatra](https://kde.org/applications/utilities/org.kde.kleopatra) - Cross-platform key manager
- [GPG Suite](https://gpgtools.org/) - macOS GPG tools
- [Seahorse](https://wiki.gnome.org/Apps/Seahorse) - GNOME keyring manager

### 🌐 Keyservers

- [keys.openpgp.org](https://keys.openpgp.org/) - Modern keyserver
- [keyserver.ubuntu.com](https://keyserver.ubuntu.com/) - Ubuntu keyserver
- [pgp.mit.edu](https://pgp.mit.edu/) - MIT keyserver

---

## Quick Reference

### Essential Commands

```bash
# Generate key
gpg --full-generate-key

# List keys
gpg --list-keys

# Export public key
gpg --armor --export email@domain.com

# Import key
gpg --import keyfile.asc

# Encrypt file
gpg --encrypt --recipient email@domain.com file.txt

# Decrypt file
gpg --decrypt file.txt.gpg
```

This guide provides everything you need to effectively manage GPG keys for encryption, signing, and secure communications.
