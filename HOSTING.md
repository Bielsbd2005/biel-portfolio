# Guía: Hostear tu Portfolio en Raspberry Pi con Auto-Deploy

Esta guía explica cómo alojar tu sitio web estático (portfolio) en una Raspberry Pi y configurar despliegues automáticos (auto-deploy) para que cada vez que hagas `git push` en GitHub, tu web se actualice sola, sin necesidad de conectarte manualmente a la Raspberry.

## 1. Clonar el repositorio

Primero, alojaremos el código fuente de tu página directamente en el escritorio de la Raspberry.

1. Abre la terminal de tu Raspberry Pi.
2. Navega al escritorio:
   ```bash
   cd ~/Desktop
   ```
3. Clona tu repositorio (cambia la URL por la tuya):
   ```bash
   git clone https://github.com/tu-usuario/tu-repositorio.git portfolio
   ```
   > **Nota:** Ahora tu código vivirá en `~/Desktop/portfolio`.

## 2. Configurar el Servidor Web (Nginx)

Usaremos Nginx por ser ligero y perfecto para los recursos de una Raspberry Pi.

1. Instala Nginx:
   ```bash
   sudo apt update
   sudo apt install nginx
   ```
2. Crea el archivo de configuración para tu portfolio:
   ```bash
   sudo nano /etc/nginx/sites-available/portfolio
   ```
3. Pega el siguiente contenido (asegúrate de cambiar `tu-dominio.com` y `tu-usuario` ):
   ```nginx
   server {
       listen 80;
       server_name tu-dominio.com;

       root /home/tu-usuario/Desktop/portfolio;
       index index.html;

       location / {
           try_files $uri $uri/ =404;
       }
   }
   ```
4. Activa el sitio creando un enlace simbólico:
   ```bash
   sudo ln -s /etc/nginx/sites-available/portfolio /etc/nginx/sites-enabled/
   ```
5. Verifica la configuración de Nginx y reinícialo si no hay errores:
   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```
6. **[IMPORTANTE] Permisos:** Nginx necesita permisos de ejecución en tu directorio para poder acceder a los archivos. Ejecuta en terminal:
   ```bash
   chmod +x /home/tu-usuario
   chmod +x /home/tu-usuario/Desktop
   ```

## 3. Conectar Cloudflare Tunnel (Requisito previo)

Debes tener el túnel de Cloudflare ya configurado en tu Raspberry instalando cloudflared. En el panel online, asegúrate de apuntarlo al Nginx local:

1. Ve a **Cloudflare Zero Trust** -> **Networks** -> **Tunnels**.
2. Edita tu túnel y en la pestaña **Public Hostname**, configura la ruta:
   - **Service:** `http://localhost:80` (o la IP local de tu Raspberry).
   - **Domain:** El dominio que hayas redirigido.

## 4. Automatizar despliegue con GitHub (Self-Hosted Runner)

Para sincronizar los cambios de tu web con cada push a la rama `main` sin exponer tu IP pública ni abrir puertos de red, usaremos GitHub Actions con un *Self-Hosted Runner*.

### A. Instalar y configurar el Runner en la Raspberry
1. En tu repositorio de GitHub, navega a **Settings** > **Actions** > **Runners**.
2. Haz clic en **New self-hosted runner** y selecciona la arquitectura **Linux -> ARM** (o ARM64 si tu sistema operativo es de 64 bits).
3. Ejecuta los comandos paso a paso que te proporciona la misma pantalla de GitHub dentro de la terminal de tu Raspberry Pi para descargarlo y vincularlo.
4. Para que el Runner funcione constantemente en segundo plano aunque cierres sesión:
   ```bash
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```
   *(Opcional: puedes verificar que funciona con `sudo ./svc.sh status`)*.

### B. Evitar errores de permisos (Pro-Tip)
El runner se ejecuta en segundo plano. Para asegurarnos de que puede hacer la acción de `git pull` sin restricciones de persmisos en la carpeta de tu Desktop, debes hacerte dueño de la recursividad del directorio:
```bash
sudo chown -R $USER:$USER ~/Desktop/portfolio
```

### C. Crear el flujo base de Actions (Workflow)
Dentro del código de tu portfolio (puedes hacerlo vía la web de GitHub), crea el archivo en esta ruta: `.github/workflows/deploy.yml` e inserta el siguiente bloque:

```yaml
name: Deploy Portfolio
on:
  push:
    branches:
      - main  # Pon aquí la rama principal de tu código

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Pull latest changes
        run: |
          cd ~/Desktop/portfolio
          git pull origin main
```

---

## 🛠 Troubleshooting: "Solo veo la página por defecto de Nginx"

Tras completar el paso 2 o 3, si al entrar a tu web ves el mensaje *"Welcome to nginx!"* en lugar del HTML de tu portafolio, es debido a que la página por defecto de nginx sigue activa y solapando tu sitio o hay un conflicto en permisos.

Desde tu terminal sigue la siguiente corrección:

1. **Elimina el enlace al sitio de relleno por defecto:**
   ```bash
   sudo rm /etc/nginx/sites-enabled/default
   ```
2. **Fuerza la activación de tu web:**
   ```bash
   sudo ln -s /etc/nginx/sites-available/portfolio /etc/nginx/sites-enabled/
   ```
3. **Rectifica la ruta raíz en la configuración:** Ejecuta `sudo nano /etc/nginx/sites-available/portfolio` prestando suma atención a la línea de `root`. Confirma que es exacta: (Ej: `root /home/biel/Desktop/portfolio;`).
4. **Revisión profunda a los Permisos Críticos:** Nginx es un 'invitado' en el sistema pidiendo leer en un directorio privado. Si no lo consigue, verás un error 403 / 404 o la página default.
   ```bash
   sudo chmod o+x /home/biel
   sudo chmod o+x /home/biel/Desktop
   sudo chmod -R o+r /home/biel/Desktop/portfolio
   ```
5. **Aplica y comprueba la corrección:**
   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```
