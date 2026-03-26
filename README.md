# Crowdfunding Investment Platform (CIP)

## 📌 Project Overview

The Crowdfunding Investment Platform (CIP) is a blockchain-based system designed to enable startups to raise funds from multiple investors in a secure, transparent, and decentralized manner.
The platform uses Hyperledger Fabric to create a permissioned blockchain network with multiple private channels among participating organizations.

This system ensures trust, data integrity, and efficient validation of investment transactions.

---

## 🎯 Objectives

* To provide a decentralized crowdfunding solution
* To ensure transparency in investment transactions
* To eliminate dependency on intermediaries
* To implement secure validation mechanisms using blockchain
* To enable startups to raise funds efficiently

---

## 🏗️ System Architecture

The platform consists of four main organizations:

* **StartupOrg** — registers startups and creates funding projects
* **InvestorOrg** — registers investors and funds projects
* **ValidatorOrg** — validates KYC of startups/investors and approves/rejects projects
* **PlatformOrg** — releases funds after a project reaches its goal

The blockchain network uses a single channel `cip-main-channel` with CouchDB state databases and Private Data Collections (PDC) for sensitive KYC information.

---

## ⚙️ Technologies Used

* Hyperledger Fabric (via **Microfab** — single-process local network)
* Docker
* Go (chaincode)
* CouchDB (peer state database)
* Private Data Collections (PDC)

---

## 🔐 Privacy Design

Sensitive KYC data is **not stored on the public ledger**. Instead it is kept in Private Data Collections:

| Collection | Members | Sensitive Fields |
|---|---|---|
| `StartupPrivateData` | StartupOrg | PAN, GST, Incorporation Date |
| `InvestorPrivateData` | InvestorOrg | PAN, Aadhar, Annual Income |

The `GetStartup` and `GetInvestor` chaincode functions return public-safe views that omit these fields.  
Use `GetPrivateStartupData` / `GetPrivateInvestorData` from within the respective org to retrieve KYC details.

---

## 🚀 Microfab 1-Channel Setup (Ubuntu + Docker)

### Prerequisites

* Ubuntu (20.04 or later) with Docker installed and running
* `jq` installed: `sudo apt-get install -y jq`
* Hyperledger Fabric `peer` binary in your PATH

Download the Fabric peer binary if you don't have it:
```bash
# Download the install script, review it, then execute
curl -sSL https://raw.githubusercontent.com/hyperledger/fabric/main/scripts/install-fabric.sh -o install-fabric.sh
# Review the script before running it: cat install-fabric.sh
bash install-fabric.sh binary
export PATH=$PWD/bin:$PATH
```

### Step 1 — Start Microfab

```bash
# Pull the Microfab image
docker pull ghcr.io/hyperledger-labs/microfab:latest

# Start Microfab using the provided config (MICROFAB.txt)
export MICROFAB_CONFIG=$(cat MICROFAB.txt)
docker run -d \
  -e MICROFAB_CONFIG="$MICROFAB_CONFIG" \
  -p 9090:9090 \
  --name microfab \
  ghcr.io/hyperledger-labs/microfab:latest

# Wait for Microfab to be ready
sleep 5
```

### Step 2 — Extract MSP Identities

```bash
# Install the microfab helper (npx works without a global install)
curl -s http://console.127-0-0-1.nip.io:9090/ak/api/v1/components | \
  npx @hyperledger-labs/weftility enroll --wallet _wallets --msp _msp 2>/dev/null || true

# Alternative using the Microfab REST API directly
for org in StartupOrg ValidatorOrg InvestorOrg PlatformOrg; do
  mkdir -p _msp/${org}/${org,,}admin/msp/{signcerts,keystore,cacerts}
done
```

> **Tip:** After `docker run`, Microfab exposes the MSP materials at
> `http://console.127-0-0-1.nip.io:9090`. You can extract them with the
> `weftility` CLI tool as shown above, or follow the
> [Microfab documentation](https://github.com/hyperledger-labs/microfab).

### Step 3 — Deploy the Chaincode

```bash
export CHANNEL="cip-main-channel"
export CHAINCODE="cipcc"
export ORDERER="orderer-api.127-0-0-1.nip.io:9090"
export CORE_PEER_TLS_ENABLED=false

# Install on all peers (run for each org)
for ORG in StartupOrg ValidatorOrg InvestorOrg PlatformOrg; do
  ORG_LOWER=$(echo "$ORG" | tr '[:upper:]' '[:lower:]')
  export CORE_PEER_LOCALMSPID="${ORG}MSP"
  export CORE_PEER_MSPCONFIGPATH="$PWD/_msp/${ORG}/${ORG_LOWER}admin/msp"
  export CORE_PEER_ADDRESS="${ORG_LOWER}peer-api.127-0-0-1.nip.io:9090"
  peer lifecycle chaincode install cipcc.tar.gz
done

# Approve and commit (one-time)
# Get the package ID from the install output, then:
PACKAGE_ID=$(peer lifecycle chaincode queryinstalled 2>&1 | grep "cipcc_1" | awk '{print $3}' | tr -d ',')

for ORG in StartupOrg ValidatorOrg InvestorOrg PlatformOrg; do
  ORG_LOWER=$(echo "$ORG" | tr '[:upper:]' '[:lower:]')
  export CORE_PEER_LOCALMSPID="${ORG}MSP"
  export CORE_PEER_MSPCONFIGPATH="$PWD/_msp/${ORG}/${ORG_LOWER}admin/msp"
  export CORE_PEER_ADDRESS="${ORG_LOWER}peer-api.127-0-0-1.nip.io:9090"
  peer lifecycle chaincode approveformyorg \
    -o "$ORDERER" -C "$CHANNEL" -n "$CHAINCODE" -v 1 \
    --sequence 1 --package-id "$PACKAGE_ID" \
    --collections-config collections_config.json
done

peer lifecycle chaincode commit \
  -o "$ORDERER" -C "$CHANNEL" -n "$CHAINCODE" -v 1 --sequence 1 \
  --collections-config collections_config.json
```

### Step 4 — Run the Tests

All test scripts auto-detect the project root, so just run them from the repo:

```bash
cd tests/
bash test-functional.sh
bash test-privacy.sh
bash test-security.sh
bash test-failure.sh
bash test-concurrency.sh
```

Results are saved under `tests/results/` as CSV files.

---

## 🔄 Workflow

1. Startup registers (`RegisterStartup`) and waits for validator KYC approval
2. Validator approves/rejects startup (`ValidateStartup`)
3. Startup creates a funding project (`CreateProject`)
4. Validator approves/rejects project (`ApproveProject` / `RejectProject`)
5. Investors fund approved projects (`Fund`)
6. Once the funding goal is reached, Platform releases funds (`ReleaseFunds`)
7. Investors can raise disputes within 7 days (`RaiseDispute` / `ResolveDispute`)
8. If a project is cancelled, investors can claim refunds (`Refund`)

---

## 📋 Chaincode Functions

| Function | Caller | Description |
|---|---|---|
| `RegisterStartup` | StartupOrg | Register a new startup with KYC details |
| `RegisterInvestor` | InvestorOrg | Register a new investor with KYC details |
| `RegisterValidator` | ValidatorOrg | Register a new validator |
| `ValidateStartup` | ValidatorOrg | Approve or reject a startup's KYC |
| `ValidateInvestor` | ValidatorOrg | Approve or reject an investor's KYC |
| `CreateProject` | StartupOrg | Create a new funding project |
| `ApproveProject` | ValidatorOrg | Approve a project for funding |
| `RejectProject` | ValidatorOrg | Reject / cancel a project |
| `Fund` | InvestorOrg | Invest in an approved project |
| `ReleaseFunds` | PlatformOrg | Release funds to startup after goal is met |
| `Refund` | InvestorOrg | Claim refund on a cancelled project |
| `RaiseDispute` | InvestorOrg | Raise a dispute within 7-day window |
| `ResolveDispute` | ValidatorOrg | Resolve a raised dispute |
| `GetProject` | Any | Query project details |
| `GetStartup` | Any | Query public startup details |
| `GetInvestor` | Any | Query public investor details |
| `GetPrivateStartupData` | StartupOrg only | Query private startup KYC data |
| `GetPrivateInvestorData` | InvestorOrg only | Query private investor KYC data |

---

## 🚀 Future Enhancements

* Integration of AI chatbot using LLM
* Investor sentiment analysis
* Mobile application support
* Smart contract automation
* Tokenized investment model

---

## 📄 License

This project is developed for academic purposes.
