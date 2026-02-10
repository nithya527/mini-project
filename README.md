from flask import Flask, request, redirect, session
from datetime import datetime
import os

app = Flask(__name__)
app.secret_key = "buycart_secret"

# ================= CONFIG =================
UPLOAD_FOLDER = "uploads"
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
app.config["UPLOAD_FOLDER"] = UPLOAD_FOLDER

# ================= DATA =================
users = {}
admins = {"admin":"admin123"}

products = [
    {"id":1,"name":"iPhone 15","price":79999,"img":"https://m.media-amazon.com/images/I/71d7rfSl0wL._SX679_.jpg"},
]

cart = {}
orders = []

# ================= STYLE =================
STYLE = """
<style>
body{margin:0;font-family:Segoe UI;background:#f4f4f4}
a{text-decoration:none}
.navbar{background:#0f1111;color:white;display:flex;align-items:center;padding:15px 25px}
.logo{font-size:26px;font-weight:bold;color:#ffd814}
.menu{margin-left:auto;position:relative}
.menu-btn{background:#232f3e;color:white;border:none;padding:10px 16px;border-radius:8px}
.dropdown{display:none;position:absolute;right:0;top:55px;background:white;width:220px;border-radius:14px;box-shadow:0 10px 30px rgba(0,0,0,.4)}
.menu:hover .dropdown{display:block}
.dropdown a{display:block;padding:14px;color:#333}
.dropdown a:hover{background:#eee}

.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:20px;padding:25px}
.card{background:white;border-radius:14px;padding:15px;box-shadow:0 5px 15px rgba(0,0,0,.2)}
.card img{width:100%;height:170px;object-fit:contain}
.card button{width:100%;margin-top:10px;background:#ffd814;border:none;padding:10px;font-weight:bold;border-radius:8px}

.box{width:400px;margin:60px auto;background:white;padding:25px;border-radius:14px;box-shadow:0 8px 25px rgba(0,0,0,.3)}
input,button{width:100%;padding:12px;margin:10px 0;border-radius:8px;border:1px solid #ccc}
button{background:#ffd814;border:none;font-weight:bold}

table{width:100%;border-collapse:collapse}
td,th{padding:10px;border-bottom:1px solid #ccc}
</style>
"""

# ================= HOME =================
@app.route("/")
def home():
    user=session.get("user","Guest")
    cards=""
    for p in products:
        cards+=f"""
        <div class='card'>
            <img src='{p["img"]}'>
            <h3>{p["name"]}</h3>
            <p>₹{p["price"]}</p>
            <a href='/add/{p["id"]}'><button>Add to Cart</button></a>
        </div>
        """
    return f"""
    <html><head>{STYLE}</head><body>
    <div class='navbar'>
        <div class='logo'>🛒 BuyCart</div>
        <div class='menu'>
            <button class='menu-btn'>☰ Menu</button>
            <div class='dropdown'>
                <a>👤 {user}</a>
                <a href='/signup'>Signup</a>
                <a href='/login'>Login</a>
                <a href='/cart'>Cart</a>
                <a href='/admin-login'>Admin</a>
                <a href='/logout'>Logout</a>
            </div>
        </div>
    </div>
    <div class='grid'>{cards}</div>
    </body></html>
    """

# ================= USER AUTH =================
@app.route("/signup",methods=["GET","POST"])
def signup():
    if request.method=="POST":
        users[request.form["username"]] = request.form["password"]
        return redirect("/login")
    return f"<html><head>{STYLE}</head><body><div class='box'><h2>Signup</h2><form method='post'><input name='username'><input type='password' name='password'><button>Create</button></form></div></body></html>"

@app.route("/login",methods=["GET","POST"])
def login():
    if request.method=="POST":
        u,p=request.form["username"],request.form["password"]
        if u in users and users[u]==p:
            session["user"]=u
            return redirect("/")
    return f"<html><head>{STYLE}</head><body><div class='box'><h2>Login</h2><form method='post'><input name='username'><input type='password' name='password'><button>Login</button></form></div></body></html>"

@app.route("/logout")
def logout():
    session.clear()
    return redirect("/")

# ================= CART =================
@app.route("/add/<int:id>")
def add(id):
    cart[id]=cart.get(id,0)+1
    return redirect("/cart")

@app.route("/cart")
def cart_view():
    total=0
    rows=""
    for pid,qty in cart.items():
        p=next(x for x in products if x["id"]==pid)
        total+=p["price"]*qty
        rows+=f"<tr><td>{p['name']}</td><td>{qty}</td><td>₹{p['price']*qty}</td></tr>"
    return f"<html><head>{STYLE}</head><body><div class='box'><h2>Cart</h2><table>{rows}</table><h3>Total ₹{total}</h3><a href='/order'><button>Place Order</button></a></div></body></html>"

# ================= ORDER =================
@app.route("/order")
def order():
    if "user" not in session:
        return redirect("/login")

    total=0
    items=[]
    for pid,qty in cart.items():
        p=next(x for x in products if x["id"]==pid)
        items.append(f"{p['name']} x{qty}")
        total+=p["price"]*qty

    order_id=f"ORD{len(orders)+1}"
    orders.append({
        "id":order_id,
        "user":session["user"],
        "items":items,
        "total":total,
        "time":datetime.now().strftime("%d-%m-%Y %H:%M")
    })
    cart.clear()

    qr=f"https://api.qrserver.com/v1/create-qr-code/?size=200x200&data={order_id}"

    return f"""
    <html><head>{STYLE}</head><body>
    <div class='box'>
        <h2>✅ Order Successful</h2>
        <p>Order ID: <b>{order_id}</b></p>
        <p>Total: ₹{total}</p>
        <img src='{qr}'>
        <a href='/'><button>Home</button></a>
    </div></body></html>
    """

# ================= ADMIN =================
@app.route("/admin-login",methods=["GET","POST"])
def admin_login():
    if request.method=="POST":
        if request.form["username"] in admins and admins[request.form["username"]]==request.form["password"]:
            session["admin"]=True
            return redirect("/admin")
    return f"<html><head>{STYLE}</head><body><div class='box'><h2>Admin Login</h2><form method='post'><input name='username'><input type='password' name='password'><button>Login</button></form></div></body></html>"

@app.route("/admin",methods=["GET","POST"])
def admin():
    if not session.get("admin"):
        return redirect("/admin-login")

    if request.method=="POST":
        file = request.files["img"]
        filename = file.filename
        path = os.path.join(app.config["UPLOAD_FOLDER"], filename)
        file.save(path)

        products.append({
            "id":len(products)+1,
            "name":request.form["name"],
            "price":int(request.form["price"]),
            "img":"/uploads/"+filename
        })

    rows=""
    for o in orders:
        rows+=f"<tr><td>{o['user']}</td><td>{', '.join(o['items'])}</td><td>₹{o['total']}</td><td>{o['time']}</td></tr>"

    return f"""
    <html><head>{STYLE}</head><body>
    <div class='navbar'><div class='logo'>Admin Dashboard</div></div>

    <div class='box'>
    <h3>Add Product</h3>
    <form method='post' enctype='multipart/form-data'>
        <input name='name' placeholder='Product name' required>
        <input name='price' placeholder='Price' required>
        <input type='file' name='img' required>
        <button>Add Product</button>
    </form>
    </div>

    <div class='box' style='width:700px'>
    <h3>Orders</h3>
    <table>
    <tr><th>User</th><th>Products</th><th>Total</th><th>Date</th></tr>
    {rows}
    </table>
    </div>
    </body></html>
    """

# ================= IMAGE SERVE =================
@app.route("/uploads/<path:filename>")
def uploaded_file(filename):
    return app.send_static_file("../uploads/"+filename)

# ================= RUN =================
if __name__=="__main__":
    app.run(debug=True)

