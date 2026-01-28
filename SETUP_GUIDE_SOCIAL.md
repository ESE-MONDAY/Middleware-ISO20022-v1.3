# Proofrails Setup Guide – Get Started in 5 Minutes

**Transform blockchain tips into banking-standard ISO 20022 receipts, anchored immutably on Flare.**



##  What is Proofrails?

Proofrails is an open-source ISO 20022 Payments Middleware that:
- ✅ Converts blockchain tips into ISO 20022 XML (pain.001 format)
- ✅ Creates cryptographic evidence bundles
- ✅ Anchors bundle hashes on Flare blockchain
- ✅ Generates immutable receipts with audit trails
- ✅ Provides REST API + Web UI + Embeddable widgets

**Perfect for:**  fintech apps, compliance-first platforms.



##  Option 1: Local Development (5 minutes)

### Prerequisites
- Python 3.11+
- pip
- docker running

### Steps

```bash
# 1. Clone and enter the repo
git clone <repo-url>
cd Middleware-ISO20022-v1.3

# 2. Create .env file with anchoring config (optional, for Flare anchoring)
cat > .env << EOF
FLARE_RPC_URL=https://coston2-api.flare.network/ext/C/rpc
ANCHOR_CONTRACT_ADDR=0x262b1C649CE016717c62b9403E719C4801974CeF
ANCHOR_PRIVATE_KEY=0x<your_funded_coston2_private_key>
ANCHOR_ABI_PATH=contracts/EvidenceAnchor.abi.json
ARTIFACTS_DIR=artifacts
EOF

# 3. Start the API + PostgreSQL
docker compose up --build
```

**Note:** If you skip the `.env` file, the API still works—receipts just won't anchor to Flare (status stays "pending").

**That's it!** Open your browser:
-  **Dashboard**: http://127.0.0.1:8000/web/index.html
-  **API Docs**: http://127.0.0.1:8000/docs
-  **Swagger**: http://127.0.0.1:8000/redoc

### First Test

```bash
# Generate a test receipt
curl -X POST http://127.0.0.1:8000/v1/iso/record-tip \
  -H "Content-Type: application/json" \
  -d '{
    "tip_tx_hash": "0xabc123",
    "chain": "coston2",
    "amount": "0.001",
    "currency": "FLR",
    "sender_wallet": "0xSender",
    "receiver_wallet": "0xReceiver",
    "reference": "test:tip:1"
  }'
```

You'll get a `receipt_id` instantly. Status updates as the bundle anchors.



##  Option 2: Deploy to Railway (10 minutes)

### Prerequisites
- Railway account (free tier available)
- This repo connected to Railway
- Funded private key on Coston2 (for anchoring)

### Steps

1. **Create Web Service**
   - New Railway service from this repo
   - Build: Use Dockerfile (default)
   - Root directory: `Middleware-ISO-20022-payments`

2. **Add PostgreSQL**
   - Add Postgres add-on
   - Copy connection string

3. **Set Environment Variables**
   ```
   FLARE_RPC_URL=https://coston2-api.flare.network/ext/C/rpc
   ANCHOR_CONTRACT_ADDR=0x262b1C649CE016717c62b9403E719C4801974CeF
   ANCHOR_PRIVATE_KEY=0x<your_funded_coston2_key>
   ANCHOR_ABI_PATH=contracts/EvidenceAnchor.abi.json
   DATABASE_URL=postgresql://<user>:<pass>@<host>:<port>/<db>
   ARTIFACTS_DIR=/data/artifacts
   PUBLIC_BASE_URL=https://<your-api-domain>
   ```

4. **Add Volume**
   - Create 1-5 GB volume
   - Mount at `/data`

5. **Deploy**
   - Railway generates public URL automatically
   - Test: `curl https://your-domain/v1/health`

**Railway gives you:**
-  Auto-scaling API
-  Persistent PostgreSQL
-  File storage for artifacts
-  Public HTTPS domain



##  Integrate with Your App

### Step 1: Copy Integration Files

Copy these files into your Capella/Next.js project:
```
capella_integration/
├── lib/isoClient.ts           → your_project/lib/
└── app/api/iso/               → your_project/app/api/
```

### Step 2: Set Environment Variable

In your Capella `.env`:
```bash
# Local development
ISO_MIDDLEWARE_URL=http://localhost:8000

# Production (Railway)
ISO_MIDDLEWARE_URL=https://your-domain.railway.app
```

### Step 3: Use in Your Backend

```typescript
import { mwRecordTip, mwGetReceipt, mwVerify } from '@/lib/isoClient';

// After a tip succeeds:
const receipt = await mwRecordTip({
  tip_tx_hash: txHash,
  chain: 'coston2',
  amount: '10.5',
  currency: 'FLR',
  sender_wallet: tipper,
  receiver_wallet: author,
  reference: `capella:tip:${tipId}`,
});

// Store receipt.receipt_id in your DB

// Later, check status:
const status = await mwGetReceipt(receipt.receipt_id);

// Verify authenticity:
const verified = await mwVerify(status.bundle_url);
console.log(verified.matches_onchain); // true = anchored on Flare ✅
```

---

## 🎨 Display Receipts to Users

### Option A: Embedded Widget

```html
<iframe 
  src="https://your-domain/embed/receipt?rid=<receipt-id>&theme=light"
  width="500" 
  height="400">
</iframe>
```

### Option B: Full Page

```
https://your-domain/receipt/<receipt-id>
```

### Option C: Custom Dashboard

Use the REST API to build your own UI:
- `GET /v1/iso/receipts/{id}` – fetch receipt details
- `GET /v1/iso/events/{id}` – live updates via SSE
- `POST /v1/iso/verify` – verify bundle



##  API Key Management

### Public Dashboard
Visit: `https://your-domain/web/index.html`
- ✅ Generate API key (auto-creates project)
- ✅ View all receipts
- ✅ No authentication needed

### Project Dashboard
Visit: `https://your-domain/web/project.html`
- ✅ View only your project's receipts
- ✅ Rotate API key
- ✅ Requires API key in browser localStorage

### Admin Dashboard
Visit: `https://your-domain/web/admin.html`
- ✅ Manage all API keys (requires `ADMIN_TOKEN`)
- ✅ Create keys for specific projects



##  Core API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/iso/record-tip` | POST | Create receipt from tip |
| `/v1/iso/receipts/{id}` | GET | Fetch receipt details |
| `/v1/iso/verify` | POST | Verify bundle authenticity |
| `/v1/iso/events/{id}` | GET | Live updates (SSE) |
| `/v1/health` | GET | Health check |
| `/receipt/{id}` | GET | Full receipt page |
| `/embed/receipt` | GET | Embeddable widget |

**Full API docs**: `https://your-domain/docs`



##  Troubleshooting

### Anchoring not working?
- ✅ Ensure `ANCHOR_PRIVATE_KEY` is funded on Coston2
- ✅ Check `FLARE_RPC_URL` is accessible
- ✅ Verify contract address matches your network

### Database errors?
- ✅ For local dev: SQLite is automatic
- ✅ For Railway: ensure PostgreSQL add-on is connected
- ✅ Check `DATABASE_URL` format

### CORS issues?
- ✅ Set `WEB_ORIGIN` to your UI domain
- ✅ Set `PUBLIC_BASE_URL` to your API domain

### Missing XSD validation?
- ✅ Place `pain.001.001.09.xsd` in `schemas/` folder
- ✅ Validation is optional; bundles still anchor without it



## 🚀 Next Steps

1. **Clone & Run Locally**
   ```bash
   git clone <repo>
   cd Middleware-ISO20022-v1.3
   pip install -r requirements.txt
   uvicorn app.main:app --reload
   ```

2. **Deploy to Railway** (or any container platform)
   - Connect repo
   - Add Postgres
   - Set env vars
   - Deploy

3. **Integrate with Your App**
   - Copy `capella_integration/` files
   - Update `ISO_MIDDLEWARE_URL`
   - Start using the client

4. **Verify Receipts**
   - Share receipt links with users
   - They can verify authenticity on Flare ✅

---

##  Resources

- **Full Documentation**: See `README.md`
- **API Reference**: `API_Documentation.md`
- **Deployment Guide**: `DEPLOY-RAILWAY.md`
- **Capella Integration**: `capella_integration/README.md`

---

##  Questions?

- 🐛 Found a bug? Open an issue
- 💬 Need help? Check the docs folder
- 🤝 Want to contribute? PRs welcome!



## 📈 Key Features At a Glance

 **Instant Receipts** – data available immediately  
 **Cryptographically Signed** – Ed25519 signatures  
 **On-Chain Anchored** – immutable proof on Flare  
 **ISO 20022 Compliant** – pain.001.001.09 XML  
 **Zero Polling** – Server-Sent Events for live updates  
 **Embeddable Widgets** – iframe-ready receipt display  
 **Multi-Project** – API keys scoped to projects  
 **REST API** – production-ready endpoints  


