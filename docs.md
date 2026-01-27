## Wolfi Overview
`Wolfi` is a community Linux undistro designed for the container and cloud-native era. Chainguard started the Wolfi project to build Chainguard Containers, our collection of curated distroless images that meet the requirements of a secure software supply chain. This required a Linux distribution with components at the appropriate granularity and with support for `glibc` 
- https://www.gnu.org/software/libc/.

## What is Melange? (APK Packaging)

- `Melange` is a tool that helps you compile and package your application into an APK (Alpine Package Keeper) — the package format used by Alpine Linux.
- `Melange` works based on a configuration file `melange.yaml` which specifies:
    - Package name and version
    - Source files and dependencies
    - How to build the application (e.g., Go, Python, Rust builds)

- The result is a `.apk` file containing the compiled application along with metadata.
- `Melange` also supports signing APK packages to ensure integrity and authenticity.
- `Example`: If you have Go source code, `Melange` will compile it into a binary and package it into an `.apk` as a standard Alpine package.


## What is Apko? (Container Image Creation)
- `Apko` is a tool that creates OCI-compliant container images from existing APK packages.
- You don't need to write a `Dockerfile`. Instead, you provide a YAML configuration file to `Apko` which defines:
    - Base image (usually `Alpine` or Wolfi base images)
    - `APK` packages to install (including the ones you built with Melange plus necessary libraries)
    - Metadata such as entrypoint, user, environment variables, etc.

Apko builds a container image containing a minimal OS `(Alpine/Wolfi)` plus your APK package.

## Melange and Apko

| Task                       | Tool      | Result                                      |
|----------------------------|-----------|---------------------------------------------|
| Compile and package app    | Melange   | Creates an `.apk` file containing your app  |
| Create OCI container image | Apko      | Creates a container image from the `.apk`   |

## Key Signing and Trust (The "Key-Lock" Model)

To ensure security, Wolfi/Alpine uses a **Key-Lock** trust system to verify that packages haven't been tampered with.

### 1. The "Key" (Private: `melange.rsa`)
This is your **secret** signing key. You keep this safe (e.g., in GitHub Secrets).
- **Role**: Signer
- **Action**: When Melange builds a package, it uses this key to create a cryptographic signature on the `.apk` file. This signature says *"I, the owner of this private key, built this."*

### 2. The "Lock" (Public: `melange.rsa.pub`)
This is your **public** verification key. You distribute this to anyone who needs to install your packages (Apko, servers, users).
- **Role**: Verifier
- **Action**: When Apko (or `apk`) tries to install a package, it uses this public key to "unlock" or verify the signature.

### The Trust Flow
1. **Build**: Melange builds `package.apk` and signs it with `melange.rsa`.
2. **Publish**: You upload `package.apk` AND `melange.rsa.pub` to your repository (`https://wolfi.duyne.me`).
3. **Install**:
    - Apko downloads `package.apk`.
    - Apko looks at your configuration: `keyring: [https://wolfi.duyne.me/melange.rsa.pub]`.
    - Apko downloads the public key.
    - Apko checks: *Does the signature on this package match this public key?*
        - **Yes** ✅: The package is trusted and installed.
        - **No** ❌: The package is rejected.
