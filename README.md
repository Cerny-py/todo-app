# # 🚀 To-Do Cloud: Despliegue Continuo (PHP & MariaDB)

Integrantes: 
Franco Ortiz, Bastian Meneses, Bastian Román, Angel Meneses y Joaquín Espinoza 

Este repositorio contiene el código fuente, la infraestructura contenerizada y el pipeline de Integración y Despliegue Continuo (CI/CD) para un gestor de tareas (To-Do App) desarrollado en PHP y MariaDB. 

El objetivo de este proyecto es garantizar que **cada cambio aprobado llegue automáticamente a la nube** en Microsoft Azure, sin intervención manual en el servidor, manteniendo la persistencia de los datos en todo momento.

---

## 🏗️ 1. Arquitectura del Sistema

La infraestructura se basa en una arquitectura de microservicios orquestada mediante Docker Compose osea nuestro archivo (`docker-compose.yml`):

*   **Frontend/Backend (PHP):** La aplicación principal se construye a partir de una imagen personalizada definida en `app/Dockerfile`. Todo el sistema de base de datos y UI se procesa aquí mediante `index.php`.
*   **Base de Datos (MariaDB):** Contenedor independiente. La estructura inicial de la base de datos se levanta automáticamente desde `database/init.sql`. Las actualizaciones posteriores del esquema (migraciones) se manejan de forma segura a través de `database/migrate.sql` para no perder las tareas registradas.
*   **Persistencia (Volúmenes):** Se utiliza un volumen gestionado por Docker para almacenar los datos de MariaDB. Esto cumple con la regla de oro del proyecto: **los datos sobreviven a todos los despliegues**.

---

## 📋 2. Requisitos Previos para el Servidor

Para que este proyecto pueda ser desplegado desde cero por otro equipo, se debe preparar la siguiente infraestructura en **Microsoft Azure**:

1.  **Máquina Virtual (VM):** Una VM con distribución Linux (ej. Ubuntu Server).
2.  **Red y Seguridad:** Habilitar Reglas de Grupo de Seguridad (NSG) para abrir los puertos:
    *   `Puerto 80` (HTTP) - Para acceder a la aplicación web.
    *   `Puerto 22` (SSH) - Para que GitHub Actions pueda inyectar el código.
3.  **Software Interno:** La VM debe tener instalados **Git**, **Docker** y **Docker Compose**.
4.  **Protección de Ramas:** En GitHub, la rama `main` debe estar protegida exigiendo revisión de Pull Requests (ningún commit directo a `main`).

---

## 🔐 3. Variables de Entorno y Secrets

### Variables de la Aplicación
La aplicación web exige estrictamente la presencia de variables de entorno para conectarse a la base de datos. Estas se configuran dentro del servidor/contenedor:
*   `DB_HOST`: Nombre del servicio de base de datos en Docker.
*   `DB_PORT`: Puerto de conexión (por defecto `3306`).
*   `DB_NAME`: Nombre de la base de datos.
*   `DB_USER`: Usuario administrador de la BD.
*   `DB_PASSWORD`: Contraseña segura para la BD.
*   `APP_VERSION`: Versión o Hash del commit (visible en el footer de la app).

### GitHub Secrets (Autenticación CI/CD)
Para que el pipeline de GitHub Actions se conecte de forma segura a la VM en Azure, configura los siguientes secretos en el repositorio (`Settings > Secrets and variables > Actions`):
*   `SSH_HOST`: Dirección IP pública de la VM en Azure.
*   `SSH_USER`: Nombre de usuario de la sesión Linux (ej. `azureuser`).
*   `SSH_PRIVATE_KEY`: Llave privada (formato `.pem`) asociada a la VM.

---

## 🔄 4. Flujo de Trabajo (CI/CD)

El pipeline de automatización está centralizado en el archivo `.github/workflows/deploy.yml`. El ciclo de vida de un despliegue sigue este estándar estricto:

1.  **Desarrollo Aislado:** El desarrollador crea una rama (ej. `PR-APP`, `PR-BD`) y sube sus cambios.
2.  **Pull Request (PR):** Se abre un PR hacia `main`. **Nadie aprueba su propio PR.** El líder del equipo realiza el Code Review.
3.  **Trigger:** Al aprobarse y hacer *merge*, se dispara GitHub Actions.
4.  **Conexión y Actualización:** El *Runner* de GitHub se conecta por SSH a la VM, ejecuta un `git pull` de los últimos cambios y redespliega los contenedores usando `docker compose up -d --build`.
5.  **Healthchecks (Pruebas de Salud):** El sistema levanta la base de datos y la aplicación espera activamente (`depends_on: condition: service_healthy`) a que la BD esté lista. El pipeline verifica que la web responda con código HTTP 200; **si la web no responde, el workflow falla.**

---

## 🚀 5. Instrucciones de Despliegue desde Cero

Si otro equipo desea replicar este proyecto, debe seguir estos pasos exactos para que funcione:

1.  **Clonar el repositorio:** `git clone https://github.com/[usuario]/todo-app.git`
2.  **Preparar Azure:** Crear la Máquina Virtual en Azure, abrir puertos (80/22) e instalar Docker.
3.  **Configurar Secrets:** Extraer la IP, Usuario y Llave Privada de Azure y guardarlos en los *Secrets* de GitHub.
4.  **Primer Despliegue:** Hacer un pequeño cambio en el código (ej. un comentario), subirlo a una rama secundaria, hacer un Pull Request, aprobarlo e integrarlo a `main`. GitHub Actions hará el primer despliegue automáticamente.
5.  **Verificación:** Acceder a la IP Pública por navegador y comprobar que el sistema permite crear tareas.
