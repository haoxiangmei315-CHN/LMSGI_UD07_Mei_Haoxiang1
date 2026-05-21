# Manual de Explotación del Sistema ERP/CRM
### WillmanTech S.L.



## 1. Introducción y Arquitectura

WillmanTech S.L. utiliza un sistema ERP basado en Odoo 16 Community desplegado mediante contenedores Docker. Esto facilita bastante el mantenimiento porque no hay que instalar nada directamente en el servidor, todo está aislado en sus propios contenedores.

El sistema tiene activados los siguientes módulos principales:

- Ventas: gestión de presupuestos, pedidos y clientes.
- Facturación / Contabilidad: emisión de facturas, gestión de pagos y cobros.
- Inventario: control del stock de productos.
- CRM: seguimiento de oportunidades y clientes potenciales.

El archivo `docker-compose.yml` define los dos servicios principales: el servidor web de Odoo y la base de datos PostgreSQL. Ambos se comunican internamente a través de la red que crea el propio Docker Compose.



## 2. Guía de Instalación y Reinstalación

Para levantar el entorno desde cero hay que tener instalado Docker y Docker Compose en la máquina. Los pasos son los siguientes:

**Paso 1 – Clonar el repositorio del proyecto:**

```bash
git clone https://github.com/willmantech/erp-deploy.git
cd erp-deploy
```

**Paso 2 – Crear el archivo de variables de entorno `.env`:**

Dentro de la carpeta del proyecto, crear un archivo `.env` con el siguiente contenido:

```
POSTGRES_DB=willmantech_db
POSTGRES_USER=odoo
POSTGRES_PASSWORD=odoo1234
ODOO_ADMIN_PASSWD=admin_seguro_2025
```

Estas variables las lee el `docker-compose.yml` automáticamente al arrancar.

**Paso 3 – Levantar los contenedores:**

```bash
docker-compose up -d
```

Con el flag `-d` arranca en segundo plano. Para ver que todo ha ido bien:

```bash
docker-compose ps
```

Deben aparecer los dos contenedores con estado `Up`.

**Paso 4 – Acceder a la interfaz web:**

Abrir el navegador y entrar en `http://localhost:8069`. La primera vez pedirá crear una base de datos con el nombre, idioma y contraseña de administrador.

**En caso de reinstalación**, primero hay que eliminar los volúmenes para partir de cero:

```bash
docker-compose down -v
docker-compose up -d
```

> Atención: el flag `-v` borra todos los datos almacenados. Hacer siempre una copia de seguridad antes (ver sección 4).

---

## 3. Seguridad y Control de Acceso

Odoo gestiona los permisos mediante grupos de usuarios. Para WillmanTech se han configurado tres perfiles principales:

| Rol--> Acceso: Descripción 
| Administrador: Puede configurar el sistema, crear usuarios y ver todos los datos. 
| Contable--> Facturación y contabilidad: Puede emitir facturas, registrar pagos y generar informes contables. 
| Comercial-->Ventas y CRM: Puede gestionar clientes, crear presupuestos y hacer seguimiento de oportunidades. 

Para crear o editar usuarios hay que ir a **Ajustes → Usuarios y Empresas → Usuarios**.

Política de contraseñas recomendada:

- Mínimo 8 caracteres.
- Combinar letras mayúsculas, minúsculas y números.
- Cambiar la contraseña cada 90 días.
- No reutilizar las últimas 3 contraseñas.

Odoo también tiene la opción de activar la autenticación en dos pasos (2FA) desde los ajustes de cada usuario. Es recomendable activarla al menos para el administrador.

Además, cada usuario solo ve los datos de los clientes y ventas que tiene asignados, a no ser que tenga permisos de gestor o administrador.


## 4. Procedimiento de Backup y Restauración

Es muy importante hacer copias de seguridad frecuentes, sobre todo de la base de datos PostgreSQL, que es donde se guarda toda la información del ERP.

**Hacer un backup de la base de datos:**

bash
docker exec -t nombre_contenedor_postgres pg_dump -U odoo willmantech_db > backup_$(date +%F).sql


Esto genera un archivo .sql con la fecha del día, por ejemplo `backup_2025-05-10.sql`.

También se puede hacer desde la propia interfaz web de Odoo, accediendo a `http://localhost:8069/web/database/manager`, aunque este método genera un archivo .zip que incluye tanto la base de datos como los archivos adjuntos almacenados.

Restaurar la base de datos:
bash
docker exec -i nombre_contenedor_postgres psql -U odoo willmantech_db < backup_2025-05-10.sql

Se recomienda guardar los backups en una carpeta externa al servidor o en un servicio en la nube, y automatizar el proceso con una tarea `cron` que lo haga cada noche.

Ejemplo de tarea cron para backup diario a las 2:00 AM:

0 2 * * * docker exec -t postgres_willmantech pg_dump -U odoo willmantech_db > /backups/backup_$(date +\%F).sql


## 5. Flujo Operativo de Facturación e Informes

Cómo generar una factura

1. Iniciar sesión en el ERP y acceder al módulo de Facturación.
2. Hacer clic en Clientes → Facturas y luego en el botón Nuevo.
3. Seleccionar el cliente en el campo correspondiente.
4. Añadir las líneas de factura: producto o servicio, cantidad y precio.
5. Revisar que los impuestos (IVA) se han aplicado correctamente.
6. Pulsar Confirmar para pasar la factura al estado "Publicada".

Una vez confirmada, la factura ya tiene número y no se puede modificar. Si hay un error, habría que crear una nota de crédito.

### Cómo exportar la factura a PDF

Una vez confirmada la factura, aparece el botón Imprimir en la parte superior. Al pulsarlo, Odoo lanza internamente el siguiente proceso:


Plantilla QWeb (XML)
        |
        v
Motor de plantillas Odoo → genera HTML
        |
        v
wkhtmltopdf (convierte el HTML a PDF)
        |
        v
Archivo PDF listo para descargar o enviar por email


Básicamente lo que hace Odoo es coger la plantilla que hemos creado en XML (con las directivas QWeb), rellenarla con los datos reales de esa factura y convertir el resultado HTML en un PDF usando la herramienta `wkhtmltopdf`, que es un programa que se instala junto con Odoo.

Si el PDF no se genera correctamente o aparece en blanco, normalmente es porque wkhtmltopdf no está instalado o hay un problema con la ruta en la configuración de Odoo.



Manual elaborado para el proyecto de implantación ERP/CRM de WillmanTech S.L. – DAM/DAW 2024/2025.
