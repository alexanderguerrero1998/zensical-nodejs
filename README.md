Para subir a github desde cero
```sh
git init
git add .
git commit -m "primer commit"
git branch -M main
git remote add origin https://github.com/TU_USUARIO/TU_REPO.git
git push -u origin main
```

Para cuando se hacen cambios
```sh
git add .
git commit -m "descripcion del cambio"
git pull --rebase origin main
git push
```
