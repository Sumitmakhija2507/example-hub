# BNB Greenfield File Upload Service - Complete Implementation Guide

## Overview

This Node.js application provides a working implementation for uploading files to BNB Greenfield, a decentralized storage network. The service handles the complete upload process including object creation on-chain, transaction broadcasting, and file data upload to storage providers.

## How to Run

```javascript
clone it

npm install

node server.js


```


## How It Works

### Architecture Overview

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Client    │────▶│  Express Server  │────▶│  BNB Greenfield │
│  (Browser)  │     │   (Node.js)      │     │    Network      │
└─────────────┘     └──────────────────┘     └─────────────────┘
                            │                          │
                            ▼                          ▼
                    ┌──────────────┐          ┌─────────────────┐
                    │ Reed-Solomon │          │Storage Providers│
                    │   Encoder    │          │   (8 Active)    │
                    └──────────────┘          └─────────────────┘
```

### Key Components

1. **Express Server**: Handles HTTP requests and serves the upload interface
2. **Greenfield JS SDK**: Interacts with the BNB Greenfield blockchain
3. **Reed-Solomon Encoder**: Generates checksums for data integrity
4. **Ethers.js**: Manages wallet operations and transaction signing

---

## ⚠️ Important Notes

* In the BNB Greenfield setup, using only the API is insufficient. The integration requires **both the SDK and the API** — the SDK handles encryption, while the API verifies the encryption and completes the upload process.
  Additionally, the SDK is responsible for creating the on-chain object before the file is uploaded to BNB Greenfield.

---

# Why On-Chain Object Creation and Upload Are Separate in BNB Greenfield

## The Two-Phase Design

BNB Greenfield intentionally separates file uploads into two distinct phases:

1. **On-Chain Phase**: Object metadata creation on blockchain
2. **Off-Chain Phase**: Actual file data upload to storage providers

This separation is a fundamental architectural decision with several important benefits.

## Key Reasons for Separation

### 1. **Blockchain Scalability**
```
┌─────────────────┐
│   Blockchain    │ ← Stores only metadata (small)
│   (On-Chain)    │   - Object name, size, checksums
│                 │   - Owner, permissions
└─────────────────┘   - Transaction hash
         │
         │ Reference
         ▼
┌─────────────────┐
│Storage Providers│ ← Stores actual file data (large)
│  (Off-Chain)    │   - Can be GBs or TBs
│                 │   - Distributed across providers
└─────────────────┘
```

**Why it matters:**
- Blockchains are not designed to store large files
- Storing a 1GB file on-chain would be prohibitively expensive
- Transaction size limits would prevent large file uploads
- Block size constraints would be exceeded

### 2. **Cost Efficiency**

```javascript
// On-chain costs (your example)
{
  gasUsed: 1200,
  gasFee: '0.000006 BNB'  // ~$0.003 at current prices
}

// If 26-byte file was stored on-chain:
// Cost would be ~$0.003 for 26 bytes
// A 1GB file would cost ~$115,384 (impractical!)
```

### 3. **Performance Optimization**

```
Traditional Approach:          Greenfield Approach:
┌──────────────┐              ┌──────────────┐
│Upload 1GB    │              │Create Object │
│to Blockchain │              │(metadata)    │
│              │              │~2-3 seconds  │
│~Hours/Failed │              └──────┬───────┘
│              │                     │
└──────────────┘              ┌──────▼───────┐
                              │Upload Data   │
                              │to Storage    │
                              │~1-2 min/GB   │
                              └──────────────┘
```

### 4. **Storage Provider Flexibility**

The separation enables:
- **Dynamic provider selection** based on availability
- **Parallel uploads** to multiple providers
- **Geographic distribution** for better performance
- **Provider switching** if one fails

### 5. **Permission & Access Control**

```javascript
// Phase 1: Set permissions on-chain (immutable record)
{
  visibility: 1,  // PUBLIC_READ
  creator: wallet.address,
  paymentAddress: wallet.address
}

// Phase 2: Storage providers check blockchain for permissions
// before accepting/serving data
```

### 6. **Data Integrity Verification**

```javascript
// Phase 1: Register checksums on-chain
expectChecksums: [
  "checksum1",
  "checksum2",
  // ... 7 pieces total
]

// Phase 2: Storage providers verify data matches checksums
// This prevents corruption and ensures authenticity
```

## Technical Flow Breakdown

### Phase 1: On-Chain (Blockchain)
```javascript
// What happens:
1. Register object metadata
2. Assign storage providers
3. Set permissions
4. Record checksums
5. Generate unique transaction hash
6. Pay gas fees

// Result: Immutable record of file ownership and properties
```

### Phase 2: Off-Chain (Storage)
```javascript
// What happens:
1. Connect to assigned storage provider
2. Authenticate using transaction hash
3. Upload actual file bytes
4. Provider verifies checksums
5. Provider distributes to other nodes
6. Confirm successful storage

// Result: File data stored with redundancy
```

## Benefits of This Architecture

### For Users:
- ✅ **Lower costs** - Only pay small gas fee for metadata
- ✅ **Faster uploads** - No blockchain bottleneck for large files
- ✅ **Reliable storage** - Multiple providers ensure availability

### For the Network:
- ✅ **Scalability** - Can handle unlimited file sizes
- ✅ **Efficiency** - Blockchain only tracks what's necessary
- ✅ **Flexibility** - Storage providers can optimize independently

### For Developers:
- ✅ **Clear separation of concerns**
- ✅ **Better error handling** - Can retry uploads without new transactions
- ✅ **Audit trail** - Blockchain provides immutable history

## Real-World Analogy

Think of it like shipping a package:

1. **Phase 1 (On-Chain)**: Creating a shipping label
   - Records sender, receiver, package details
   - Generates tracking number
   - Pays for shipping

2. **Phase 2 (Off-Chain)**: Actually shipping the package
   - Courier picks up package
   - Verifies label matches package
   - Delivers to destination

The shipping company doesn't store your package details in their corporate database - they just track the metadata while the actual package travels through their logistics network.

This architecture combines the best of both worlds: blockchain's immutability and security for metadata with distributed storage's efficiency for actual data.

## Step-by-Step Upload Process

### 1. **Initialization Phase**
```javascript
// Server starts and initializes:
- Wallet from private key
- Greenfield client connection
- Reed-Solomon encoder
- Validates connection to 8 storage providers
```

### 2. **File Preparation**
When a file is uploaded:
- Generate unique object name with timestamp and random ID
- Example: `test-file-1749970129458-1a01a259`
- This prevents naming conflicts

### 3. **Checksum Generation**
```javascript
// Reed-Solomon encoding for data integrity
const fileBytes = new Uint8Array(file.buffer);
const expectCheckSums = rs.encode(fileBytes);
// Generates 7 pieces for redundancy
```

### 4. **On-Chain Object Creation**
```javascript
// Create object metadata on blockchain
const createObjectTx = await client.object.createObject({
  bucketName: bucketName,
  objectName: uniqueObjectName,
  creator: wallet.address,
  visibility: 1, // PUBLIC_READ
  contentType: file.mimetype,
  redundancyType: 0, // EC_TYPE
  payloadSize: Long.fromInt(file.buffer.length),
  expectChecksums: expectCheckSums
});
```

### 5. **Transaction Simulation**
- Estimates gas requirements
- Example output: `gasLimit: 1200n, gasPrice: '5000000000', gasFee: '0.000006'`

### 6. **Transaction Broadcasting**
- Signs and broadcasts transaction to blockchain
- Returns transaction hash for tracking
- Example: `F6F7E84294E57B2C461C9A8A2912096DC86C0F3FC632D2693E347B493C50FD92`

### 7. **File Data Upload**
```javascript
// Upload actual file content to storage provider
const uploadRes = await client.object.uploadObject({
  bucketName: bucketName,
  objectName: uniqueObjectName,
  body: file.buffer,
  txnHash: broadcastRes.transactionHash
});
```

## Technical Details

### Storage Provider Selection
- Automatically selects primary storage provider from 8 available
- Current primary: `0x46Bd342605Aed134e7eA2eE65b4af91486f66BF6`

### Redundancy & Reliability
- Uses EC_TYPE (Erasure Coding) for redundancy
- Reed-Solomon encoding creates 7 checksum pieces
- Ensures data integrity and availability

### Security Features
- ECDSA signature authentication
- Private key never exposed in responses
- Unique object names prevent overwrites

## API Endpoints

### Main Endpoints
- `GET /` - Upload interface
- `POST /upload` - File upload handler
- `GET /test-sdk` - SDK connection test
- `GET /check-balance` - Account verification
- `GET /health` - Service health check
- `POST /create-bucket` - Create new storage bucket

### File Access URLs
After successful upload, files are accessible at:
```
https://greenfield-sp.defibit.io/view/{bucketName}/{objectName}
https://greenfield-sp.nodereal.io/view/{bucketName}/{objectName}
```

## Configuration Requirements

### Prerequisites
1. **Private Key**: BNB Greenfield wallet private key
2. **BNB Balance**: For gas fees (approximately 0.000006 BNB per upload)
3. **Bucket**: Pre-existing bucket with write permissions

### Environment Setup
```javascript
const PRIVATE_KEY = '0x...'; // Your wallet private key
const GREEN_CHAIN_ID = '1017'; // Greenfield mainnet
const GRPC_URL = 'https://greenfield-chain.bnbchain.org';
```

## Error Handling

The service includes comprehensive error handling for:
- Insufficient funds
- Missing buckets
- Network connectivity issues
- Invalid file formats
- SDK parameter errors

## Success Metrics

From your example upload:
- ✅ File size: 26 bytes (JSON)
- ✅ Gas used: 1200 units
- ✅ Gas fee: 0.000006 BNB
- ✅ Upload time: ~5-10 seconds
- ✅ Success rate: 100% (when configured correctly)

## Best Practices

1. **Always use unique object names** to prevent conflicts
2. **Check wallet balance** before uploads
3. **Verify bucket exists** and has proper permissions
4. **Test with small files first** before large uploads
5. **Monitor gas prices** for cost optimization

## Troubleshooting Guide

### Common Issues:
1. **"Insufficient funds"** → Add BNB to wallet
2. **"Bucket not found"** → Create bucket first or check name
3. **"SDK parameter error"** → Ensure Reed-Solomon encoding is working
4. **"Network timeout"** → Check internet connection and retry
