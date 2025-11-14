# Guía de Despliegue Simple - Una Sola Instancia EC2

## ⚡ Todo en una instancia: Django + MySQL + Demo SQL Injection

Esta guía te permite desplegar todo el sistema en una sola instancia EC2 de Ubuntu.

---

## 📋 Paso 1: Crear Instancia EC2

### En AWS Console:
1. **Ir a EC2** → Launch Instance
2. **Nombre**: `django-sql-injection-demo`
3. **AMI**: Ubuntu Server 22.04 LTS (Free tier)
4. **Tipo**: t2.small (mínimo recomendado) o t2.micro si tienes limitaciones
5. **Key pair**: Crear o seleccionar una existente
6. **Security Group**: Configurar reglas:
   - SSH (22) - Tu IP
   - HTTP (80) - 0.0.0.0/0
   - Custom TCP (8000) - 0.0.0.0/0 (para Django dev server)
   - MySQL (3306) - Solo si necesitas acceso externo (opcional)

7. **Launch Instance**

8. **Anotar**: La IP pública de tu instancia (ej: `54.123.45.67`)

---

## 🔌 Paso 2: Conectar a la Instancia

```bash
# Desde tu Mac terminal
chmod 400 tu-key.pem
ssh -i tu-key.pem ubuntu@TU_IP_PUBLICA
```

---

## 📦 Paso 3: Instalar Dependencias

```bash
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar Python y pip
sudo apt install -y python3 python3-pip python3-venv

# Instalar MySQL Server
sudo apt install -y mysql-server

# Instalar Git
sudo apt install -y git

# Verificar instalaciones
python3 --version
mysql --version
git --version
```

---

## 🔐 Paso 4: Configurar MySQL

```bash
# Iniciar servicio MySQL
sudo systemctl start mysql
sudo systemctl enable mysql

# Verificar que está corriendo
sudo systemctl status mysql

# Configurar MySQL (asegurar instalación)
sudo mysql_secure_installation
```

Responder a las preguntas:
- **VALIDATE PASSWORD component**: `n` (no)
- **Remove anonymous users**: `y`
- **Disallow root login remotely**: `y`
- **Remove test database**: `y`
- **Reload privilege tables**: `y`

### Crear Base de Datos y Usuario:

```bash
# Entrar a MySQL como root
sudo mysql

# Dentro de MySQL, ejecutar:
```

```sql
-- Crear base de datos
CREATE DATABASE rutasbodega CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Crear usuario
CREATE USER 'django_user'@'localhost' IDENTIFIED BY 'django123';

-- Dar permisos
GRANT ALL PRIVILEGES ON rutasbodega.* TO 'django_user'@'localhost';

-- Aplicar cambios
FLUSH PRIVILEGES;

-- Verificar
SHOW DATABASES;
SELECT user, host FROM mysql.user;

-- Salir
EXIT;
```

### Probar Conexión:

```bash
# Probar que el usuario puede conectar
mysql -u django_user -p
# Contraseña: django123

# Dentro de MySQL:
USE rutasbodega;

-- Crear la tabla de objetos
CREATE TABLE IF NOT EXISTS objetos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(50) UNIQUE NOT NULL,
    descripcion VARCHAR(10) NOT NULL,
    ubicacion VARCHAR(20) NOT NULL,
    activo BOOLEAN DEFAULT TRUE,
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Crear la tabla de consultas
CREATE TABLE IF NOT EXISTS consultas_rutas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objeto_origen VARCHAR(50) NOT NULL,
    objeto_destino VARCHAR(50) NOT NULL,
    ruta_resultado VARCHAR(100) NOT NULL,
    tiempo_frontend DECIMAL(10,2),
    tiempo_backend DECIMAL(10,2) NOT NULL,
    tiempo_aws_obj1 DECIMAL(10,2) DEFAULT 0,
    tiempo_aws_obj2 DECIMAL(10,2) DEFAULT 0,
    tiempo_concatenacion DECIMAL(10,2) DEFAULT 0,
    ip_cliente VARCHAR(45),
    fecha_consulta TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_fecha (fecha_consulta),
    INDEX idx_objetos (objeto_origen, objeto_destino)
);

-- Insertar datos iniciales de prueba
INSERT INTO objetos (nombre, descripcion, ubicacion) VALUES
    ('zapatos', 'Z', 'A1-B1'),
    ('caja', 'C', 'A2-B1'),
    ('libro', 'L', 'A3-B1'),
    ('mesa', 'M', 'A4-B1'),
    ('silla', 'S', 'A5-B1'),
    ('computadora', 'CO', 'A6-B1'),
    ('telefono', 'T', 'A7-B1'),
    ('reloj', 'R', 'A8-B1');

-- Verificar que se crearon correctamente
SHOW TABLES;
SELECT COUNT(*) FROM objetos;

EXIT;
```

---

## 📁 Paso 5: Clonar y Configurar Proyecto

```bash
# Ir al directorio home
cd ~

# Clonar tu repositorio
git clone https://github.com/jrpinto2005/Arquisoft.git
cd Arquisoft/casoArquisoft

# Cambiar a la rama de inyección
git checkout inyeccion

# Crear entorno virtual
python3 -m venv venv

# Activar entorno virtual
source venv/bin/activate

# Actualizar pip
pip install --upgrade pip

# Instalar dependencias del proyecto
pip install -r requirements.txt
```

---

## ⚙️ Paso 6: Configurar Django

```bash
# Asegurarse de estar en el directorio del proyecto
cd ~/Arquisoft/casoArquisoft

# Hacer migraciones (aunque usamos MySQL directo, Django necesita esto)
python3 manage.py makemigrations
python3 manage.py migrate

# Crear superusuario (opcional)
python3 manage.py createsuperuser
# Usuario: admin
# Email: admin@test.com
# Password: admin123
```

---

## 🚀 Paso 7: Iniciar el Servidor

### Opción A: Servidor de Desarrollo (Más simple)

```bash
# Activar entorno virtual si no está activado
source ~/Arquisoft/casoArquisoft/venv/bin/activate

# Ir al directorio del proyecto
cd ~/Arquisoft/casoArquisoft

# Iniciar servidor en todas las interfaces (0.0.0.0)
python3 manage.py runserver 0.0.0.0:8000
```

**Acceder desde tu navegador**:
- `http://TU_IP_PUBLICA:8000/`
- `http://TU_IP_PUBLICA:8000/sql_demo/` (Demo SQL Injection)

### Opción B: Servidor en Background (tmux)

```bash
# Instalar tmux
sudo apt install -y tmux

# Crear sesión tmux
tmux new -s django

# Dentro de tmux:
source ~/Arquisoft/casoArquisoft/venv/bin/activate
cd ~/Arquisoft/casoArquisoft
python3 manage.py runserver 0.0.0.0:8000

# Salir de tmux (servidor sigue corriendo): Ctrl+B, luego D

# Para volver a la sesión:
tmux attach -t django

# Para matar la sesión:
tmux kill-session -t django
```

---

## 🧪 Paso 8: Probar SQL Injection

### 1. Acceder a la Demo:
```
http://TU_IP_PUBLICA:8000/sql_demo/
```

### 2. Probar Inserción Normal:
- **Nombre**: `laptop`
- **Descripción**: `L`
- **Ubicación**: `A9-B1`
- Click **Insertar Objeto**

### 3. Probar SQL Injection - Multi-Insert:
- **Nombre**: `test'), ('hacker', 'HACK', 'EVIL'); --`
- **Descripción**: `T`
- **Ubicación**: `A10`
- Click **Insertar Objeto**

### 4. Verificar en la Base de Datos:

```bash
# En la instancia EC2
mysql -u django_user -p
# Password: django123

USE rutasbodega;
SELECT * FROM objetos ORDER BY id DESC LIMIT 10;
```

Deberías ver tanto el registro "test" como "hacker" insertados (2 filas con 1 solo INSERT).

### 5. Otros Ataques a Probar:

**UPDATE Attack** (Modifica registros existentes):
- **Nombre**: `test2`
- **Descripción**: `T`
- **Ubicación**: `A1'); UPDATE objetos SET descripcion='HACKED' WHERE id=1; --`

Verificar: `SELECT * FROM objetos WHERE id=1;` debería mostrar descripcion='HACKED'

**DELETE Attack** (Elimina registros):
- **Nombre**: `test3`
- **Descripción**: `T`
- **Ubicación**: `A1'); DELETE FROM objetos WHERE nombre='zapatos'; --`

Verificar: El objeto 'zapatos' ya no existe

**Multiple UPDATEs** (Caos total):
- **Nombre**: `test4`
- **Descripción**: `T`
- **Ubicación**: `A1'); UPDATE objetos SET activo=0; UPDATE objetos SET ubicacion='PWNED'; --`

Verificar: Todos los objetos ahora tienen activo=0 y ubicacion='PWNED'

**DROP TABLE Attack** (¡Máximo peligro!):
- **Nombre**: `test5`
- **Descripción**: `T`
- **Ubicación**: `A1'); DROP TABLE consultas_rutas; --`

⚠️ CUIDADO: Esto eliminará toda la tabla de consultas

---

## 🔍 Paso 9: Monitorear y Verificar

### Ver Logs del Servidor:
Los logs de Django aparecen en la terminal donde corriste `runserver`.

### Consultar Base de Datos:
```bash
mysql -u django_user -p

USE rutasbodega;

-- Ver todos los objetos
SELECT * FROM objetos;

-- Ver consultas registradas
SELECT * FROM consultas_rutas ORDER BY fecha_consulta DESC LIMIT 10;

-- Contar objetos
SELECT COUNT(*) FROM objetos;

-- Ver objetos sospechosos (con comillas o guiones)
SELECT * FROM objetos WHERE nombre LIKE "%'%" OR nombre LIKE "%--%";
```

---

## 🛑 Paso 10: Detener y Limpiar

### Detener Servidor:
- Si está en foreground: `Ctrl + C`
- Si está en tmux: `tmux kill-session -t django`

### Limpiar Base de Datos:
```bash
mysql -u django_user -p

USE rutasbodega;

-- Eliminar todos los objetos
DELETE FROM objetos;

-- Resetear auto-increment
ALTER TABLE objetos AUTO_INCREMENT = 1;

-- Eliminar consultas
DELETE FROM consultas_rutas;
ALTER TABLE consultas_rutas AUTO_INCREMENT = 1;

EXIT;
```

### Detener MySQL:
```bash
sudo systemctl stop mysql
```

---

## 🔧 Troubleshooting

### Error: "Can't connect to MySQL"
```bash
# Verificar que MySQL está corriendo
sudo systemctl status mysql

# Si no está activo, iniciarlo
sudo systemctl start mysql

# Ver logs de MySQL
sudo tail -f /var/log/mysql/error.log
```

### Error: "Access denied for user 'django_user'"
```bash
# Verificar permisos
sudo mysql

SELECT user, host FROM mysql.user;
SHOW GRANTS FOR 'django_user'@'localhost';

# Recrear usuario si es necesario
DROP USER 'django_user'@'localhost';
CREATE USER 'django_user'@'localhost' IDENTIFIED BY 'django123';
GRANT ALL PRIVILEGES ON rutasbodega.* TO 'django_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Error: "Port 8000 already in use"
```bash
# Encontrar proceso usando el puerto
sudo lsof -i :8000

# Matar proceso
sudo kill -9 PID_DEL_PROCESO
```

### No puedo acceder desde navegador:
1. **Verificar Security Group** en AWS Console
   - Puerto 8000 debe estar abierto
2. **Verificar que el servidor escucha en 0.0.0.0**:
   ```bash
   sudo netstat -tulpn | grep 8000
   ```
3. **Probar desde la instancia**:
   ```bash
   curl http://localhost:8000/
   ```

---

## 📊 Arquitectura Final

```
┌─────────────────────────────────────┐
│     EC2 Ubuntu Instance             │
│  (Una sola máquina)                 │
│                                     │
│  ┌─────────────────────────────┐   │
│  │   Django App (Port 8000)    │   │
│  │   - views.py (vulnerable)   │   │
│  │   - SQL Injection Demo      │   │
│  └──────────┬──────────────────┘   │
│             │                       │
│             ↓                       │
│  ┌─────────────────────────────┐   │
│  │   MySQL Server (Port 3306)  │   │
│  │   - Database: rutasbodega   │   │
│  │   - User: django_user       │   │
│  └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
            ↑
            │ HTTP (Port 8000)
            │
    ┌───────┴────────┐
    │   Internet     │
    │   (Tu Browser) │
    └────────────────┘
```

---

## ⏱️ Tiempo Total de Setup

- **Paso 1-2**: 5 minutos (crear instancia y conectar)
- **Paso 3**: 5 minutos (instalar dependencias)
- **Paso 4**: 5 minutos (configurar MySQL)
- **Paso 5-6**: 5 minutos (clonar y configurar proyecto)
- **Paso 7**: 2 minutos (iniciar servidor)
- **Paso 8**: 5 minutos (probar)

**Total**: ~25-30 minutos

---

## 🎯 URLs Importantes

Una vez desplegado, estas son las URLs principales:

```
http://TU_IP_PUBLICA:8000/                    # Página principal
http://TU_IP_PUBLICA:8000/sql_demo/           # Demo SQL Injection ⚠️
http://TU_IP_PUBLICA:8000/objetos/            # Lista de objetos
http://TU_IP_PUBLICA:8000/buscar/             # Buscar rutas
http://TU_IP_PUBLICA:8000/admin/              # Panel admin Django
```

---

## ⚠️ ADVERTENCIA DE SEGURIDAD

Este servidor es **INTENCIONALMENTE VULNERABLE** para propósitos educativos:

- ✅ Úsalo solo en entornos de prueba
- ✅ Termina la instancia EC2 cuando termines
- ✅ NO expongas datos reales
- ✅ NO uses en producción
- ✅ Mantén el Security Group restringido

---

## 🎓 Para la Demostración

### Script de Demo:

1. **Mostrar página funcionando**: Navegar a `/sql_demo/`
2. **Inserción normal**: Insertar un objeto legítimo
3. **Verificar en BD**: Mostrar en MySQL que se insertó
4. **SQL Injection**: Usar el ataque multi-insert
5. **Verificar resultado**: Mostrar múltiples registros insertados
6. **Explicar**: Mostrar el código vulnerable en `views.py`

---

## 📝 Notas Adicionales

- El servidor de desarrollo de Django (`runserver`) es solo para pruebas
- Para producción real se usaría Gunicorn + Nginx
- Esta configuración es perfecta para demostraciones educativas
- Todos los datos se pierden al terminar la instancia

---

## 🔄 Actualizar Código

Si haces cambios en tu repositorio:

```bash
cd ~/Arquisoft/casoArquisoft
git pull origin inyeccion

# Si cambiaste views.py u otro archivo Python, reinicia el servidor
# (Ctrl+C y volver a correr python3 manage.py runserver)
```

---

¡Listo! Ahora tienes todo en una sola instancia EC2. 🚀
