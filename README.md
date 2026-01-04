Fork de https://github.com/0xConstant1/stremio-addon-manager
Avec construction auto du Docker sur GitHub pour pouvoir le déployer simplement

```bash
services:
  stremio-addon-manager-new:
    image: ghcr.io/aerya/0xconstant1-stremio-addon-manager:latest
    container_name: stremio-addon-manager-new
    restart: always
    ports:
      - "8061:80"
```