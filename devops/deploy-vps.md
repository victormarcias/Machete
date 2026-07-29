# Deploy a un VPS (conceptos genéricos)

Cómo llevar una app (FastAPI, pero aplica a cualquier backend) de "corre en mi laptop" a "corre en un servidor real, con dominio y HTTPS". Los pasos son genéricos — sirven igual en Akamai/Linode, DigitalOcean, Hetzner, AWS EC2, etc. Es config de Linux, no del proveedor.

## 1. SSH con keys en vez de password

Un VPS nuevo por default deja loguear por SSH con usuario+contraseña — eso es fuerza-bruteable desde cualquier lado del mundo. La alternativa es un par de claves asimétricas: la privada se queda en tu máquina, la pública viaja al servidor. Nadie puede entrar sin la privada, y esa nunca sale de tu disco.

```bash
# en tu máquina local
ssh-keygen -t ed25519 -C "tu_email@ejemplo.com"
ssh-copy-id usuario@ip-del-servidor

# en el servidor, /etc/ssh/sshd_config
PasswordAuthentication no
PermitRootLogin no

sudo systemctl restart sshd
```

## 2. Firewall (ufw)

El servidor tiene todos los puertos potencialmente alcanzables desde internet. Un firewall es una whitelist explícita: solo dejás pasar lo que necesitás (SSH, HTTP, HTTPS) y todo lo demás se descarta antes de llegar a ningún proceso.

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

## 3. Protección contra fuerza bruta (fail2ban)

Aunque el login por password esté desactivado, bots siguen escaneando IPs y probando logins contra el puerto 22. `fail2ban` lee los logs de auth, detecta intentos fallidos repetidos desde la misma IP, y la banea a nivel firewall por un tiempo. Es una segunda capa, no reemplaza al SSH key auth del punto 1.

```bash
sudo apt install fail2ban
sudo systemctl enable --now fail2ban
```

## 4. Nginx como reverse proxy

Tu app (`uvicorn`/`fastapi run`) escucha en un puerto interno, **no expuesto directamente a internet** — por ejemplo `127.0.0.1:8000`. Nginx escucha en los puertos públicos (80/443) y reenvía el tráfico hacia tu app. La ventaja de no exponer uvicorn directo: Nginx maneja TLS, compresión, archivos estáticos y rate limiting sin que tu código tenga que saber nada de eso, y podés correr varias apps en el mismo servidor por dominio.

```nginx
# /etc/nginx/sites-available/miapp
server {
    listen 80;
    server_name miapp.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/miapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 5. HTTPS con Let's Encrypt

Un certificado HTTPS tiene que estar firmado por una autoridad confiable, o el navegador te tira warning. Let's Encrypt los emite gratis y `certbot` automatiza tanto la emisión como la renovación (expiran cada ~90 días). Certbot reescribe la config de Nginx para que termine el TLS ahí: descifra el HTTPS público y le pasa HTTP plano a tu app por dentro.

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d miapp.com
```

## 6. systemd para correr la app como servicio

Si corrés `fastapi run` a mano en una sesión SSH, el proceso muere en cuanto cerrás esa sesión. Un `.service` de systemd lo corre como daemon: arranca solo al bootear el servidor, y si crashea lo reinicia automáticamente — es el mismo mecanismo que usa el propio sistema operativo para sus servicios internos.

```ini
# /etc/systemd/system/miapp.service
[Unit]
Description=FastAPI app
After=network.target

[Service]
User=deploy
WorkingDirectory=/home/deploy/miapp
ExecStart=/home/deploy/miapp/.venv/bin/fastapi run main.py --port 8000
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now miapp
sudo systemctl status miapp
```

---
Relacionado: [Sync vs Async en FastAPI](../python/sync-vs-async-fastapi.md), [Escalabilidad vertical vs horizontal](README.md).
