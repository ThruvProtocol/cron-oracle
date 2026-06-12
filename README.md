# Thruv Protocol - Price Oracle Updater

Automated service that fetches FLOW token prices from CoinGecko and updates both the Flow blockchain PriceOracle contract and the backend PostgreSQL database.

## Features

- Fetches real-time FLOW/USD price from CoinGecko API
- Updates price on Flow blockchain (testnet) every 5 minutes
- Stores historical price data in PostgreSQL database
- Calculates and stores protocol APY snapshots based on FLOW price
- Supports multiple staking protocols (Ankr, Increment, Figment)
- Automatic retry and error handling
- Comprehensive logging
- Auto-loads .env configuration

## Architecture

```
┌─────────────────┐
│  CoinGecko API  │ ← Fetch FLOW price
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│    Price Oracle Updater     │
│  • Update blockchain price  │
│  • Store price history      │
│  • Calculate protocol APYs  │
└────┬───────┬────────────┬───┘
     │       │            │
     │       │            └────────────┐
     │       └──────────────┐          │
     ▼                      ▼          ▼
┌──────────┐    ┌─────────────────┐  ┌────────────────┐
│   Flow   │    │  price_oracle   │  │ protocol_apy_  │
│Blockchain│    │      table      │  │   snapshots    │
│(Testnet) │    │  (PostgreSQL)   │  │  (PostgreSQL)  │
└──────────┘    └─────────────────┘  └────────────────┘
```

## Prerequisites

- Go 1.21 or later
- PostgreSQL database (shared with backend)
- Flow blockchain account with PriceOracle admin access

## Installation

1. **Clone and navigate to the directory**
   ```bash
   cd cron-oracle
   ```

2. **Install dependencies**
   ```bash
   go mod tidy
   ```

3. **Run the database migration**
   ```bash
   psql $DATABASE_URL -f ../backend/migrations/007_price_oracle.sql
   ```

4. **Configure environment variables**
   ```bash
   cp .env.example .env
   # Edit .env with your configuration
   ```

## Configuration

Create a `.env` file with the following variables:

```bash
# Database Configuration
DATABASE_URL=postgresql://user:password@localhost:5432/trixy-flow-indexer

# Flow Blockchain Configuration
FLOW_PRIVATE_KEY=your_private_key_here
FLOW_ACCOUNT_ADDRESS=0x0a80bc2ee7f90ab5
PRICE_ORACLE_CONTRACT=0x0a80bc2ee7f90ab5
```

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `DATABASE_URL` | PostgreSQL connection string | ✅ Yes |
| `FLOW_PRIVATE_KEY` | Private key (ECDSA_secp256k1) with admin access | ✅ Yes |
| `FLOW_ACCOUNT_ADDRESS` | Flow account address with admin rights | ✅ Yes |
| `PRICE_ORACLE_CONTRACT` | PriceOracle contract address | ✅ Yes |

## Usage

### Run with Script (Recommended)

```bash
./run.sh
```

This automatically loads `.env` and starts the service.

### Run Manually

```bash
export $(cat .env | grep -v '^#' | xargs)
go run main.go
```

### Build and Run

```bash
go build -o oracle main.go
./oracle
```

## How It Works

1. **Price Fetching**: Every 5 minutes, fetches current FLOW/USD price from CoinGecko
2. **Blockchain Update**: Sends transaction to update PriceOracle contract on Flow testnet
3. **Database Storage**: Saves price data with transaction hash to `price_oracle` table
4. **APY Calculation**: Calculates protocol APY based on FLOW price (lower price = higher APY)
5. **APY Storage**: Saves APY snapshots for Ankr, Increment, and Figment to `protocol_apy_snapshots` table
6. **Logging**: Logs all operations (success/failure) to stdout

### Update Cycle

```
┌─────────────────────────────────────────┐
│  Fetch Price from CoinGecko             │
│  (e.g., $0.2784)                        │
└────────────┬──────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  Update Flow Blockchain                 │
│  - Create & sign transaction            │
│  - Send to PriceOracle contract         │
│  - Wait for seal                        │
└────────────┬──────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  Save Price to Database                 │
│  - Insert to price_oracle table         │
│  - Store TX hash & timestamp            │
└────────────┬──────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  Calculate Protocol APYs                │
│  - Ankr: 12.5% base × price impact      │
│  - Increment: 15.3% base × impact       │
│  - Figment: 10.8% base × impact         │
└────────────┬──────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  Save APY Snapshots                     │
│  - Insert to protocol_apy_snapshots     │
│  - Link to price_oracle record          │
└─────────────────────────────────────────┘
             │
             ▼
        Wait 5 minutes 
             │
             └──────────────┐
                            │
                            ▼
                    Repeat cycle...
```

## Database Schema

The service uses two tables:

### price_oracle

```sql
CREATE TABLE price_oracle (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    symbol VARCHAR(20) NOT NULL,
    price_usd DECIMAL(20, 8) NOT NULL,
    tx_hash TEXT,
    block_number BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### protocol_apy_snapshots

```sql
CREATE TABLE protocol_apy_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    protocol_name VARCHAR(50) NOT NULL,
    apy DECIMAL(10, 4) NOT NULL,
    flow_price DECIMAL(20, 8) NOT NULL,
    price_oracle_id UUID REFERENCES price_oracle(id),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### Query Examples

**Get latest price:**
```sql
SELECT * FROM price_oracle 
WHERE symbol = 'FLOW' 
ORDER BY created_at DESC 
LIMIT 1;
```

**Get price history (24 hours):**
```sql
SELECT * FROM price_oracle 
WHERE symbol = 'FLOW' 
AND created_at > NOW() - INTERVAL '24 hours'
ORDER BY created_at DESC;
```

**Calculate average price:**
```sql
SELECT 
    DATE_TRUNC('hour', created_at) as hour,
    AVG(price_usd) as avg_price,
    MIN(price_usd) as min_price,
    MAX(price_usd) as max_price
FROM price_oracle
WHERE symbol = 'FLOW'
GROUP BY hour
ORDER BY hour DESC;
```

**Get latest APY for all protocols:**
```sql
SELECT DISTINCT ON (protocol_name)
    protocol_name,
    apy,
    flow_price,
    created_at
FROM protocol_apy_snapshots
ORDER BY protocol_name, created_at DESC;
```

**Get APY history with price:**
```sql
SELECT 
    a.protocol_name,
    a.apy,
    a.flow_price,
    p.tx_hash,
    a.created_at
FROM protocol_apy_snapshots a
JOIN price_oracle p ON a.price_oracle_id = p.id
WHERE a.protocol_name = 'ankr'
ORDER BY a.created_at DESC
LIMIT 10;
```

## Testing

### Test Database Connection

```bash
./test_db.sh
```

This verifies:
- Database table exists
- Can insert records
- Can query records
- Indexes are working

### Verify Price Fetching

```bash
curl -s "https://api.coingecko.com/api/v3/simple/price?ids=flow&vs_currencies=usd" | python3 -m json.tool
```

## Logs

The service outputs structured logs:

```
Starting FLOW price oracle updater
Update interval: 5m0s
Contract address: 0x0a80bc2ee7f90ab5
Network: Flow Testnet

Flow account loaded: 0a80bc2ee7f90ab5 (keys: 1)
Database connection established
Fetched FLOW price: $0.2784
Price updated to $0.2784 (TX: 11b0313b...)
Price saved to database (ID: 0876c2bb...)
ankr APY: 21.52% (price impact: 1.72x)
increment APY: 26.34% (price impact: 1.72x)
figment APY: 18.59% (price impact: 1.72x)
```

### Log Symbols

- Service started
- Price fetched
- Operation successful
- Error occurred
- Database operation
- Status information

## Deployment

### Production Considerations

1. **Use systemd service** (Linux)
   ```ini
   [Unit]
   Description=Trixy Price Oracle
   After=network.target postgresql.service

   [Service]
   Type=simple
   User=trixy
   WorkingDirectory=/opt/trixy/cron-oracle
   EnvironmentFile=/opt/trixy/cron-oracle/.env
   ExecStart=/opt/trixy/cron-oracle/oracle
   Restart=always
   RestartSec=10

   [Install]
   WantedBy=multi-user.target
   ```

2. **Monitor logs**
   ```bash
   journalctl -u trixy-oracle -f
   ```

3. **Set up alerts** for:
   - Price fetch failures
   - Blockchain transaction failures
   - Database connection issues

## Troubleshooting

### Common Issues

**1. "failed to connect to Flow"**
- Check internet connection
- Verify Flow testnet is accessible: `access.devnet.nodes.onflow.org:9000`

**2. "failed to decode private key"**
- Ensure `FLOW_PRIVATE_KEY` is valid hex string
- Check key format (should be 64 hex characters)

**3. "failed to connect to database"**
- Verify PostgreSQL is running
- Check `DATABASE_URL` format
- Ensure database exists and migrations are run

**4. "transaction failed: Could not borrow admin resource"**
- Account doesn't have admin access to PriceOracle contract
- Verify correct account address and private key

**5. "Error fetching price"**
- CoinGecko API might be rate-limited
- Check internet connection
- Verify API endpoint is accessible

## Files

```
cron-oracle/
├── main.go              # Main application code
├── go.mod               # Go dependencies
├── go.sum               # Dependency checksums
├── .env                 # Environment configuration (gitignored)
├── .env.example         # Environment template
├── .gitignore           # Git ignore rules
├── run.sh               # Startup script
├── test_db.sh           # Database test script
├── README.md            # This file
├── README_DB.md         # Database integration docs
└── SETUP_COMPLETE.md    # Setup summary
```

## API Reference

### CoinGecko API

**Endpoint:**
```
GET https://api.coingecko.com/api/v3/simple/price?ids=flow&vs_currencies=usd
```

**Response:**
```json
{
  "flow": {
    "usd": 0.276873
  }
}
```

### Flow PriceOracle Contract

**Contract Address:** `0xe3f7e4d39675d8d3` (Testnet)

**Transaction Script:**
```cadence
import PriceOracle from 0xe3f7e4d39675d8d3

transaction(newPrice: UFix64) {
    prepare(signer: auth(Storage) &Account) {
        let admin = signer.storage.borrow<&PriceOracle.Admin>(
            from: PriceOracle.AdminStoragePath
        ) ?? panic("Could not borrow admin resource")
        
        admin.updateFlowPrice(newPrice: newPrice)
    }
}
```

## Contributing

When contributing, ensure:
1. Code compiles: `go build`
2. Tests pass: `./test_db.sh`
3. Environment variables are documented
4. Logs are clear and helpful

## License

Part of the Trixy Protocol project.

## Support

For issues or questions:
1. Check the troubleshooting section
2. Review logs for error messages
3. Verify all environment variables are set correctly
4. Test database connection with `./test_db.sh`

---

**Note:** This service is designed for Flow testnet. For mainnet deployment, update the `flowAccessNode` constant and contract address in `main.go`.
