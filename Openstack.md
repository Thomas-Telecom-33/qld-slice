# BI BACKEND

---

## 1. Installations nécessaires

### Config

```bash
docker --version
sudo apt install -y docker.io docker-compose
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world

curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
>   -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose
```
Dans Vscode (si utilisation), ouvrir avec le container : open containers ....
Laissez le tout charger


Une fois dans le container dockker :
alembic upgrade head
python -m uvicorn slices_bi_blueprint_backend.app:app --reload --host 0.0.0.0 --port 8000

On essaie ensuite d'accéder aux docs suivantes :
>http://localhost:8000/docs
>http://localhost:8000/redoc

On voit alors dans notre terminal du container : 
INFO:     127.0.0.1:34672 - "GET /redoc HTTP/1.1" 200 OK
INFO:     127.0.0.1:34672 - "GET /apis/bi.slices.eu/v1alpha2/openapi.json HTTP/1.1" 200 OK
INFO:     127.0.0.1:53914 - "GET /docs HTTP/1.1" 200 OK
INFO:     127.0.0.1:53914 - "GET /apis/bi.slices.eu/v1alpha2/openapi.js


