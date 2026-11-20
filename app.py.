import os, secrets, sqlite3
from datetime import datetime, timezone
from functools import wraps
from flask import Flask, render_template, request, redirect, url_for, session, flash, abort
from werkzeug.security import generate_password_hash, check_password_hash

app = Flask(__name__)
app.secret_key = os.environ.get("SECRET_KEY", "CHANGE-ME-before-deployment")
DB_PATH = os.environ.get("DATABASE_PATH", "shipments.db")
ADMIN_USERNAME = os.environ.get("ADMIN_USERNAME", "admin")
ADMIN_PASSWORD_HASH = os.environ.get("ADMIN_PASSWORD_HASH", "")

STATUSES = [
    "Shipment registered", "Package received at warehouse",
    "Dispatched from origin country", "In transit",
    "Arrived in destination country", "Customs clearance",
    "Out for delivery", "Successfully delivered"
]

def db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    with db() as conn:
        conn.execute("""CREATE TABLE IF NOT EXISTS shipments (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            tracking_number TEXT UNIQUE NOT NULL,
            sender TEXT NOT NULL,
            recipient TEXT NOT NULL,
            origin TEXT NOT NULL,
            destination TEXT NOT NULL,
            description TEXT DEFAULT '',
            status TEXT NOT NULL,
            current_location TEXT NOT NULL,
            estimated_delivery TEXT DEFAULT '',
            created_at TEXT NOT NULL,
            updated_at TEXT NOT NULL
        )""")
        conn.execute("""CREATE TABLE IF NOT EXISTS events (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            shipment_id INTEGER NOT NULL,
            status TEXT NOT NULL,
            location TEXT NOT NULL,
            note TEXT DEFAULT '',
            created_at TEXT NOT NULL,
            FOREIGN KEY(shipment_id) REFERENCES shipments(id)
        )""")

def now():
    return datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M UTC")

def admin_required(fn):
    @wraps(fn)
    def wrapped(*args, **kwargs):
        if not session.get("admin"):
            return redirect(url_for("login"))
        return fn(*args, **kwargs)
    return wrapped

@app.before_request
def csrf_setup():
    if "csrf" not in session:
        session["csrf"] = secrets.token_urlsafe(32)

@app.context_processor
def inject_globals():
    return {"csrf_token": session.get("csrf", ""), "company": "Express Logistics Services"}

@app.after_request
def security_headers(response):
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    response.headers["Content-Security-Policy"] = "default-src 'self'; style-src 'self' 'unsafe-inline';"
    return response

def validate_csrf():
    if request.form.get("csrf_token") != session.get("csrf"):
        abort(400, "Invalid form token")

@app.route("/")
def home():
    return render_template("index.html")

@app.route("/track", methods=["POST"])
def track():
    number = request.form.get("tracking_number", "").strip().upper()
    if not number:
        flash("Please enter a tracking number.", "error")
        return redirect(url_for("home"))
    return redirect(url_for("tracking_result", tracking_number=number))

@app.route("/track/<tracking_number>")
def tracking_result(tracking_number):
    with db() as conn:
        shipment = conn.execute("SELECT * FROM shipments WHERE tracking_number=?", (tracking_number.upper(),)).fetchone()
        events = []
        if shipment:
            events = conn.execute("SELECT * FROM events WHERE shipment_id=? ORDER BY id DESC", (shipment["id"],)).fetchall()
    return render_template("tracking.html", shipment=shipment, events=events, statuses=STATUSES)

@app.route("/admin/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        validate_csrf()
        username = request.form.get("username", "")
        password = request.form.get("password", "")
        if ADMIN_PASSWORD_HASH and username == ADMIN_USERNAME and check_password_hash(ADMIN_PASSWORD_HASH, password):
            session.clear()
            session["admin"] = True
            session["csrf"] = secrets.token_urlsafe(32)
            return redirect(url_for("dashboard"))
        flash("Login details are incorrect. Check your deployment configuration.", "error")
    return render_template("login.html")

@app.route("/admin/logout", methods=["POST"])
@admin_required
def logout():
    validate_csrf()
    session.clear()
    return redirect(url_for("home"))

@app.route("/admin")
@admin_required
def dashboard():
    with db() as conn:
        shipments = conn.execute("SELECT * FROM shipments ORDER BY id DESC").fetchall()
    return render_template("dashboard.html", shipments=shipments)

@app.route("/admin/new", methods=["GET", "POST"])
@admin_required
def new_shipment():
    if request.method == "POST":
        validate_csrf()
        sender = request.form.get("sender", "").strip()
        recipient = request.form.get("recipient", "").strip()
        origin = request.form.get("origin", "").strip()
        destination = request.form.get("destination", "").strip()
        location = request.form.get("location", "").strip()
        status = request.form.get("status", STATUSES[0])
        description = request.form.get("description", "").strip()
        eta = request.form.get("estimated_delivery", "").strip()
        if not all([sender, recipient, origin, destination, location]) or status not in STATUSES:
            flash("Please complete all required fields.", "error")
            return render_template("new_shipment.html", statuses=STATUSES)
        tracking = "ELS" + datetime.now().strftime("%y%m%d") + secrets.token_hex(3).upper()
        stamp = now()
        with db() as conn:
            cur = conn.execute("""INSERT INTO shipments
                (tracking_number,sender,recipient,origin,destination,description,status,current_location,estimated_delivery,created_at,updated_at)
                VALUES (?,?,?,?,?,?,?,?,?,?,?)""",
                (tracking,sender,recipient,origin,destination,description,status,location,eta,stamp,stamp))
            conn.execute("INSERT INTO events (shipment_id,status,location,note,created_at) VALUES (?,?,?,?,?)",
                         (cur.lastrowid,status,location,"Shipment created",stamp))
        flash(f"Shipment created. Tracking number: {tracking}", "success")
        return redirect(url_for("dashboard"))
    return render_template("new_shipment.html", statuses=STATUSES)

@app.route("/admin/update/<int:shipment_id>", methods=["GET", "POST"])
@admin_required
def update_shipment(shipment_id):
    with db() as conn:
        shipment = conn.execute("SELECT * FROM shipments WHERE id=?", (shipment_id,)).fetchone()
        if not shipment:
            abort(404)
        if request.method == "POST":
            validate_csrf()
            status = request.form.get("status", "")
            location = request.form.get("location", "").strip()
            note = request.form.get("note", "").strip()
            eta = request.form.get("estimated_delivery", "").strip()
            if status not in STATUSES or not location:
                flash("Choose a valid status and enter the current location.", "error")
                return render_template("update_shipment.html", shipment=shipment, statuses=STATUSES)
            stamp = now()
            conn.execute("UPDATE shipments SET status=?,current_location=?,estimated_delivery=?,updated_at=? WHERE id=?",
                         (status,location,eta,stamp,shipment_id))
            conn.execute("INSERT INTO events (shipment_id,status,location,note,created_at) VALUES (?,?,?,?,?)",
                         (shipment_id,status,location,note,stamp))
            flash("Shipment tracking information updated.", "success")
            return redirect(url_for("dashboard"))
    return render_template("update_shipment.html", shipment=shipment, statuses=STATUSES)

@app.errorhandler(400)
def bad_request(error):
    return render_template("error.html", code=400, message="The request could not be verified."), 400

@app.errorhandler(404)
def not_found(error):
    return render_template("error.html", code=404, message="We couldn't find that page or shipment."), 404

init_db()

if __name__ == "__main__":
    app.run(debug=False)
