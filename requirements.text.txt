import streamlit as st
import sqlite3
import pandas as pd
from datetime import datetime

@st.cache_resource
def get_connection():
    conn = sqlite3.connect('book_my_saloon.db', check_same_thread=False)
    return conn

def init_db():
    conn = get_connection()
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        phone TEXT UNIQUE NOT NULL,
        password TEXT NOT NULL,
        role TEXT,
        name TEXT,
        approved INTEGER DEFAULT 0
    )''')
    c.execute('''CREATE TABLE IF NOT EXISTS barbers (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        owner_id INTEGER,
        name TEXT,
        address TEXT,
        services TEXT,
        rating REAL DEFAULT 0,
        approved INTEGER DEFAULT 0
    )''')
    c.execute('''CREATE TABLE IF NOT EXISTS bookings (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        customer_id INTEGER,
        barber_id INTEGER,
        service TEXT,
        date_time TEXT,
        status TEXT DEFAULT 'pending'
    )''')
    c.execute("INSERT OR IGNORE INTO users (phone, password, role, name, approved) VALUES ('admin', 'admin', 'admin', 'Super Admin', 1)")
    conn.commit()

st.set_page_config(page_title="Book My Saloon", layout="wide")
init_db()

if 'user' not in st.session_state:
    st.session_state.user = None

if st.session_state.user is None:
    st.title("👨‍✂️ Book My Saloon")
    st.header("Login")
    col1, col2 = st.columns([1,2])
    with col1:
        role = st.selectbox("Role", ["admin", "barber", "customer"])
    with col2:
        phone = st.text_input("Phone")
        password = st.text_input("Password", type="password")
        if st.button("Login"):
            conn = get_connection()
            df = pd.read_sql("SELECT * FROM users WHERE phone=? AND password=? AND role=? AND approved=1", 
                           conn, params=(phone, password, role))
            if not df.empty:
                st.session_state.user = df.iloc[0].to_dict()
                st.rerun()
            else:
                st.error("❌ Invalid login")
else:
    st.sidebar.title(f"👋 {st.session_state.user['name']}")
    if st.sidebar.button("Logout"):
        st.session_state.user = None
        st.rerun()
    
    conn = get_connection()
    role = st.session_state.user['role']
    
    if role == 'admin':
        st.header("🛡️ Admin Panel")
        tab1, tab2, tab3 = st.tabs(["📋 Barbers", "📅 Bookings", "👥 Users"])
        
        with tab1:
            df = pd.read_sql("SELECT * FROM barbers", conn)
            st.dataframe(df)
            if st.button("Add Test Barber"):
                c = conn.cursor()
                c.execute("INSERT INTO barbers (owner_id, name, address, services, approved) VALUES (1, 'Test Salon', 'Lucknow', 'Haircut,Shave', 1)")
                conn.commit()
                st.rerun()
        
        with tab2:
            df = pd.read_sql("SELECT * FROM bookings", conn)
            st.dataframe(df)
        
        with tab3:
            df = pd.read_sql("SELECT * FROM users", conn)
            st.dataframe(df)
            col1, col2, col3, col4 = st.columns(4)
            with col1:
                phone = st.text_input("Phone")
            with col2:
                pwd = st.text_input("Password", type="password")
            with col3:
                name = st.text_input("Name")
            with col4:
                role_new = st.selectbox("Role", ["customer", "barber"])
                if st.button("Add User"):
                    try:
                        c = conn.cursor()
                        c.execute("INSERT INTO users (phone, password, role, name, approved) VALUES (?,?,?,?,1)", 
                                (phone, pwd, role_new, name))
                        conn.commit()
                        st.success("✅ User Added!")
                        st.rerun()
                    except:
                        st.error("❌ Phone exists!")
    
    elif role == 'barber':
        st.header("💈 Barber Panel")
        df = pd.read_sql("SELECT * FROM barbers WHERE owner_id=?", conn, params=(st.session_state.user['id'],))
        if df.empty:
            st.info("Create your salon profile")
            with st.form("profile"):
                name = st.text_input("Salon Name")
                addr = st.text_area("Address")
                serv = st.text_area("Services")
                if st.form_submit_button("Save"):
                    c = conn.cursor()
                    c.execute("INSERT INTO barbers (owner_id, name, address, services) VALUES (?,?,?,?)", 
                            (st.session_state.user['id'], name, addr, serv))
                    conn.commit()
                    st.success("✅ Profile Created!")
                    st.rerun()
        else:
            st.success("✅ Profile Ready!")
            st.write(df.iloc[0])
    
    elif role == 'customer':
        st.header("👤 Customer Panel")
        df = pd.read_sql("SELECT * FROM barbers WHERE approved=1", conn)
        if df.empty:
            st.info("No salons available")
        else:
            for idx, row in df.iterrows():
                with st.expander(row['name']):
                    st.write(row['address'])
                    if st.button(f"Book {row['name']}", key=idx):
                        c = conn.cursor()
                        c.execute("INSERT INTO bookings (customer_id, barber_id, service, date_time) VALUES (?,?,?,'Now')", 
                                (st.session_state.user['id'], row['id'], 'Haircut'))
                        conn.commit()
                        st.success("✅ Booked!")
