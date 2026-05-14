# Zoho Integration from Catalyst Python Apps

Connecting your Catalyst Python AppSail app to Zoho Books, Zoho CRM, Zoho Desk, and other Zoho services.

---

## Integration Methods (Choose One)

| Method | Best for | Token management |
|---|---|---|
| **Connections** (Cloud Scale) | Functions, event-driven | Automatic |
| **Connectors** (SDK) | Functions with SDK | Automatic via Cache |
| **Direct REST API** | AppSail apps, full control | Manual refresh |

---

## Method 1: Direct REST API (Recommended for AppSail)

### Step 1 — Register your app at Zoho API Console

1. Go to [https://api-console.zoho.com](https://api-console.zoho.com)
2. Click **Add Client** → choose **Self Client** (same org) or **Server-based Application**
3. Copy the **Client ID** and **Client Secret**
4. Click **Generate Code** tab
5. Enter scopes (see table below) and set duration to **10 minutes**
6. Copy the **Grant Token (code)**

### Step 2 — Generate Refresh Token (one-time)

```bash
curl -X POST https://accounts.zoho.com/oauth/v2/token \
  -d "code=YOUR_GRANT_CODE" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "redirect_uri=https://www.zoho.com/books" \
  -d "grant_type=authorization_code"
```

Save the `refresh_token` from the response — this is your permanent credential.

### Step 3 — Store as Catalyst Environment Variables

In `app-config.json`:
```json
{
  "env_variables": {
    "ZOHO_CLIENT_ID": "your_client_id",
    "ZOHO_CLIENT_SECRET": "your_client_secret",
    "ZOHO_REFRESH_TOKEN": "your_refresh_token",
    "ZOHO_BOOKS_ORG_ID": "your_org_id",
    "ZOHO_ACCOUNTS_URL": "https://accounts.zoho.com"
  }
}
```

> For Indonesian data center, use `https://accounts.zoho.com` (global). If your Zoho org is on IN DC, use `https://accounts.zoho.in`.

### Step 4 — Token Helper in Python

Create `zoho_auth.py` in your project:

```python
import os
import requests
import time

_token_cache = {"access_token": None, "expires_at": 0}

def get_access_token():
    """Returns a valid Zoho access token, refreshing if expired."""
    now = time.time()
    if _token_cache["access_token"] and now < _token_cache["expires_at"] - 60:
        return _token_cache["access_token"]

    resp = requests.post(
        f"{os.environ['ZOHO_ACCOUNTS_URL']}/oauth/v2/token",
        data={
            "refresh_token": os.environ["ZOHO_REFRESH_TOKEN"],
            "client_id": os.environ["ZOHO_CLIENT_ID"],
            "client_secret": os.environ["ZOHO_CLIENT_SECRET"],
            "grant_type": "refresh_token"
        }
    )
    resp.raise_for_status()
    data = resp.json()

    _token_cache["access_token"] = data["access_token"]
    _token_cache["expires_at"] = now + data.get("expires_in", 3600)

    return _token_cache["access_token"]
```

---

## Zoho Books API Examples

### Get Invoices

```python
import os, requests
from zoho_auth import get_access_token

def get_invoices(status=None):
    token = get_access_token()
    org_id = os.environ["ZOHO_BOOKS_ORG_ID"]
    params = {"organization_id": org_id}
    if status:
        params["status"] = status  # e.g. "unpaid", "paid", "overdue"

    resp = requests.get(
        "https://www.zohoapis.com/books/v3/invoices",
        headers={"Authorization": f"Zoho-oauthtoken {token}"},
        params=params
    )
    return resp.json().get("invoices", [])

# Usage
unpaid = get_invoices(status="unpaid")
```

### Create Invoice

```python
def create_invoice(customer_id, line_items):
    token = get_access_token()
    org_id = os.environ["ZOHO_BOOKS_ORG_ID"]

    payload = {
        "customer_id": customer_id,
        "line_items": line_items
    }

    resp = requests.post(
        "https://www.zohoapis.com/books/v3/invoices",
        headers={
            "Authorization": f"Zoho-oauthtoken {token}",
            "Content-Type": "application/json"
        },
        params={"organization_id": org_id},
        json=payload
    )
    return resp.json()
```

### Get Contacts

```python
def get_contacts(contact_type="customer"):
    token = get_access_token()
    org_id = os.environ["ZOHO_BOOKS_ORG_ID"]

    resp = requests.get(
        "https://www.zohoapis.com/books/v3/contacts",
        headers={"Authorization": f"Zoho-oauthtoken {token}"},
        params={"organization_id": org_id, "contact_type": contact_type}
    )
    return resp.json().get("contacts", [])
```

### Get Expenses

```python
def get_expenses(from_date=None, to_date=None):
    token = get_access_token()
    org_id = os.environ["ZOHO_BOOKS_ORG_ID"]
    params = {"organization_id": org_id}
    if from_date:
        params["date_start"] = from_date  # format: YYYY-MM-DD
    if to_date:
        params["date_end"] = to_date

    resp = requests.get(
        "https://www.zohoapis.com/books/v3/expenses",
        headers={"Authorization": f"Zoho-oauthtoken {token}"},
        params=params
    )
    return resp.json().get("expenses", [])
```

---

## Zoho Books API Scopes Reference

| Scope | Access |
|---|---|
| `ZohoBooks.invoices.READ` | Read invoices |
| `ZohoBooks.invoices.CREATE` | Create/update invoices |
| `ZohoBooks.invoices.DELETE` | Delete invoices |
| `ZohoBooks.contacts.READ` | Read customers/vendors |
| `ZohoBooks.contacts.CREATE` | Create contacts |
| `ZohoBooks.expenses.READ` | Read expenses |
| `ZohoBooks.expenses.CREATE` | Create expenses |
| `ZohoBooks.reports.READ` | Read reports (P&L, balance sheet) |
| `ZohoBooks.payments.READ` | Read payments |
| `ZohoBooks.fullaccess.all` | Full access (use carefully) |

---

## Zoho CRM Integration

```python
def get_crm_leads():
    token = get_access_token()  # Use CRM scopes: ZohoCRM.modules.leads.READ

    resp = requests.get(
        "https://www.zohoapis.com/crm/v3/Leads",
        headers={"Authorization": f"Zoho-oauthtoken {token}"}
    )
    return resp.json().get("data", [])
```

**CRM Scopes:**
- `ZohoCRM.modules.leads.READ` — Read leads
- `ZohoCRM.modules.deals.READ` — Read deals
- `ZohoCRM.modules.contacts.ALL` — Full contact access

---

## Zoho Desk Integration

```python
DESK_ORG_ID = os.environ.get("ZOHO_DESK_ORG_ID", "666926728")

def get_tickets(status="open"):
    token = get_access_token()  # Scope: Desk.tickets.READ

    resp = requests.get(
        "https://desk.zoho.com/api/v1/tickets",
        headers={
            "Authorization": f"Zoho-oauthtoken {token}",
            "orgId": DESK_ORG_ID
        },
        params={"status": status}
    )
    return resp.json().get("data", [])
```

---

## Method 2: Catalyst Connections (for Functions)

If using Catalyst Serverless Functions (not AppSail), use the Python SDK Connector:

```python
import zcatalyst_sdk as catalyst

def handler(context, basic_io):
    app = catalyst.initialize(context)
    
    connector = app.connection({
        "ZohoBooksConnector": {
            "client_id": "YOUR_CLIENT_ID",
            "client_secret": "YOUR_CLIENT_SECRET",
            "auth_url": "https://accounts.zoho.com/oauth/v2/auth",
            "refresh_url": "https://accounts.zoho.com/oauth/v2/token",
            "refresh_token": "YOUR_REFRESH_TOKEN",
            "refresh_in": 3600
        }
    }).get_connector("ZohoBooksConnector")

    access_token = connector.get_access_token()
    # Use access_token in Zoho Books API calls
```

---

## Event-Driven: Signals + Zoho Books Publisher

For real-time reactions to Zoho Books events (invoice paid, contact created, etc.):

1. Go to **Catalyst Console → Signals → Publishers → Add Publisher**
2. Select **Zoho Books** as publisher type
3. Choose your Zoho Books organization
4. Create a **Rule**: select the event (e.g., `Invoice Updated`)
5. Add a **filter** (e.g., `status = "paid"`)
6. Set **Target** to your Catalyst Function or AppSail webhook endpoint

No auth configuration needed — Signals connects to Zoho Books internally.

---

## Data Center Domains

| Region | Zoho Accounts URL | Zoho API URL |
|---|---|---|
| Global (US) | `https://accounts.zoho.com` | `https://www.zohoapis.com` |
| India (IN) | `https://accounts.zoho.in` | `https://www.zohoapis.in` |
| Europe (EU) | `https://accounts.zoho.eu` | `https://www.zohoapis.eu` |
| Australia (AU) | `https://accounts.zoho.com.au` | `https://www.zohoapis.com.au` |

> Indonesia typically uses the global (US) data center. Confirm at: **Zoho Books → Settings → Organization Profile → Data Center**.
