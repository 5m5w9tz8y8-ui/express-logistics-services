# Express Logistics Services — website starter

A responsive international-shipping website with a public tracking lookup and a private, single-admin shipment dashboard.

## Included
- Company homepage branded Express Logistics Services, headquartered in Florida, USA
- Public tracking lookup by tracking number
- Private admin login
- Create shipment records and automatically generate ELS tracking numbers
- Add timestamped shipment status/location updates
- SQLite database for a starter deployment

## Run locally
1. Install Python 3.11+.
2. In this folder, create a virtual environment and activate it.
3. Install dependencies: `pip install -r requirements.txt`
4. Generate a password hash:
   `python -c "from werkzeug.security import generate_password_hash; print(generate_password_hash('YOUR-STRONG-PASSWORD'))"`
5. Set environment variables before starting:
   - `SECRET_KEY` to a long random secret
   - `ADMIN_USERNAME` to your chosen admin username
   - `ADMIN_PASSWORD_HASH` to the generated hash
6. Start: `python app.py`
7. Open `http://127.0.0.1:5000`
8. Admin login: `http://127.0.0.1:5000/admin/login`

## Deployment
Deploy as a Python web service with Gunicorn (`gunicorn app:app`). Set the environment variables above in the hosting provider's secret/configuration settings. Use persistent storage for `shipments.db`; otherwise SQLite data may be lost when an instance is replaced. Connect your purchased domain through the hosting provider's domain settings and enable HTTPS.

## Important before accepting real shipments
This is a functional starter, not a fully audited enterprise logistics platform. Before production use:
- Set strong unique secrets and admin credentials; never deploy with the example values.
- Use a host with persistent database storage and automated encrypted backups.
- Add rate limiting and monitoring to the public tracking endpoint and login.
- Consider managed PostgreSQL for multi-instance production deployments.
- Add privacy policy, terms, contact details, and a process for customer data requests.
- Do not store payment-card details or sensitive identity documents in this starter.
- Only enter truthful shipment scans/statuses. No live GPS carrier integrations are included.
- Configure email/SMS notifications only after choosing a verified provider and consent workflow.

## Routes
- `/` — public homepage
- `/track/<tracking_number>` — public shipment result
- `/admin/login` — admin sign-in
- `/admin` — shipment dashboard
