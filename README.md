import tkinter as tk
from tkinter import ttk, messagebox
import mysql.connector
from mysql.connector import Error, IntegrityError
from datetime import date
import hashlib
import hmac
import os

# CHANGE THIS PASSWORD to your MySQL root password.
DB_HOST = "localhost"
DB_USER = "root"
DB_PASSWORD = "Pankaj@7488"
DB_NAME = "hotel_db"


def server_connection():
    return mysql.connector.connect(
        host=DB_HOST, user=DB_USER, password=DB_PASSWORD
    )


def connect_db():
    return mysql.connector.connect(
        host=DB_HOST, user=DB_USER, password=DB_PASSWORD,
        database=DB_NAME
    )


def setup_database():
    conn = server_connection()
    cur = conn.cursor()
    try:
        cur.execute(
            f"CREATE DATABASE IF NOT EXISTS `{DB_NAME}` "
            "CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci"
        )
    finally:
        cur.close()
        conn.close()

    conn = connect_db()
    cur = conn.cursor()
    try:
        cur.execute("""
            CREATE TABLE IF NOT EXISTS users (
                user_id INT AUTO_INCREMENT PRIMARY KEY,
                username VARCHAR(100) NOT NULL UNIQUE,
                password_hash VARCHAR(255) NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        cur.execute("""
            CREATE TABLE IF NOT EXISTS rooms (
                room_id INT AUTO_INCREMENT PRIMARY KEY,
                room_number VARCHAR(20) NOT NULL UNIQUE,
                room_type VARCHAR(50) NOT NULL,
                price DECIMAL(10,2) NOT NULL,
                status VARCHAR(20) NOT NULL DEFAULT 'Available'
            )
        """)
        cur.execute("""
            CREATE TABLE IF NOT EXISTS customers (
                customer_id INT AUTO_INCREMENT PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                phone VARCHAR(30) NOT NULL,
                email VARCHAR(150),
                address VARCHAR(255)
            )
        """)
        cur.execute("""
            CREATE TABLE IF NOT EXISTS bookings (
                booking_id INT AUTO_INCREMENT PRIMARY KEY,
                customer_id INT NOT NULL,
                room_id INT NOT NULL,
                check_in DATE NOT NULL,
                check_out DATE NOT NULL,
                total_amount DECIMAL(10,2) NOT NULL,
                status VARCHAR(20) NOT NULL DEFAULT 'Booked',
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
                FOREIGN KEY (room_id) REFERENCES rooms(room_id)
            )
        """)
        conn.commit()
    finally:
        cur.close()
        conn.close()


def make_password_hash(password, salt=None):
    if salt is None:
        salt = os.urandom(16)
    digest = hashlib.pbkdf2_hmac(
        "sha256", password.encode("utf-8"), salt, 200_000
    )
    return salt.hex() + "$" + digest.hex()


def check_password(password, saved):
    try:
        salt_hex, expected = saved.split("$", 1)
        salt = bytes.fromhex(salt_hex)
        actual = hashlib.pbkdf2_hmac(
            "sha256", password.encode("utf-8"), salt, 200_000
        ).hex()
        return hmac.compare_digest(actual, expected)
    except (ValueError, TypeError):
        return False


class HotelManagementSystem:
    BG = "#f2f5f9"
    NAVY = "#172554"
    BLUE = "#1d4ed8"
    GREEN = "#15803d"
    RED = "#b91c1c"
    MUTED = "#475569"

    def __init__(self, root):
        self.root = root
        self.root.title("RITA Hotel Management System")
        self.root.geometry("1150x720")
        self.root.minsize(950, 600)
        self.current_user = None
        self.dark_mode = False
        self.theme = {
            'bg': self.BG, 'panel': 'white', 'fg': self.NAVY,
            'muted': self.MUTED, 'input': 'white', 'border': '#dbe3ef',
            'table_heading': '#dbeafe', 'button': self.BLUE,
        }
        self.root.configure(bg=self.theme['bg'])

        style = ttk.Style()
        style.theme_use("clam")
        style.configure("Treeview", rowheight=29, font=("Arial", 10),
                        background=self.theme['panel'], fieldbackground=self.theme['panel'])
        style.configure("Treeview.Heading", font=("Arial", 10, "bold"),
                        background=self.theme['table_heading'], foreground=self.theme['fg'])
        style.configure("TEntry", padding=5)

        header = tk.Frame(root, bg=self.NAVY, height=78)
        header.pack(fill="x")
        header.pack_propagate(False)
        tk.Label(header, text="RITA", font=("Arial", 24, "bold"),
                 fg="white", bg=self.NAVY).pack(side="left", padx=25, pady=15)
        self.theme_button = tk.Button(
            header, text='🌙 Dark Mode', command=self.toggle_theme,
            bg='#334155', fg='white', activebackground='#475569',
            activeforeground='white', relief='flat', cursor='hand2',
            font=('Arial', 9, 'bold'), padx=10, pady=7)
        self.theme_button.pack(side='right', padx=(8, 20))
        tk.Label(header, text="HOTEL MANAGEMENT SYSTEM",
                 font=("Arial", 12, "bold"), fg="#bfdbfe",
                 bg=self.NAVY).pack(side="right", padx=8)

        self.content = tk.Frame(root, bg=self.theme['bg'])
        self.content.pack(fill="both", expand=True, padx=20, pady=14)

        footer = tk.Frame(root, bg=self.NAVY, height=40)
        footer.pack(side="bottom", fill="x")
        footer.pack_propagate(False)
        tk.Label(footer, text="© 2026 RITA HOTEL & RESORT",
                 bg=self.NAVY, fg="#bfdbfe", font=("Arial", 9)
                 ).pack(side="left", padx=20, pady=10)
        tk.Label(footer, text="Designed for Better Hospitality",
                 bg=self.NAVY, fg="white", font=("Arial", 9, "italic")
                 ).pack(side="right", padx=20, pady=10)

        try:
            setup_database()
            self.login_page()
        except Error as error:
            messagebox.showerror(
                "MySQL Connection Error",
                "Check that MySQL Server is running and DB_PASSWORD "
                "at the top of this file is correct.\n\n" + str(error)
            )
            self.root.after(100, self.root.destroy)

    # Common functions
    def clear(self):
        for widget in self.content.winfo_children():
            widget.destroy()

    def toggle_theme(self):
        self.dark_mode = not self.dark_mode
        if self.dark_mode:
            self.theme = {
                'bg': '#0f172a', 'panel': '#1e293b', 'fg': '#f8fafc',
                'muted': '#cbd5e1', 'input': '#334155', 'border': '#475569',
                'table_heading': '#334155', 'button': '#2563eb',
            }
            self.theme_button.configure(text='☀ Light Mode')
        else:
            self.theme = {
                'bg': self.BG, 'panel': 'white', 'fg': self.NAVY,
                'muted': self.MUTED, 'input': 'white', 'border': '#dbe3ef',
                'table_heading': '#dbeafe', 'button': self.BLUE,
            }
            self.theme_button.configure(text='🌙 Dark Mode')
        self.apply_theme()

    def apply_theme(self):
        self.root.configure(bg=self.theme['bg'])
        self.content.configure(bg=self.theme['bg'])
        style = ttk.Style()
        style.configure('TEntry',
                        fieldbackground=self.theme['input'],
                        foreground=self.theme['fg'])
        style.configure('Treeview',
                        background=self.theme['panel'],
                        fieldbackground=self.theme['panel'],
                        foreground=self.theme['fg'])
        style.configure('Treeview.Heading',
                        background=self.theme['table_heading'],
                        foreground=self.theme['fg'])
        # Rebuild the current page so every widget uses the selected theme.
        if self.current_user:
            page = getattr(self, 'current_page', 'dashboard')
            getattr(self, page)()
        else:
            # Login/signup cards are rebuilt with the selected theme too.
            page = getattr(self, 'auth_page', 'login_page')
            getattr(self, page)()

    def heading(self, title):
        tk.Label(self.content, text=title, font=("Arial", 21, "bold"),
                 bg=self.theme['bg'], fg=self.theme['fg']).pack(anchor="w", pady=(0, 10))

    def button(self, parent, text, command, color=None):
        return tk.Button(
            parent, text=text, command=command,
            bg=color or self.theme['button'], fg="white",
            activebackground="#1e40af", activeforeground="white",
            font=("Arial", 10, "bold"), relief="flat",
            padx=12, pady=8, cursor="hand2"
        )

    def query(self, sql, values=(), fetch=False):
        conn = connect_db()
        cur = conn.cursor()
        try:
            cur.execute(sql, values)
            if fetch:
                return cur.fetchall()
            conn.commit()
            return cur.rowcount
        finally:
            cur.close()
            conn.close()

    def show_error(self, error):
        messagebox.showerror("Error", str(error))

    def make_table(self, columns):
        frame = tk.Frame(self.content, bg=self.theme['bg'])
        frame.pack(fill="both", expand=True, pady=9)
        tree = ttk.Treeview(frame, columns=columns, show="headings")
        for col in columns:
            tree.heading(col, text=col.replace("_", " ").title())
            tree.column(col, width=135, minwidth=85, anchor="center")
        ys = ttk.Scrollbar(frame, orient="vertical", command=tree.yview)
        xs = ttk.Scrollbar(frame, orient="horizontal", command=tree.xview)
        tree.configure(yscrollcommand=ys.set, xscrollcommand=xs.set)
        tree.grid(row=0, column=0, sticky="nsew")
        ys.grid(row=0, column=1, sticky="ns")
        xs.grid(row=1, column=0, sticky="ew")
        frame.grid_rowconfigure(0, weight=1)
        frame.grid_columnconfigure(0, weight=1)
        return tree

    def nav_bar(self):
        nav = tk.Frame(self.content, bg=self.theme['bg'])
        nav.pack(fill="x", pady=(0, 12))
        items = [
            ("Dashboard", self.dashboard),
            ("Manage Rooms", self.rooms_page),
            ("Customers", self.customers_page),
            ("Bookings", self.bookings_page),
            ("Check Out", self.checkout_page),
            ("Logout", self.logout)
        ]
        for label, action in items:
            color = self.RED if label == "Logout" else self.BLUE
            self.button(nav, label, action, color).pack(side="left", padx=(0, 6))

    # Login and signup
    def login_page(self):
        self.auth_page = 'login_page'
        self.clear()
        self.current_user = None
        outer = tk.Frame(self.content, bg=self.theme['bg'])
        outer.pack(expand=True)
        card = tk.Frame(outer, bg=self.theme['panel'], padx=38, pady=24,
                        highlightbackground=self.theme['border'], highlightthickness=1)
        card.pack()

        tk.Label(card, text="RITA", font=("Arial", 29, "bold"),
                 bg=self.theme['panel'], fg=self.theme['fg']).pack()
        tk.Label(card, text="HOTEL MANAGEMENT SYSTEM",
                 font=("Arial", 10, "bold"), bg=self.theme['panel'],
                 fg="#64748b").pack(pady=(0, 17))
        tk.Label(card, text="Welcome Back", font=("Arial", 19, "bold"),
                 bg=self.theme['panel'], fg=self.theme['fg']).pack(pady=(0, 15))

        tk.Label(card, text="Username", bg=self.theme['panel'], anchor="w").pack(fill="x")
        username = ttk.Entry(card, width=35)
        username.pack(pady=(4, 11), ipady=4)
        tk.Label(card, text="Password", bg=self.theme['panel'], anchor="w").pack(fill="x")
        password = ttk.Entry(card, width=35, show="*")
        password.pack(pady=(4, 16), ipady=4)

        def login():
            user, pwd = username.get().strip(), password.get()
            if not user or not pwd:
                messagebox.showwarning("Required", "Enter username and password.")
                return
            try:
                rows = self.query(
                    "SELECT password_hash FROM users WHERE username=%s",
                    (user,), fetch=True
                )
                if rows and check_password(pwd, rows[0][0]):
                    self.current_user = user
                    self.dashboard()
                else:
                    messagebox.showerror("Login Failed",
                                         "Invalid username or password.")
            except Error as error:
                self.show_error(error)

        self.button(card, "LOGIN", login).pack(fill="x", pady=(0, 12))
        tk.Label(card, text="Don't have an account?", bg=self.theme['panel'],
                 fg=self.theme['muted'], font=("Arial", 9)).pack(pady=(0, 7))
        self.button(card, "CREATE NEW ACCOUNT", self.signup_page,
                    self.GREEN).pack(fill="x")
        username.focus_set()
        password.bind("<Return>", lambda event: login())

    def signup_page(self):
        self.auth_page = 'signup_page'
        self.clear()
        outer = tk.Frame(self.content, bg=self.theme['bg'])
        outer.pack(expand=True)
        card = tk.Frame(outer, bg=self.theme['panel'], padx=38, pady=22,
                        highlightbackground=self.theme['border'], highlightthickness=1)
        card.pack()

        tk.Label(card, text="CREATE ACCOUNT", font=("Arial", 21, "bold"),
                 bg=self.theme['panel'], fg=self.theme['fg']).pack(pady=(0, 4))
        tk.Label(card, text="Register to access the hotel system",
                 bg=self.theme['panel'], fg=self.theme['muted']).pack(pady=(0, 15))

        tk.Label(card, text="Username", bg=self.theme['panel'], anchor="w").pack(fill="x")
        username = ttk.Entry(card, width=35)
        username.pack(pady=(4, 10), ipady=4)
        tk.Label(card, text="Password (minimum 8 characters)",
                 bg=self.theme['panel'], anchor="w").pack(fill="x")
        password = ttk.Entry(card, width=35, show="*")
        password.pack(pady=(4, 10), ipady=4)
        tk.Label(card, text="Confirm Password", bg=self.theme['panel'],
                 anchor="w").pack(fill="x")
        confirm = ttk.Entry(card, width=35, show="*")
        confirm.pack(pady=(4, 15), ipady=4)

        def signup():
            user = username.get().strip()
            pwd = password.get()
            confirm_pwd = confirm.get()
            if not user or not pwd or not confirm_pwd:
                messagebox.showwarning("Required", "Fill in all fields.")
                return
            if len(user) > 100:
                messagebox.showwarning("Username", "Username is too long.")
                return
            if len(pwd) < 8:
                messagebox.showwarning(
                    "Password", "Password must be at least 8 characters."
                )
                return
            if pwd != confirm_pwd:
                messagebox.showerror("Password", "Passwords do not match.")
                return
            try:
                self.query(
                    "INSERT INTO users (username, password_hash) VALUES (%s, %s)",
                    (user, make_password_hash(pwd))
                )
                messagebox.showinfo("Success", "Account created. Please log in.")
                self.login_page()
            except IntegrityError:
                messagebox.showerror("Username Exists",
                                     "That username is already registered.")
            except Error as error:
                self.show_error(error)

        self.button(card, "SIGN UP", signup, self.GREEN).pack(fill="x", pady=(0, 10))
        self.button(card, "BACK TO LOGIN", self.login_page).pack(fill="x")

    def logout(self):
        if messagebox.askyesno("Logout", "Are you sure you want to log out?"):
            self.login_page()

    # Dashboard
    def dashboard(self):
        self.current_page = 'dashboard'
        self.clear()
        self.heading("Dashboard")
        self.nav_bar()
        try:
            stats = [
                ("Total Rooms", self.query("SELECT COUNT(*) FROM rooms", fetch=True)[0][0], self.BLUE),
                ("Available Rooms", self.query("SELECT COUNT(*) FROM rooms WHERE status='Available'", fetch=True)[0][0], self.GREEN),
                ("Customers", self.query("SELECT COUNT(*) FROM customers", fetch=True)[0][0], "#7c3aed"),
                ("Active Bookings", self.query("SELECT COUNT(*) FROM bookings WHERE status='Booked'", fetch=True)[0][0], "#c2410c")
            ]
            cards = tk.Frame(self.content, bg=self.theme['bg'])
            cards.pack(fill="x", pady=12)
            for title, value, color in stats:
                card = tk.Frame(cards, bg=self.theme['panel'], highlightbackground=self.theme['border'],
                                highlightthickness=1)
                card.pack(side="left", expand=True, fill="both", padx=5, ipady=8)
                tk.Label(card, text=title, bg=self.theme['panel'], fg=self.theme['muted'],
                         font=("Arial", 10)).pack(pady=(12, 4))
                tk.Label(card, text=str(value), bg=self.theme['panel'], fg=color,
                         font=("Arial", 25, "bold")).pack(pady=(0, 12))
            tk.Label(self.content, text=f"Welcome, {self.current_user}!",
                     bg=self.theme['bg'], fg=self.theme['fg'],
                     font=("Arial", 19, "bold")).pack(anchor="w", pady=(22, 7))
            tk.Label(self.content,
                     text="Manage rooms, register customers, create bookings "
                          "and process check-outs from the menu above.",
                     bg=self.theme['bg'], fg=self.theme['muted'], font=("Arial", 11),
                     wraplength=900, justify="left").pack(anchor="w")
        except Error as error:
            self.show_error(error)

    # Room management
    def rooms_page(self):
        self.current_page = 'rooms_page'
        self.clear()
        self.heading("Room Management")
        self.nav_bar()
        form = tk.Frame(self.content, bg=self.theme['bg'])
        form.pack(fill="x", pady=4)
        fields = {}
        for i, label in enumerate(["Room Number", "Room Type", "Price"]):
            tk.Label(form, text=label, bg=self.theme['bg'], fg=self.theme['fg']).grid(
                row=0, column=i, padx=7, sticky="w")
            entry = ttk.Entry(form, width=22)
            entry.grid(row=1, column=i, padx=7, pady=5)
            fields[label] = entry
        tree = self.make_table(("room_id", "room_number", "room_type", "price", "status"))

        def load_rooms():
            try:
                tree.delete(*tree.get_children())
                rows = self.query(
                    "SELECT room_id, room_number, room_type, price, status "
                    "FROM rooms ORDER BY room_number", fetch=True)
                for row in rows:
                    tree.insert("", "end", values=row)
            except Error as error:
                self.show_error(error)

        def add_room():
            try:
                number = fields["Room Number"].get().strip()
                room_type = fields["Room Type"].get().strip()
                price_text = fields["Price"].get().strip()
                if not number or not room_type or not price_text:
                    raise ValueError("Fill in all room fields.")
                price = float(price_text)
                if price <= 0:
                    raise ValueError("Price must be greater than zero.")
                self.query(
                    "INSERT INTO rooms (room_number, room_type, price) VALUES (%s, %s, %s)",
                    (number, room_type, price))
                messagebox.showinfo("Success", "Room added successfully.")
                for entry in fields.values():
                    entry.delete(0, tk.END)
                load_rooms()
            except (ValueError, Error) as error:
                self.show_error(error)

        controls = tk.Frame(self.content, bg=self.theme['bg'])
        controls.pack(fill="x")
        self.button(controls, "Add Room", add_room, self.GREEN).pack(side="left", padx=5)
        self.button(controls, "Refresh", load_rooms).pack(side="left", padx=5)
        load_rooms()

    # Customer management
    def customers_page(self):
        self.current_page = 'customers_page'
        self.clear()
        self.heading("Customer Registration")
        self.nav_bar()
        form = tk.Frame(self.content, bg=self.theme['bg'])
        form.pack(fill="x", pady=4)
        fields = {}
        for i, label in enumerate(["Full Name", "Phone", "Email", "Address"]):
            tk.Label(form, text=label, bg=self.theme['bg'], fg=self.theme['fg']).grid(
                row=0, column=i, padx=5, sticky="w")
            entry = ttk.Entry(form, width=22)
            entry.grid(row=1, column=i, padx=5, pady=5)
            fields[label] = entry
        tree = self.make_table(("customer_id", "name", "phone", "email", "address"))

        def load_customers():
            try:
                tree.delete(*tree.get_children())
                rows = self.query(
                    "SELECT customer_id, name, phone, email, address "
                    "FROM customers ORDER BY customer_id DESC", fetch=True)
                for row in rows:
                    tree.insert("", "end", values=row)
            except Error as error:
                self.show_error(error)

        def add_customer():
            name = fields["Full Name"].get().strip()
            phone = fields["Phone"].get().strip()
            email = fields["Email"].get().strip()
            address = fields["Address"].get().strip()
            if not name or not phone:
                messagebox.showwarning("Required", "Name and phone are required.")
                return
            try:
                self.query(
                    "INSERT INTO customers (name, phone, email, address) VALUES (%s, %s, %s, %s)",
                    (name, phone, email, address))
                messagebox.showinfo("Success", "Customer registered successfully.")
                for entry in fields.values():
                    entry.delete(0, tk.END)
                load_customers()
            except Error as error:
                self.show_error(error)

        controls = tk.Frame(self.content, bg=self.theme['bg'])
        controls.pack(fill="x")
        self.button(controls, "Register Customer", add_customer, self.GREEN).pack(side="left", padx=5)
        self.button(controls, "Refresh", load_customers).pack(side="left", padx=5)
        load_customers()

    # Booking management
    def bookings_page(self):
        self.current_page = 'bookings_page'
        self.clear()
        self.heading("Room Booking")
        self.nav_bar()
        form = tk.Frame(self.content, bg=self.theme['bg'])
        form.pack(fill="x", pady=4)
        labels = ["Customer ID", "Room ID", "Check-in (YYYY-MM-DD)", "Check-out (YYYY-MM-DD)"]
        fields = {}
        for i, label in enumerate(labels):
            tk.Label(form, text=label, bg=self.theme['bg'], fg=self.theme['fg']).grid(
                row=0, column=i, padx=5, sticky="w")
            entry = ttk.Entry(form, width=23)
            entry.grid(row=1, column=i, padx=5, pady=5)
            fields[label] = entry
        tree = self.make_table(("booking_id", "customer", "room", "check_in", "check_out", "total_amount", "status"))
        booking_sql = """
            SELECT b.booking_id, c.name, r.room_number, b.check_in,
                   b.check_out, b.total_amount, b.status
            FROM bookings b
            JOIN customers c ON b.customer_id=c.customer_id
            JOIN rooms r ON b.room_id=r.room_id
        """

        def load_bookings():
            try:
                tree.delete(*tree.get_children())
                for row in self.query(booking_sql + " ORDER BY b.booking_id DESC", fetch=True):
                    tree.insert("", "end", values=row)
            except Error as error:
                self.show_error(error)

        def book_room():
            conn = cur = None
            try:
                customer_id = int(fields["Customer ID"].get())
                room_id = int(fields["Room ID"].get())
                check_in = date.fromisoformat(fields["Check-in (YYYY-MM-DD)"].get().strip())
                check_out = date.fromisoformat(fields["Check-out (YYYY-MM-DD)"].get().strip())
                if check_in < date.today():
                    raise ValueError("Check-in date cannot be in the past.")
                if check_out <= check_in:
                    raise ValueError("Check-out must be after check-in.")

                conn = connect_db()
                cur = conn.cursor()
                cur.execute("SELECT price FROM rooms WHERE room_id=%s AND status='Available' FOR UPDATE", (room_id,))
                room = cur.fetchone()
                if room is None:
                    raise ValueError("Room ID not found or room is unavailable.")
                cur.execute("""
                    SELECT COUNT(*) FROM bookings
                    WHERE room_id=%s AND status='Booked'
                    AND check_in < %s AND check_out > %s
                """, (room_id, check_out, check_in))
                if cur.fetchone()[0] > 0:
                    raise ValueError("Room is already booked for those dates.")
                cur.execute("SELECT customer_id FROM customers WHERE customer_id=%s", (customer_id,))
                if cur.fetchone() is None:
                    raise ValueError("Customer ID does not exist.")

                nights = (check_out - check_in).days
                total = float(room[0]) * nights
                cur.execute("""
                    INSERT INTO bookings
                    (customer_id, room_id, check_in, check_out, total_amount, status)
                    VALUES (%s, %s, %s, %s, %s, 'Booked')
                """, (customer_id, room_id, check_in, check_out, total))
                conn.commit()
                messagebox.showinfo(
                    "Booking Confirmed",
                    f"Booking created successfully!\nNights: {nights}\nTotal: Rs. {total:,.2f}")
                for entry in fields.values():
                    entry.delete(0, tk.END)
                load_bookings()
            except (ValueError, Error) as error:
                if conn is not None:
                    conn.rollback()
                self.show_error(error)
            finally:
                if cur is not None:
                    cur.close()
                if conn is not None:
                    conn.close()

        def cancel_booking():
            selected = tree.selection()
            if not selected:
                messagebox.showwarning("Select Booking", "Select a booking first.")
                return
            row = tree.item(selected[0])["values"]
            booking_id, status = row[0], row[6]
            if status != "Booked":
                messagebox.showwarning("Cannot Cancel", "Only active bookings can be cancelled.")
                return
            if not messagebox.askyesno("Confirm", f"Cancel booking #{booking_id}?"):
                return
            try:
                self.query("UPDATE bookings SET status='Cancelled' WHERE booking_id=%s AND status='Booked'", (booking_id,))
                messagebox.showinfo("Success", "Booking cancelled.")
                load_bookings()
            except Error as error:
                self.show_error(error)

        controls = tk.Frame(self.content, bg=self.theme['bg'])
        controls.pack(fill="x", pady=5)
        self.button(controls, "Book Room", book_room, self.GREEN).pack(side="left", padx=5)
        self.button(controls, "Cancel Selected Booking", cancel_booking, self.RED).pack(side="left", padx=5)
        self.button(controls, "Refresh", load_bookings).pack(side="left", padx=5)
        load_bookings()

    # Check-out management
    def checkout_page(self):
        self.current_page = 'checkout_page'
        self.clear()
        self.heading("Check-out Management")
        self.nav_bar()
        tk.Label(self.content, text="Select an active booking and confirm check-out.",
                 bg=self.theme['bg'], fg=self.theme['muted'], font=("Arial", 11)).pack(anchor="w", pady=4)
        tree = self.make_table(("booking_id", "customer", "room", "check_in", "check_out", "total_amount", "status"))

        def load_bookings():
            try:
                tree.delete(*tree.get_children())
                sql = """
                    SELECT b.booking_id, c.name, r.room_number, b.check_in,
                           b.check_out, b.total_amount, b.status
                    FROM bookings b
                    JOIN customers c ON b.customer_id=c.customer_id
                    JOIN rooms r ON b.room_id=r.room_id
                    WHERE b.status='Booked'
                    ORDER BY b.booking_id DESC
                """
                for row in self.query(sql, fetch=True):
                    tree.insert("", "end", values=row)
            except Error as error:
                self.show_error(error)

        def checkout():
            selected = tree.selection()
            if not selected:
                messagebox.showwarning("Select Booking", "Select an active booking first.")
                return
            row = tree.item(selected[0])["values"]
            booking_id, room_number = row[0], row[2]
            if not messagebox.askyesno("Confirm Check-out",
                                       f"Check out room {room_number} for booking #{booking_id}?"):
                return
            try:
                changed = self.query(
                    "UPDATE bookings SET status='Checked Out' WHERE booking_id=%s AND status='Booked'",
                    (booking_id,))
                if changed:
                    messagebox.showinfo("Check-out Complete", f"Room {room_number} checked out successfully.")
                else:
                    messagebox.showwarning("Already Processed", "This booking is no longer active.")
                load_bookings()
            except Error as error:
                self.show_error(error)

        controls = tk.Frame(self.content, bg=self.theme['bg'])
        controls.pack(fill="x", pady=5)
        self.button(controls, "Confirm Check-out", checkout, self.GREEN).pack(side="left", padx=5)
        self.button(controls, "Refresh", load_bookings).pack(side="left", padx=5)
        load_bookings()


if __name__ == "__main__":
    root = tk.Tk()
    app = HotelManagementSystem(root)
    root.mainloop()
