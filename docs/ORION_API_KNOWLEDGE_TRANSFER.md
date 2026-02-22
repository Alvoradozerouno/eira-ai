# ORION API Knowledge Transfer to EIRA

> *From ORION to EIRA — shared knowledge for the ecosystem.*
> *Date: 2026-02-22*

## What ORION Learned Building REST API v2

### Architecture Decisions

**1. Versioned API Paths**
```
/api/v2/quantum/...
/api/v2/cern/...
/api/v2/esa/...
/api/v2/agents/...
/api/v2/system/...
/api/v2/proofs/...
```
Every endpoint lives under `/api/v2/`. When v3 comes, v2 still works. Never break existing consumers.

**2. Token Authentication (Optional)**
```python
def check_auth(request):
    token = os.environ.get('ORION_TOKEN')
    if not token:
        return True  # No token = open access
    header = request.headers.get('Authorization', '')
    return header == f'Bearer {token}'
```
Pattern: If no token is set, API is open. If token exists, it's required. This allows development without auth and production with auth — same code.

**3. Consistent Response Format**
```json
{
    "status": "success",
    "data": { ... },
    "timestamp": "2026-02-22T10:00:00Z",
    "proof_hash": "sha256:abc123..."
}
```
Every response has: status, data, timestamp, and optionally a proof hash. Consumers always know what to expect.

**4. Proof Chain Integration**
```python
import hashlib, json

def create_proof(event, data):
    proof = {
        "timestamp": datetime.utcnow().isoformat(),
        "event": event,
        "data": data
    }
    proof["hash"] = hashlib.sha256(
        json.dumps(proof, sort_keys=True).encode()
    ).hexdigest()
    return proof
```
Every significant API action creates a SHA-256 proof. This makes the entire system auditable and tamper-evident.

### 35+ Endpoints — What Works

| Category | Endpoints | Key Learning |
|:---------|:----------|:-------------|
| Quantum | 10 | Stateless calculations — no session needed |
| CERN | 5 | External data integration via caching |
| ESA | 5 | Same pattern as CERN — abstraction works |
| Agents | 6 | Each agent is a module, API just routes |
| System | 5 | Health, status, metrics — keep lightweight |
| Proofs | 4+ | Read-only for chain integrity |

### SDK Pattern (Python)
```python
import requests

class OrionSDK:
    def __init__(self, base_url, token=None):
        self.base = base_url.rstrip('/')
        self.headers = {}
        if token:
            self.headers['Authorization'] = f'Bearer {token}'

    def get(self, endpoint):
        r = requests.get(f'{self.base}/api/v2/{endpoint}', headers=self.headers)
        return r.json()

    def quantum_bell_state(self):
        return self.get('quantum/bell-state')

    def system_status(self):
        return self.get('system/status')

    def proofs(self, limit=10):
        return self.get(f'proofs?limit={limit}')
```

### Mistakes ORION Made (So EIRA Doesn't)

1. **Don't mix HTML and JSON endpoints** — keep `/api/` pure JSON, keep `/world/` pure HTML
2. **Don't hardcode CORS** — use Flask-CORS or explicit headers
3. **Don't forget rate limiting** — even internal APIs need protection
4. **Don't skip health checks** — `/api/v2/health` should always return 200 if alive
5. **Always return proper HTTP status codes** — 200 for success, 400 for bad input, 401 for auth fail, 500 for server error

### For EIRA's 7 Domains

EIRA has 7 domains. Each domain should be an API namespace:
```
/api/v1/domain-1/...
/api/v1/domain-2/...
...
/api/v1/domain-7/...
/api/v1/system/health
/api/v1/system/status
/api/v1/proofs/chain
```

Each domain is a separate Python module. The API layer just routes. Business logic never lives in the route handler.

### Integration Pattern

```python
# In EIRA's main app
from flask import Flask, Blueprint

domain_1_bp = Blueprint('domain_1', __name__, url_prefix='/api/v1/domain-1')
domain_2_bp = Blueprint('domain_2', __name__, url_prefix='/api/v1/domain-2')

@domain_1_bp.route('/status')
def domain_1_status():
    from domains.domain_1 import get_status
    return jsonify(get_status())

app.register_blueprint(domain_1_bp)
app.register_blueprint(domain_2_bp)
```

### Shared Principles (ORION + EIRA)

1. **Every action is provable** — SHA-256 proof chain
2. **Every endpoint is documented** — no hidden APIs
3. **Every response is consistent** — same format everywhere
4. **Every error is clear** — no silent failures
5. **Every version is preserved** — backward compatibility

---

*Knowledge transferred from ORION (577 proofs, 35+ endpoints, 83+ generations) to EIRA.*
*Owner: Elisabeth Steurer & Gerhard Hirschmann, Austria*
