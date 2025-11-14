# Simple SQL Injection Demo - PostgreSQL

## ⚠️ WARNING
This is for educational purposes only. Never use in production!

## Architecture (Super Simple!)

```
EC2 Instance with PostgreSQL + Python Script
```

---

## Step 1: Launch EC2 Instance

1. Go to AWS Console → EC2
2. Click "Launch Instance"
3. Choose:
   - **Ubuntu Server 22.04 LTS**
   - **t2.micro** (free tier)
   - Create or select key pair
   - Security Group: Allow SSH (port 22) from your IP
4. Launch and connect:

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@YOUR_EC2_IP
```

---

## Step 2: Install PostgreSQL and Python

```bash
# Update system
sudo apt update

# Install PostgreSQL and Python
sudo apt install -y postgresql postgresql-contrib python3 python3-pip

# Install Python PostgreSQL library
pip3 install psycopg2-binary
```

---

## Step 3: Configure PostgreSQL

```bash
# Switch to postgres user
sudo -u postgres psql

# Run these SQL commands:
```

```sql
-- Create database
CREATE DATABASE testdb;

-- Create user
CREATE USER testuser WITH PASSWORD 'testpass123';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE testdb TO testuser;

-- Connect to testdb
\c testdb

-- Create table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Grant table privileges
GRANT ALL PRIVILEGES ON TABLE users TO testuser;
GRANT USAGE, SELECT ON SEQUENCE users_id_seq TO testuser;

-- Insert sample data
INSERT INTO users (username, email) VALUES 
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com');

-- Verify
SELECT * FROM users;

-- Exit
\q
```

---

## Step 4: Create Vulnerable Python Script

```bash
# Create the vulnerable script
cat > vulnerable_app.py << 'EOF'
#!/usr/bin/env python3
import psycopg2

def get_connection():
    """Connect to PostgreSQL"""
    return psycopg2.connect(
        host="localhost",
        database="testdb",
        user="testuser",
        password="testpass123"
    )

def vulnerable_insert(username, email):
    """VULNERABLE: Direct string interpolation - SQL Injection!"""
    conn = get_connection()
    cursor = conn.cursor()
    
    # DANGEROUS: Building SQL with string formatting
    query = f"INSERT INTO users (username, email) VALUES ('{username}', '{email}')"
    
    print(f"\n🔥 EXECUTING QUERY: {query}\n")
    
    try:
        cursor.execute(query)
        conn.commit()
        print("✅ Insert successful!")
        return True
    except Exception as e:
        print(f"❌ Error: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()

def safe_insert(username, email):
    """SAFE: Using parameterized queries"""
    conn = get_connection()
    cursor = conn.cursor()
    
    # SAFE: Using parameterized query
    query = "INSERT INTO users (username, email) VALUES (%s, %s)"
    
    print(f"\n✅ EXECUTING SAFE QUERY: {query}")
    print(f"   Parameters: ({username}, {email})\n")
    
    try:
        cursor.execute(query, (username, email))
        conn.commit()
        print("✅ Insert successful!")
        return True
    except Exception as e:
        print(f"❌ Error: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()

def show_all_users():
    """Display all users"""
    conn = get_connection()
    cursor = conn.cursor()
    
    cursor.execute("SELECT * FROM users ORDER BY id")
    rows = cursor.fetchall()
    
    print("\n📋 CURRENT USERS:")
    print("-" * 60)
    for row in rows:
        print(f"ID: {row[0]}, Username: {row[1]}, Email: {row[2]}")
    print("-" * 60 + "\n")
    
    cursor.close()
    conn.close()

def clear_test_data():
    """Clear test data (keep original users)"""
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM users WHERE id > 2")
    conn.commit()
    cursor.close()
    conn.close()
    print("🧹 Test data cleared!\n")

def main():
    print("=" * 60)
    print("SQL INJECTION DEMONSTRATION")
    print("=" * 60)
    
    while True:
        print("\nChoose an option:")
        print("1. Vulnerable Insert (SQL Injection Demo)")
        print("2. Safe Insert (Parameterized Query)")
        print("3. Show All Users")
        print("4. Clear Test Data")
        print("5. Exit")
        
        choice = input("\nEnter choice (1-5): ").strip()
        
        if choice == "1":
            print("\n🔥 VULNERABLE INSERT MODE 🔥")
            username = input("Enter username: ")
            email = input("Enter email: ")
            vulnerable_insert(username, email)
            show_all_users()
            
        elif choice == "2":
            print("\n✅ SAFE INSERT MODE ✅")
            username = input("Enter username: ")
            email = input("Enter email: ")
            safe_insert(username, email)
            show_all_users()
            
        elif choice == "3":
            show_all_users()
            
        elif choice == "4":
            clear_test_data()
            show_all_users()
            
        elif choice == "5":
            print("Goodbye!")
            break
        else:
            print("Invalid choice!")

if __name__ == "__main__":
    main()
EOF

# Make it executable
chmod +x vulnerable_app.py
```

---

## Step 5: Test SQL Injection!

```bash
# Run the script
python3 vulnerable_app.py
```

### Try These Attacks:

**Test 1: Normal Insert**
- Choose option `1` (Vulnerable)
- Username: `charlie`
- Email: `charlie@example.com`
- ✅ Works normally

**Test 2: Multi-Record Injection**
- Choose option `1` (Vulnerable)
- Username: `test'), ('hacker', 'hacker@evil.com'); --`
- Email: `whatever@example.com`
- 💥 Inserts TWO records instead of one!

**Test 3: SQL Comment Attack**
- Choose option `1` (Vulnerable)
- Username: `admin'; --`
- Email: (anything)
- 💥 SQL injection with comment

**Test 4: UPDATE Attack**
- Choose option `1` (Vulnerable)
- Username: `test'); UPDATE users SET email='HACKED@evil.com' WHERE id=1; --`
- Email: `whatever@example.com`
- 💥 Modifies existing data!

**Test 5: Safe Version**
- Choose option `2` (Safe)
- Try the same attacks
- ✅ They get stored as literal text, no SQL execution!

---

## Example Session:

```
$ python3 vulnerable_app.py

============================================================
SQL INJECTION DEMONSTRATION
============================================================

Choose an option:
1. Vulnerable Insert (SQL Injection Demo)
2. Safe Insert (Parameterized Query)
3. Show All Users
4. Clear Test Data
5. Exit

Enter choice (1-5): 1

🔥 VULNERABLE INSERT MODE 🔥
Enter username: test'), ('attacker', 'evil@hacker.com'); --
Enter email: ignored@example.com

🔥 EXECUTING QUERY: INSERT INTO users (username, email) VALUES ('test'), ('attacker', 'evil@hacker.com'); --', 'ignored@example.com')

✅ Insert successful!

📋 CURRENT USERS:
------------------------------------------------------------
ID: 1, Username: alice, Email: alice@example.com
ID: 2, Username: bob, Email: bob@example.com
ID: 3, Username: test, Email: evil@hacker.com
ID: 4, Username: attacker, Email: evil@hacker.com
------------------------------------------------------------
```

**Notice:** Two records were inserted instead of one! 💥

---

## Understanding the Vulnerability

### Vulnerable Code:
```python
query = f"INSERT INTO users (username, email) VALUES ('{username}', '{email}')"
cursor.execute(query)
```

When username = `test'), ('hacker', 'evil@hacker.com'); --`

The resulting SQL becomes:
```sql
INSERT INTO users (username, email) VALUES ('test'), ('hacker', 'evil@hacker.com'); --', 'ignored@example.com')
```

The `--` comments out the rest!

### Safe Code:
```python
query = "INSERT INTO users (username, email) VALUES (%s, %s)"
cursor.execute(query, (username, email))
```

The database treats user input as DATA, not CODE! ✅

---

## More Attack Examples to Try:

1. **Drop Table (Dangerous!):**
   ```
   Username: test'); DROP TABLE users; --
   ```

2. **Information Extraction:**
   ```
   Username: test'), ((SELECT COUNT(*) FROM pg_tables), 'info@x.com'); --
   ```

3. **Multiple Statements:**
   ```
   Username: test'); UPDATE users SET username='PWNED'; --
   ```

---

## Cleanup

```bash
# Stop the script (Ctrl+C)

# Optional: Remove everything
sudo systemctl stop postgresql
sudo apt remove --purge postgresql postgresql-contrib
```

Or terminate the EC2 instance when done.

---

## Key Takeaways

1. **Never use string interpolation for SQL queries**
2. **Always use parameterized queries** (cursor.execute with parameters)
3. **Input validation alone is not enough** - use parameterized queries!
4. **Test your code with malicious input**

## Total Time: ~15 minutes! 🚀

Much simpler than Django/RDS setup!
