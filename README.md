# Práctica Calificada: Despliegue de Aplicaciones Dockerizadas en AWS bajo Arquitectura de 3 Capas con SQL Server

## 👤 Integrante

**Ebert Bernardo Ocares Luna**  
Área Académica de Análisis de Sistemas  
Versión: 1.0.0 — Mayo 2026

---

## 📋 Descripción del Proyecto

Aplicación web fullstack desplegada en AWS utilizando una arquitectura de 3 capas con contenedores Docker. El sistema permite registrar y consultar información a través de una interfaz web Angular que se comunica con una API REST Spring Boot, la cual se conecta a una base de datos SQL Server, todo comunicado mediante IP privada dentro de AWS.

La arquitectura garantiza que:
- El **Frontend** es el único servicio accesible desde Internet (IP pública).
- El **Backend** solo es accesible desde el Frontend mediante IP privada.
- La **Base de Datos** no tiene ningún acceso público.

---

## 🏗️ Arquitectura

```
Usuario (Internet)
       │
       │ IP Pública (HTTP 80)
       ▼
┌─────────────────────┐
│   EC2-Frontend      │  Security Group: 80, 443, 22
│   Angular + Nginx   │  Docker → ninahuancaromani/angular-frontend
└─────────┬───────────┘
          │ IP Privada (proxy /api → :8080)
          ▼
┌─────────────────────┐
│   EC2-Backend       │  Security Group: 8080, 22 (solo desde Frontend)
│   Spring Boot       │  Docker → ninahuancaromani/spring-backend
└─────────┬───────────┘
          │ IP Privada (puerto 1433)
          ▼
┌─────────────────────┐
│   EC2-DB            │  Security Group: 1433, 22 (solo desde Backend)
│   SQL Server 2022   │  Docker → mcr.microsoft.com/mssql/server:2022-latest
└─────────────────────┘
```

---

## 🛠️ Tecnologías Utilizadas

| Capa | Tecnología | Acceso |
|------|-----------|--------|
| Frontend | Angular + Nginx (Docker) | Público |
| Backend | Spring Boot / Java 17 (Docker) | Privado |
| Base de Datos | SQL Server 2022 (Docker) | Privado |
| Infraestructura | AWS EC2, Security Groups, IP Elástica | — |
| Contenedores | Docker | — |
| Repositorio | GitHub + Docker Hub | — |

---

## ⚙️ Configuración de Red (Security Groups)

### EC2-Frontend
| Puerto | Protocolo | Origen |
|--------|-----------|--------|
| 80 | TCP | 0.0.0.0/0 |
| 22 | TCP | 0.0.0.0/0 |

### EC2-Backend
| Puerto | Protocolo | Origen |
|--------|-----------|--------|
| 8080 | TCP | IP privada EC2-Frontend |
| 22 | TCP | 0.0.0.0/0 |

### EC2-DB
| Puerto | Protocolo | Origen |
|--------|-----------|--------|
| 1433 | TCP | IP privada EC2-Backend |
| 22 | TCP | 0.0.0.0/0 |

---

## 🚀 Pasos de Despliegue

### 1. Crear las instancias EC2 en AWS

Crear 3 instancias EC2 (Amazon Linux 2 / Ubuntu, t2.micro):
- `EC2-Frontend` → asignar IP Elástica pública
- `EC2-Backend` → solo IP privada
- `EC2-DB` → solo IP privada

Configurar los Security Groups según la tabla anterior.

---

### 2. Instalar Docker en cada instancia

```bash
sudo apt update && sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

---

### 3. Desplegar SQL Server (EC2-DB)

```bash
docker run -e "ACCEPT_EULA=Y" \
  -e "SA_PASSWORD=BerryControl*" \
  -p 1433:1433 \
  --name sqlserver \
  --restart always \
  -d mcr.microsoft.com/mssql/server:2022-latest

# Verificar
docker ps

# Crear base de datos y tablas
docker exec -it sqlserver /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U SA -P "Password123*" -No \
  -Q "CREATE DATABASE MiApp;"
```

---

### 4. Desplegar Backend Spring Boot (EC2-Backend)

```bash
docker pull ninahuancaromani/spring-backend:latest

docker run -d \
  -p 8080:8080 \
  -e DB_HOST=<IP_PRIVADA_EC2_DB> \
  -e DB_PASS=Password123* \
  --name backend \
  --restart always \
  ninahuancaromani/spring-backend:latest

# Verificar
docker ps
docker logs backend
```

---

### 5. Desplegar Frontend Angular (EC2-Frontend)

```bash
docker pull ninahuancaromani/angular-frontend:latest

docker run -d \
  -p 80:80 \
  --name frontend \
  --restart always \
  ninahuancaromani/angular-frontend:latest

# Verificar
docker ps
docker logs frontend
```

---

## 🐳 Comandos Docker Utilizados

```bash
# Construir imagen
docker build -t ninahuancaromani/angular-frontend:latest .
docker build -t ninahuancaromani/spring-backend:latest .

# Subir imagen a Docker Hub
docker push ninahuancaromani/angular-frontend:latest
docker push ninahuancaromani/spring-backend:latest

# Descargar imagen
docker pull ninahuancaromani/angular-frontend:latest
docker pull ninahuancaromani/spring-backend:latest

# Ver contenedores activos
docker ps

# Ver logs de un contenedor
docker logs frontend
docker logs backend
docker logs sqlserver

# Detener y eliminar contenedor
docker stop frontend && docker rm frontend

# Ejecutar comando dentro de un contenedor
docker exec -it sqlserver bash
```

---

## 🔌 Variables de Entorno Utilizadas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `DB_HOST` | IP privada de EC2-DB | 172.31.x.x |
| `DB_PASS` | Contraseña SQL Server SA | Password123* |
| `ACCEPT_EULA` | Aceptar licencia SQL Server | Y |
| `SA_PASSWORD` | Contraseña administrador SQL | Password123* |

---

## ✅ Evidencia de Funcionamiento

Las capturas de pantalla se encuentran en la carpeta `/evidencias/` del repositorio:

- `01-ec2-instancias.png` — EC2 desplegadas en AWS
- `02-security-groups.png` — Configuración de Security Groups
- `03-docker-ps-frontend.png` — `docker ps` en EC2-Frontend
- `04-docker-ps-backend.png` — `docker ps` en EC2-Backend
- `05-docker-ps-db.png` — `docker ps` en EC2-DB
- `06-sqlserver-corriendo.png` — SQL Server funcionando
- `07-app-navegador.png` — Aplicación Angular desde el navegador
- `08-prueba-conectividad.png` — Prueba de conectividad entre capas

---

## 🐛 Identificación y Solución de Errores

| Error detectado | Posible causa | Solución aplicada |
|----------------|--------------|-------------------|
| Backend no conecta con SQL Server | Puerto 1433 bloqueado en Security Group | Se habilitó acceso privado desde IP del Backend |
| Frontend no llega al Backend | IP privada incorrecta en variable de entorno | Se actualizó `DB_HOST` con la IP privada correcta |
| `docker ps` no muestra contenedor | Dockerfile mal construido | Se revisó con `docker logs` y se corrigió el CMD |
| SQL Server no inicia | Contraseña no cumple requisitos | Se cambió a contraseña con mayúscula, número y símbolo |
| Angular no carga en navegador | Nginx no encuentra `index.html` | Se verificó la ruta del build y se reconstruyó la imagen |

---

## 🔧 Herramientas de Diagnóstico Utilizadas

```bash
docker ps                    # Ver contenedores activos
docker logs <contenedor>     # Ver logs de errores
ping <IP_PRIVADA>            # Verificar conectividad de red
curl http://<IP>:8080/api/   # Probar endpoint del backend
telnet <IP> 1433             # Verificar puerto SQL Server
systemctl status docker      # Estado del servicio Docker
```

---

## 📁 Estructura del Repositorio

```
├── frontend/
│   ├── src/
│   ├── Dockerfile
│   └── nginx.conf
├── backend/
│   ├── src/
│   ├── target/
│   │   └── mi-app-0.0.1-SNAPSHOT.jar
│   └── Dockerfile
├── sql/
│   └── init.sql
├── evidencias/
│   ├── 01-ec2-instancias.png
│   ├── 02-security-groups.png
│   └── ...
└── README.md
```