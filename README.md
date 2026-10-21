# Multi-client Secure File Transfer System

A command-line file-transfer service built in Python. Multiple clients can connect to the same server concurrently, authenticate over TLS, and upload, list, or download files that belong to their account.

> **Project status:** The repository is currently being scaffolded. The commands and layout below describe the intended first-version interface and implementation boundaries.

## Features

- Concurrent TCP client connections
- TLS-protected client/server communication
- Username/password authentication
- Per-user file ownership and authorization
- Upload, list, download, and logout operations
- Length-prefixed JSON control messages
- Binary-safe file transfers in bounded chunks
- SHA-256 verification for downloaded files
- Safe filename validation and path traversal protection
- Configurable file-size and connection limits
- Temporary-file handling to avoid exposing incomplete uploads
- Tests for framing, authentication, authorization, malformed requests, interrupted connections, and concurrent clients

## Scope

### In scope

This project focuses on a small, educational file service with practical security controls:

1. The server listens on a configured address and port.
2. Clients establish a TLS-protected TCP connection.
3. Clients authenticate with a username and password.
4. The server validates each command and checks the authenticated user's permissions.
5. Files are stored in per-user locations.
6. Uploads and downloads use bounded binary chunks.
7. The client verifies downloaded content using SHA-256.

### Out of scope

The first version does not target:

- Public cloud deployment
- A browser or mobile interface
- End-to-end encryption where the server cannot read files
- Distributed storage, replication, or high availability
- Resuming interrupted transfers

## Architecture

```text
                         TLS-protected TCP
    +----------------+  ---------------------->  +----------------+
    |  Client CLI A  |                            |                |
    +----------------+                            |                |
                                                  |  Secure file   |
    +----------------+  ---------------------->  |  server        |
    |  Client CLI B  |                            |                |
    +----------------+                            |                |
                                                  +-------+--------+
                                                          |
                 +------------------+---------------------+------------------+
                 |                  |                    |                  |
           Protocol/framing     Authentication       Authorization       File storage
```

### Planned modules

| Module | Responsibility |
| --- | --- |
| `server.py` | Listen for connections, configure TLS, enforce connection limits, and run client workers |
| `client.py` | Provide the CLI, authenticate, issue commands, transfer files, and verify hashes |
| `protocol.py` | Encode/decode JSON control messages, read exact byte counts, and enforce message limits |
| `security.py` | Configure TLS, hash passwords with Argon2id, validate login attempts, and expose security helpers |
| `file_manager.py` | Validate names, enforce size limits, safely write uploads, list owned files, and authorize downloads |
| `config.py` | Keep host, port, storage, TLS, and limit settings separate from application logic |
| `tests/` | Unit, integration, security, and concurrency tests |

## Requirements

- Python 3.11 or newer
- OpenSSL (for generating a local development certificate)
- A local network interface or loopback access

The planned password-hashing implementation uses Argon2id through `argon2-cffi`. TLS should use Python's standard-library `ssl` module rather than custom cryptography.

## Installation

Create and activate a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install the project dependencies once a dependency manifest is present:

```bash
python -m pip install -r requirements.txt
```

For development, install the test dependencies as well:

```bash
python -m pip install -r requirements-dev.txt
```

## TLS development certificate

Never commit private keys or production certificates. For local development, generate a self-signed certificate and keep the resulting files outside version control:

```bash
mkdir -p certs
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 \
  -nodes \
  -keyout certs/server.key \
  -out certs/server.crt \
  -subj "/CN=localhost"
```

The client should trust `certs/server.crt` explicitly. It should not disable certificate verification or use an unverified TLS context.

## Running the application

Once the server and client modules are implemented, start the server in one terminal:

```bash
python server.py
```

Start one or more clients in separate terminals:

```bash
python client.py
```

Configuration should be supplied through the project’s configuration mechanism rather than hard-coded credentials or private-key paths. At minimum, the server configuration should cover:

- Bind host and port
- Storage root
- TLS certificate and private-key paths
- Trusted certificate or CA path
- Maximum upload size
- Maximum concurrent connections
- Protocol message-size limits

## Client workflow

The intended CLI workflow is:

1. Connect to the server.
2. Authenticate with a username and password.
3. Choose one of the supported operations:
   - Upload a file
   - List the authenticated user's files
   - Download an owned file
   - Log out
4. For downloads, compare the received SHA-256 digest with the server-provided digest.

The server must reject unauthenticated file operations, access to another user’s files, unsafe filenames, oversized uploads, malformed messages, and invalid commands.

## Protocol design

Control messages use a bounded length prefix followed by UTF-8 JSON. File payloads are transferred separately so arbitrary binary data cannot be confused with control messages.

Implementations should:

- Read exact byte counts from sockets
- Reject lengths above the configured message limit
- Validate JSON structure and command allowlists
- Treat socket closure and partial reads as transfer errors
- Avoid loading whole files into memory
- Return explicit success or error responses

## Storage and security requirements

- Normalize and validate filenames before constructing filesystem paths.
- Prevent `..`, absolute paths, and separator-based path traversal.
- Scope every file operation to the authenticated user’s directory.
- Use temporary files for uploads and atomically rename only after successful completion and digest validation.
- Apply a maximum file size before accepting an upload.
- Hash passwords with Argon2id; never store plaintext passwords.
- Keep credentials, private keys, and certificates out of source control.
- Use TLS certificate validation on the client.
- Limit concurrent connections to protect server resources.
- Do not log passwords, private keys, or file contents.

## Testing

Run the test suite with:

```bash
python -m pytest
```

The test suite should cover:

- Length-prefixed message encoding and decoding
- Partial reads and interrupted connections
- Authentication success and failure
- Per-user access-control boundaries
- Filename and path traversal validation
- Upload size limits and temporary-file cleanup
- Download integrity verification
- Malformed or unknown commands
- Multiple clients operating concurrently
- TLS certificate and handshake behavior

## Suggested repository layout

```text
.
├── README.md
├── client.py
├── server.py
├── protocol.py
├── security.py
├── file_manager.py
├── config.py
├── requirements.txt
├── requirements-dev.txt
├── certs/                  # local only; ignored by Git
├── storage/                # runtime data; ignored by Git
└── tests/
    ├── test_protocol.py
    ├── test_security.py
    ├── test_file_manager.py
    ├── test_client_server.py
    └── test_concurrency.py
```

## Responsible use

This project is intended for coursework, local testing, and controlled network environments. Before using it beyond a development environment, review certificate management, account provisioning, audit logging, rate limiting, backup strategy, denial-of-service protections, and operational secret management.

## License

No license has been specified yet. Add a license before distributing the project.
