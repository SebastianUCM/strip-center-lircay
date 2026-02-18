# Subir a GitHub y desplegar

## 1. Subir a GitHub

Ya está hecho el primer commit. Sigue estos pasos:

### Crear el repositorio en GitHub

1. Entra en **https://github.com/new**
2. Nombre del repo (ejemplo): `strip-center-lircay`
3. Elige **Público**
4. **No** marques "Add a README" (ya tienes código local)
5. Clic en **Create repository**

### Conectar y subir desde tu PC

En la terminal, dentro de la carpeta del proyecto (`strip-center-lircay`), ejecuta (sustituye `TU_USUARIO` por tu usuario de GitHub):

```bash
git remote add origin https://github.com/TU_USUARIO/strip-center-lircay.git
git branch -M main
git push -u origin main
```

**Importante:** siempre es `git branch` (con **git** delante), no solo `branch`.

Si GitHub te pide usuario/contraseña, usa un **Personal Access Token** como contraseña (Settings → Developer settings → Personal access tokens).

---

## 2. Deploy (elegir una opción)

### Opción A: Vercel (recomendado, gratis)

1. Entra en **https://vercel.com** e inicia sesión con GitHub.
2. **Add New** → **Project** → importa el repo `strip-center-lircay`.
3. Vercel detecta Vue/Vite; deja **Build Command**: `npm run build` y **Output Directory**: `dist`.
4. **Deploy**. Te dará una URL tipo `strip-center-lircay.vercel.app`.

Cada vez que hagas `git push` a `main`, Vercel volverá a desplegar.

### Opción B: Firebase Hosting (ya tienes config)

Si prefieres Firebase:

```bash
npm install -g firebase-tools
firebase login
firebase deploy
```

El proyecto ya tiene `firebase.json` y `.firebaseerc` configurados.

### Opción C: Netlify

1. **https://app.netlify.com** → **Add new site** → **Import an existing project** → GitHub → elige el repo.
2. Build command: `npm run build`, Publish directory: `dist`.
3. **Deploy**.

---

Resumen: crea el repo en GitHub, ejecuta los 3 comandos `git remote` / `git branch` / `git push`, y luego conecta el repo en Vercel (o Firebase/Netlify) para el deploy.
