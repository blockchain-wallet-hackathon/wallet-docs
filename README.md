# Solana Hardware Wallet

A Raspberry Pi–based hardware wallet prototype for the Solana blockchain, designed to make cryptocurrency transactions more transparent, secure, and understandable before they are signed.

The project combines physical transaction verification, on-device signing, risk analysis, real-time market monitoring, and AI-generated explanations in a hardware-and-software wallet ecosystem.

**Current scope:** Solana Devnet prototype. This project is intended for development and demonstration, not for securing real cryptocurrency funds.

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [System Architecture](#system-architecture)
- [Technology Stack](#technology-stack)
- [Hardware Requirements](#hardware-requirements)
- [How It Works](#how-it-works)
- [Project Structure](#project-structure)
- [Hardware Setup](#hardware-setup)
- [Firmware Development](#firmware-development)
- [Desktop Application](#desktop-application)
- [Backend and Database](#backend-and-database)
- [AI Integration](#ai-integration)
- [Solana Payment Integration](#solana-payment-integration)
- [Security Design](#security-design)
- [Development Roadmap](#development-roadmap)
- [Demonstration](#demonstration)
- [Limitations](#limitations)
- [Future Improvements](#future-improvements)

---

## Overview

Traditional cryptocurrency wallets can present users with transaction details that are difficult to verify, particularly when malicious software modifies transaction information before signing.

This project addresses that problem by moving transaction verification onto a dedicated physical device.

The Raspberry Pi independently receives transaction data, interprets supported instructions, and displays the intended recipient and transfer amount on its own screen. The user must physically approve the transaction before the device signs it.

The desktop application provides wallet management, balances, transaction history, analytics, and payment functionality. A backend service supplies market data, stores historical observations, evaluates transaction risk, and generates explanations for significant market movements.

The project is built around three principles:

1. **Independent verification:** Transaction details must be checked on the hardware device rather than trusted solely from the desktop application.
2. **Physical authorization:** Signing requires an explicit action on the device.
3. **Explainable risk analysis:** Users receive contextual warnings and explanations instead of relying exclusively on automated risk scores.

All blockchain transactions in the initial implementation use Solana Devnet.

## Key Features

### Hardware wallet

- Raspberry Pi 4B running Raspberry Pi OS Lite.
- On-device generation of a 12-word BIP-39 recovery phrase.
- Solana-compatible Ed25519 key derivation.
- Private-key signing performed on the device.
- LCD display for transaction details and device status.
- Physical buttons for navigation, approval, and rejection.
- RGB backlight indicating transaction risk.
- Buzzer alerts for signing requests and elevated-risk transactions.
- USB serial communication with the desktop application.

### Desktop application

- Cross-platform desktop architecture using Svelte and Tauri.
- Device connection detection and onboarding.
- Wallet address and Devnet balance display.
- Transaction history and wallet details.
- Live cryptocurrency price charts.
- Transaction creation and hardware-assisted signing.
- Transaction risk explanations.
- Solana Pay payment requests and payment processing.

### Backend and analytics

- Historical cryptocurrency price collection.
- Time-series storage using Tiger Data and PostgreSQL.
- One-minute candle aggregation and chart data endpoints.
- Detection of unusually large market movements.
- AI-generated explanations for significant market changes.
- Device event logging for connection, signing, and rejection events.

### AI-assisted intelligence

- Transaction risk assessments using recipient information and transfer context.
- Risk rules that override unsafe model recommendations.
- AI-generated explanations of significant market movements.
- Persistent contextual memory through Backboard.
- A conversational assistant for wallet and market questions.

---

## System Architecture

The system is divided into four main components.

| Component | Responsibility |
|---|---|
| Raspberry Pi firmware | Holds signing keys, parses supported transactions, displays transaction details, and authorizes signing |
| Desktop application | Manages the user interface, constructs transactions, communicates with the device, and displays wallet information |
| Backend service | Retrieves market data, evaluates risk, stores events, and coordinates AI requests |
| Database and external services | Provide time-series storage, blockchain RPC access, market data, and AI capabilities |

### Transaction flow

1. The user creates a transfer in the desktop application.
2. The application constructs a Solana transaction message.
3. The application sends the serialized message to the Raspberry Pi over USB serial.
4. The device parses the message and independently identifies supported instructions.
5. The LCD displays the transfer amount and recipient address.
6. The device applies its own recipient-history and transaction-limit rules.
7. The user physically approves or rejects the transaction.
8. If approved, the device signs the transaction message with its private key.
9. The desktop application attaches the signature and submits the transaction to Solana Devnet.
10. The application monitors confirmation and displays the result.

The backend can provide additional risk context, but the device must not blindly trust the risk level supplied by the application.

### Market analytics flow

1. A backend poller retrieves cryptocurrency prices.
2. Price observations are stored in Tiger Data.
3. Continuous aggregates produce one-minute market candles.
4. The backend compares recent prices against historical observations.
5. Significant movements trigger an AI explanation request.
6. The desktop application displays the chart and the explanation, including source links when available.

---

## Technology Stack

| Layer | Technologies | Purpose |
|---|---|---|
| Hardware | Raspberry Pi 4B, Grove LCD, physical buttons, buzzer | Independent transaction verification |
| Operating system | Raspberry Pi OS Lite | Lightweight device environment |
| Firmware | C++, libsodium, trezor-crypto, libgpiod | Key management, cryptographic signing, and hardware control |
| Desktop UI | Svelte, TypeScript | Wallet interface and application screens |
| Desktop runtime | Tauri, Rust | Desktop packaging and serial communication |
| Blockchain | Solana Devnet, `@solana/web3.js` | Transaction construction, submission, and confirmation |
| Backend | C#/.NET or Node.js | Market data, risk evaluation, and API endpoints |
| Database | Tiger Data, PostgreSQL, TimescaleDB | Historical price and device-event storage |
| AI | Google Gemini API | Risk analysis and market explanations |
| Agent memory | Backboard | Persistent assistant context |
| Payments | Solana Pay | Payment requests and payment validation |

**Implementation decision:** Select either C#/.NET or Node.js for the backend before implementation. Avoid maintaining two competing backend implementations for the same responsibilities.

The initial design uses a public Solana Devnet RPC endpoint or an appropriate RPC provider. Market prices can be retrieved from supported market-data APIs such as Jupiter or CoinGecko.

---

## Hardware Requirements

### Primary components

- Raspberry Pi 4B
- Raspberry Pi power supply
- Raspberry Pi OS Lite
- Grove 16×2 RGB LCD with I²C
- Three Grove buttons, plus one spare
- Grove buzzer
- Grove cables, jumper wires, and breadboard
- USB-C data cable
- Powered USB hub if required

### Optional fallback

- Raspberry Pi Pico for an alternative serial connection.

The Pico is a communication fallback, not a replacement for the Raspberry Pi's signing functionality.

### GPIO connections

| Component | Raspberry Pi connection |
|---|---|
| LCD GND | Physical pin 6 |
| LCD VCC | Physical pin 2 (5 V) |
| LCD SDA | Physical pin 3 (GPIO2) |
| LCD SCL | Physical pin 5 (GPIO3) |
| Left button signal | GPIO17 |
| Middle button signal | GPIO27 |
| Right button signal | GPIO22 |
| Buzzer signal | GPIO23 |
| Button power | Physical pin 1 (3.3 V), where required by the module |
| Ground | Appropriate Raspberry Pi ground pin |

**Electrical safety:** Raspberry Pi GPIO pins are not 5 V tolerant. Never allow a 5 V signal to reach a GPIO input. Verify the voltage requirements of the specific LCD, button, and buzzer modules before connecting them.

The LCD backlight controller address may vary by hardware revision. Confirm the actual I²C addresses rather than assuming they are identical across modules.

---

## How It Works

### 1. Device initialization

When the desktop application launches, it checks whether a compatible device is connected.

- **No device connected:** Display `No device connected`.
- **Device connected and initialized:** Verify the device's identity and proceed to the wallet homepage if authentication succeeds.
- **Device connected but not initialized:** Begin the recovery-phrase setup process.
- **Invalid authentication or communication failure:** Display an error and provide a retry option.

The device's cryptographic identity should be based on its public key, not its MAC address. A MAC address may be useful as connection metadata, but it is not proof of device authenticity.

### 2. Wallet setup and recovery phrase

During first-time setup:

1. The device generates 128 bits of cryptographically secure entropy.
2. The entropy is converted into a 12-word BIP-39 mnemonic.
3. The device displays the words in sequence.
4. The user records the recovery phrase offline.
5. The user confirms the phrase through the onboarding flow.
6. The wallet derives its Solana signing key.
7. The device returns its public key to the application.
8. The application verifies the device identity and opens the homepage.

If phrase verification fails, the application allows a retry. The user can navigate back through the device's phrase display to review the words.

**Recovery-phrase handling:** The phrase and derived private key must never be sent to the desktop application, backend, database, or AI providers. The phrase should be displayed only during the controlled setup and recovery flow.

### 3. Wallet homepage

The homepage displays:

- Device name and connection status.
- Wallet public address.
- Solana Devnet balance.
- Wallet and transaction details.
- Historical transactions, including sent and received transfers.
- Live market charts.
- Relevant market-movement explanations.
- Access to payment and assistant features.

Balances and transaction history are retrieved from the configured Solana RPC provider. Market prices are retrieved from the backend.

### 4. Sending a transaction

The user enters a recipient address and transfer amount in the desktop application.

The application constructs a Solana transaction and sends its serialized message to the hardware wallet. The device parses the transaction and displays the relevant information.

The user holds the physical confirmation button for two seconds to authorize signing. A rejection returns a failure response without producing an authorized signature.

Unknown or unsupported instructions must be rejected by default rather than presented as safe.

### 5. Transaction risk indicators

The device uses the LCD backlight to communicate risk:

- **Green:** Low risk under the configured rules.
- **Yellow:** Elevated uncertainty or an unfamiliar recipient.
- **Red:** High risk, suspicious transaction details, or an unsupported instruction.

A buzzer provides an additional alert for signing requests and high-risk transactions.

These indicators communicate a risk assessment, not a guarantee that a transaction is safe.

---

## Project Structure

The following is the intended organization of the repository. Adjust paths as implementation progresses.

```text
solana-hardware-wallet/
├── firmware/
│   ├── src/
│   │   ├── main.cpp
│   │   ├── wallet/
│   │   ├── transaction/
│   │   ├── display/
│   │   ├── buttons/
│   │   ├── communication/
│   │   └── security/
│   ├── include/
│   └── CMakeLists.txt
│
├── app/
│   ├── src/
│   │   ├── components/
│   │   ├── routes/
│   │   ├── lib/
│   │   └── stores/
│   ├── src-tauri/
│   │   └── src/
│   └── package.json
│
├── backend/
│   ├── src/
│   │   ├── Api/
│   │   ├── MarketData/
│   │   ├── RiskEngine/
│   │   ├── Ai/
│   │   └── Persistence/
│   └── tests/
│
├── database/
│   └── migrations/
│
├── docs/
│   ├── architecture.md
│   ├── hardware-setup.md
│   └── protocol.md
│
├── tests/
├── .env.example
├── .gitignore
└── README.md
```

---

## Hardware Setup

### Step 1: Configure Raspberry Pi OS

Install Raspberry Pi OS Lite and connect to the device through a secure development connection.

Enable I²C using:

```bash
sudo raspi-config
```

Navigate to the interface options and enable I²C.

Verify the connected devices:

```bash
i2cdetect -y 1
```

The LCD controller and RGB backlight controller should appear at their detected addresses. If they do not appear, check the wiring, power supply, and module compatibility.

### Step 2: Establish USB serial communication

The firmware and desktop application communicate using a line-oriented serial protocol.

Configure the Raspberry Pi USB device interface as appropriate for the Pi model and operating-system configuration. Test the connection by sending a message from the desktop and verifying that the device receives and echoes it.

On the Raspberry Pi, the serial device path depends on the selected USB gadget or UART configuration. Confirm the actual device path before implementing the serial service.

If the USB gadget approach proves unreliable, the optional Pico-based UART connection can be used as a fallback.

### Step 3: Implement hardware drivers

Implement separate modules for:

- LCD text output.
- RGB backlight control.
- Button input and debouncing.
- Two-second confirmation holds.
- Buzzer alerts.
- Serial message handling.

Keep the drivers separate from wallet and transaction logic so they can be tested independently.

---

## Firmware Development

The firmware is responsible for all private-key operations and final transaction authorization.

### Cryptographic key generation

The intended wallet setup uses:

- libsodium for secure random-number generation and cryptographic primitives.
- trezor-crypto for BIP-39 mnemonic conversion and compatible hierarchical key derivation.
- Ed25519 for Solana transaction signatures.

The target derivation path is:

```text
m/44'/501'/0'/0'
```

This path is intended to provide compatibility with common Solana wallet conventions. Validate the derivation against known test vectors and a reference wallet before treating compatibility as established.

The implementation must clearly distinguish between the BIP-39 mnemonic, its derived seed, the Ed25519 private key, and the corresponding public key.

### Serial protocol

Each message consists of a single JSON object followed by a newline.

| Message | Purpose |
|---|---|
| `ping` | Verify connectivity and retrieve firmware version |
| `get_pubkey` | Retrieve the public key or return `not_initialized` |
| `setup` | Begin wallet initialization and display the recovery phrase |
| `sign_tx` | Request transaction signing after on-device verification |

A signing request contains the serialized transaction message and supporting metadata, including the proposed risk level.

A successful response contains the signature. A rejected request returns a rejection result without signing.

The protocol should include message validation, size limits, timeouts, error handling, and a defined response format.

### Transaction parsing

The firmware must parse the Solana serialized message according to the actual Solana message format, including compact-length arrays and instruction encoding.

The initial implementation recognizes the System Program's native SOL transfer instruction.

For a supported transfer, the device displays:

```text
SEND 0.50 SOL
TO 7xKX..9fQ2
```

The middle button can scroll through the full recipient address.

The parser must validate account indices, instruction lengths, transfer amounts, and all relevant message fields. It must not rely on fixed offsets without correctly parsing preceding variable-length fields.

Unknown instructions display `UNKNOWN TX` and cannot be signed in the initial prototype.

### Recipient history

The device maintains a local history of previously approved recipients.

The intended policy is:

- Previously approved recipients may receive a lower risk classification.
- New recipients are treated with greater caution.
- A new recipient receiving more than a configured share of the wallet balance is classified as high risk.
- The device can raise the risk classification supplied by the application, but must not lower a risk classification established by its own rules.

Recipient history must be stored in a format that supports integrity checks and appropriate persistence across restarts.

---

## Desktop Application

The application uses Svelte for the interface and Tauri for desktop packaging and native functionality.

The initial target is one operating system to reduce development complexity. Windows and macOS builds can be added once the main workflow is stable.

### Main screens

**Connection and onboarding**

- Device connection status.
- Setup instructions.
- Recovery-phrase confirmation.
- Retry and recovery navigation.
- Device authentication status.

**Wallet homepage**

- Wallet address and balance.
- Transaction history.
- Live cryptocurrency charts.
- Market-movement explanations.
- Access to the AI assistant.

**Send**

- Recipient address.
- Transfer amount.
- Risk level and explanation.
- Confirmation prompt directing the user to the hardware device.

**Request payment**

- Payment amount.
- Label and optional note.
- Generated Solana Pay link and QR code.
- Payment-status tracking.

**Pay request**

- Input for a Solana Pay link.
- Parsed recipient and amount.
- Transaction risk assessment.
- Hardware-based signing.

### Transaction submission

The application constructs the transaction using `@solana/web3.js`, serializes the message, and requests a signature from the device.

The signature is attached to the transaction before submission through the Solana RPC endpoint.

The application then monitors confirmation and updates the transaction history.

The application must not display a successful transfer merely because the device returned a signature. Submission and confirmation are separate steps.

---

## Backend and Database

The backend provides market-data collection, transaction risk context, analytics, AI integration, and data persistence.

### Price storage

The initial `price_ticks` table contains:

| Column | Type | Purpose |
|---|---|---|
| `time` | `TIMESTAMPTZ` | Observation timestamp |
| `symbol` | `TEXT` | Asset symbol |
| `price` | `DOUBLE PRECISION` | Observed price |

Convert the table into a time-series hypertable using Tiger Data/TimescaleDB.

The `device_events` table records connection, signing, and rejection events:

| Column | Type | Purpose |
|---|---|---|
| `time` | `TIMESTAMPTZ` | Event timestamp |
| `device_id` | `TEXT` | Device public-key identifier |
| `event` | `TEXT` | Event category |
| `detail` | `JSONB` | Non-secret event metadata |

Never store recovery phrases, private keys, PINs, or authentication secrets in event logs.

### Price polling and aggregation

The market-data service periodically retrieves SOL and selected token prices. The initial target interval is five seconds, subject to provider limits and availability.

A one-minute continuous aggregate named `candles_1m` summarizes price observations by asset and time bucket.

The aggregate supports:

- One-minute chart data.
- Recent market comparisons.
- Historical volatility and price-change analysis.
- Detection of large movements.

The desktop chart initially displays the most recent two hours of data.

Configure refresh and compression policies according to the database service's supported features and the project's data-retention requirements.

### Market movement detection

The backend compares the latest completed one-minute candle with the candle from approximately 60 minutes earlier.

If the percentage change exceeds a configured threshold, the service requests an explanation from the AI provider.

The detector should account for missing data, stale prices, provider outages, and assets with insufficient history.

---

## AI Integration

The project uses the Google Gemini API for model calls, with Backboard providing persistent assistant memory.

### Transaction risk assessment

Before a signing request, the backend can provide contextual information to the risk engine:

- Transfer amount.
- Percentage of wallet balance being transferred.
- Recipient address.
- Whether the recipient has been seen before.
- Recipient account age, when reliably available.
- Recipient transaction count, when reliably available.
- Relevant historical transaction information.

The AI returns structured data containing a risk level and a short explanation suitable for display on the hardware screen.

Example:

```json
{
  "level": "high",
  "reason": "New recipient, large transfer"
}
```

The reason should be limited to 16 characters in the final LCD-oriented implementation if the selected display format requires it. The example illustrates the data structure, not a finalized compliant reason string.

### Hard risk rules

Deterministic rules take precedence over the AI recommendation.

Initial rules include:

- A first-time recipient receiving more than 50% of the wallet balance is high risk.
- Unsupported transaction instructions are rejected.
- An unavailable or timed-out AI risk service produces a medium-risk fallback rather than a low-risk result.
- The hardware device independently applies its own recipient and transaction-limit rules.

The AI must never receive private keys or recovery phrases.

AI output is advisory. It does not replace transaction parsing, cryptographic validation, or physical approval.

### Market movement explanations

When the market detector identifies a significant movement, Gemini generates a concise explanation using available market context and grounded sources where supported.

The response should include:

- A summary of the observed price movement.
- Plausible contributing factors.
- Links to relevant supporting sources.
- Clear distinctions between confirmed facts, reported explanations, and speculation.

For example, an explanation may examine whether a major company announcement, regulatory development, or broader market event coincided with a price change.

A correlation between a news event and a market movement does not establish that the event caused the movement.

### Backboard memory

Backboard provides persistent contextual memory for the assistant.

The intended memory includes:

- Recipient labels and transfer history metadata.
- User-stated risk preferences.
- Previously displayed market explanations.

Memory can support recipient familiarity checks and a homepage chat panel.

Sensitive information must be excluded. Memory should be treated as contextual assistance, not as an authoritative source for signing decisions. The implementation must also define retention and deletion behavior.

---

## Solana Payment Integration

The optional payment module uses Solana Pay to create payment requests and process incoming requests.

### Request a payment

The user supplies an amount, label, and optional note.

The application:

1. Generates a fresh public reference key.
2. Builds a Solana Pay URL with the recipient, amount, reference, label, and message.
3. Generates a QR code using the `qrcode` package.
4. Provides a button to copy the payment link.
5. Monitors the reference for a matching transaction.
6. Validates the transfer amount and recipient.
7. Updates the request to `Paid` and links to the transaction on Solana Explorer.

### Pay a request

The user pastes a Solana Pay URL into the desktop application.

The application parses the request, validates the parameters, constructs the transfer, and sends the transaction through the same hardware-signing workflow used by ordinary transfers.

The reference key must be incorporated according to Solana Pay's protocol requirements so the receiving application can locate the payment.

**Acceptance test:** A payment request created on one computer can be paid using the hardware device, and the requester's application detects and validates the payment automatically.

All payment tests remain on Devnet during development.

---

## Security Design

Security is central to the project because a hardware wallet must remain trustworthy even when the connected computer is compromised.

### Cryptographic identity

The initial design uses the wallet's public key as its device identity.

The device signs an authentication challenge, allowing the application to verify that it is communicating with the holder of the corresponding private key.

A public key alone does not prove that a connection is fresh or that the device is communicating with the intended application. The final handshake should incorporate nonces, transcript authentication, replay protection, and explicit protocol-version checks.

### Encrypted communication

The planned hardened protocol uses:

- X25519 for ephemeral key agreement.
- HKDF-SHA256 for deriving session keys.
- ChaCha20-Poly1305 for authenticated encryption.
- Ed25519 signatures to authenticate the device's handshake.

The handshake must bind both parties' identities and the negotiated session parameters. Direction-specific keys, unique nonces, sequence numbers, and replay protection are required.

Simply deriving a session key or encrypting individual messages does not, by itself, establish a secure authenticated channel.

### PIN and encrypted storage

The planned hardening stage includes:

- PIN entry through the physical buttons.
- Argon2id-based key derivation.
- Authenticated encryption of stored key material using libsodium's secretbox.
- A defined retry and recovery policy.

The design must account for offline PIN guessing and the limitations of a general-purpose Raspberry Pi. A PIN and software encryption alone do not provide the protections of a dedicated secure element.

### Transaction verification

The device independently parses and displays supported transaction details before signing.

The firmware should reject malformed or unsupported transactions, validate the requested signer, and ensure the signature is applied only to the exact message shown to the user.

The desktop application cannot be trusted to accurately describe a transaction merely because its interface displays the intended recipient.

### Development safeguards

- Use Devnet exclusively during initial development.
- Never commit credentials or API keys.
- Store backend secrets in environment variables or a managed secret store.
- Keep Gemini API credentials on the backend.
- Avoid logging private or sensitive wallet information.
- Use a dedicated test wallet with no real funds.
- Rotate any development credentials that may have been exposed.
- Disable unused network interfaces only after confirming that remote development access is no longer needed.

---

## Development Roadmap

The following phases establish the recommended implementation order.

| Phase | Deliverables | Completion criteria |
|---|---|---|
| 1. Hardware | LCD, RGB backlight, buttons, buzzer | All components respond correctly |
| 2. Communication | USB serial protocol | Application and Pi exchange valid messages |
| 3. Wallet initialization | Recovery phrase and key derivation | Test wallet produces a valid Solana public key |
| 4. Desktop onboarding | Connection and setup screens | First-time and returning-device flows work |
| 5. Basic wallet | Balance and transaction history | Devnet wallet information displays correctly |
| 6. Transaction signing | Message transfer, signing, submission | A signed Devnet transfer is confirmed |
| 7. Clear signing | Instruction parser and risk indicators | Device independently displays supported transfers |
| 8. Market analytics | Price poller, database, chart | Historical and live price data display correctly |
| 9. AI risk analysis | Risk endpoint and deterministic rules | Risk results and failure handling work |
| 10. Market explanations | Movement detector and Gemini | Significant movements produce sourced explanations |
| 11. Persistent assistant | Backboard integration | Assistant uses approved contextual memory |
| 12. Hardening | PIN, encryption, handshake | Security tests pass for the implemented protocol |
| 13. Payments | Solana Pay requests and QR codes | Devnet payment is detected and validated |

### Priorities under time constraints

If time becomes limited, prioritize the transaction-signing flow and independent on-device verification.

The minimum viable demonstration must show:

1. Successful device communication.
2. Wallet initialization.
3. A signed Devnet transfer.
4. Independent display of the actual recipient and amount.
5. Physical transaction approval or rejection.
6. Detection of a deliberately mismatched transaction.

Market analytics and payment requests can be simplified. Persistent memory and advanced storage hardening should not take priority over getting the signing flow correct.

---

## Demonstration

The demo should prove that the hardware device provides an independent view of the transaction.

### 1. Connect the device

Plug the Raspberry Pi into the computer and show the application detecting the device.

If the wallet is uninitialized, complete onboarding and generate a test wallet.

### 2. Execute a normal transfer

Create a small SOL transfer on Devnet.

Show the amount and recipient on the hardware LCD, approve the transaction using the physical button, and verify the result in Solana Explorer with the cluster set to Devnet.

### 3. Demonstrate a compromised application

Enable a clearly labeled demo mode that deliberately constructs a transaction to a different recipient from the one entered in the user interface.

The application displays the requested recipient while the hardware device displays the actual transaction recipient.

The device should flag the unexpected recipient, display the higher-risk status, and allow the user to reject the transaction.

The altered transaction must remain a test scenario and must never be broadcast to mainnet.

### 4. Demonstrate market intelligence

Show the live price chart, identify a significant movement, and display an AI-generated explanation with supporting links.

### 5. Demonstrate payments, if implemented

Generate a payment request QR code, pay it through the hardware-signing flow, and show the requester application detecting and validating the payment.

### 6. Explain the architecture

Summarize the role of each component:

- Raspberry Pi: independent verification and signing.
- Svelte and Tauri: desktop wallet interface.
- Backend: risk context, price collection, and AI coordination.
- Tiger Data: time-series storage and aggregation.
- Gemini: risk analysis and market explanations.
- Backboard: persistent assistant context.
- Solana Devnet: transaction execution and testing.

---

## Limitations

This is a prototype, not a production-ready cryptocurrency custody system.

Important limitations include:

- Raspberry Pi 4B does not provide the same security guarantees as a dedicated hardware-wallet secure element.
- Private-key protection depends on the operating system, firmware, storage configuration, and physical security.
- Transaction parsing must be expanded before supporting instructions beyond the explicitly implemented subset.
- AI-generated risk scores and market explanations can be inaccurate.
- RPC providers and market-data services may be unavailable, rate-limited, or inconsistent.
- Public Devnet behavior and balances do not represent production mainnet conditions.
- A successful signature does not guarantee that a transaction will be accepted or confirmed.
- The security of a desktop-to-device channel depends on the complete authentication and encryption protocol, not on individual cryptographic algorithms alone.

Do not use this prototype to store or transact real funds until it has undergone substantial security engineering, independent review, and testing.

---

## Future Improvements

Potential improvements beyond the initial prototype include:

- Support for additional Solana transaction instructions with strict parsing and validation.
- Independent firmware security review and fuzz testing.
- A dedicated secure element or hardware security module.
- A formally specified and tested authenticated communication protocol.
- Improved wallet recovery and encrypted backup workflows.
- Multi-wallet account management.
- Broader transaction simulation and risk context.
- More comprehensive market analytics and historical volatility measurements.
- Improved source attribution and confidence reporting for AI explanations.
- Production-ready cross-platform installers.
- Accessibility improvements and clearer transaction-review screens.

## Acknowledgments and References

The project draws on the following technologies and documentation:

- [Solana Documentation](https://solana.com/docs)
- [Solana Pay](https://docs.solanapay.com/)
- [Tauri Documentation](https://tauri.app/)
- [Svelte Documentation](https://svelte.dev/docs)
- [libsodium Documentation](https://doc.libsodium.org/)
- [Tiger Data Documentation](https://docs.timescale.com/)
- [Google Gemini API](https://ai.google.dev/gemini-api/docs)
- [Backboard Documentation](https://docs.backboard.io/)
- [Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)

---

**Project status:** Prototype in development.

**Network:** Solana Devnet.

**Primary objective:** Demonstrate how independent hardware transaction verification can protect users from signing transactions that do not match what their desktop application claims they are approving.
