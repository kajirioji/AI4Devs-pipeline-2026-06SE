# Prompts — Pipeline de GitHub Actions

Documentación de los prompts usados para generar el pipeline de CI/CD del ejercicio
"Creando un Pipeline en GitHub Actions" (Módulo 13). Trabajé con Claude como asistente,
en una conversación iterativa en vez de prompts sueltos, así que reconstruyo aquí el
prompt equivalente a cada paso pedido en el enunciado.

## Contexto de partida

Antes de pedir el pipeline, ya tenía:

- Una cuenta de AWS (plan Free) y un EC2 (`t3.micro`, Amazon Linux 2023) con Node 18 y
PM2 instalados a mano.
- Los tres secrets creados en el repo: `EC2_HOST`, `EC2_USERNAME`, `EC2_SSH_KEY`.
- Confirmado que el backend escucha en el puerto 3010 (revisando `backend/src/index.ts`)
y que Prisma no se conecta a la base de datos al arrancar, solo en la primera consulta.



## Prompt 1 — Tests de backend

> Necesito un pipeline de GitHub Actions para este proyecto (backend en TypeScript con
> Express y Prisma, tests con Jest). Quiero que se dispare con un push a una rama que
> tenga un Pull Request abierto contra `main`. El primer job debe:
>
> - hacer checkout del código,
> - instalar Node.js 18,
> - instalar las dependencias del backend con `npm ci` (usando el `package-lock.json`
> para que sea reproducible),
> - ejecutar los tests del backend con `npm test`.
>
> Antes de nada, dime cómo debería estructurarse el `on:` para que dispare exactamente
> con "push a una rama con PR abierto", porque no tengo claro si es `push` o
> `pull_request`.

**Resultado:** el trigger correcto es `pull_request` con `types: [opened, synchronize]` —
`opened` cubre el momento de abrir el PR y `synchronize` cada push posterior a esa rama.
`push` a secas no sirve porque dispararía con cualquier push, tenga o no PR abierto.

## Prompt 2 — Build del backend

> Añade al mismo job un paso que compile el backend (es TypeScript, el script es
> `npm run build`). También necesito generar el cliente de Prisma antes de compilar,
> porque el código importa tipos que Prisma genera (`npx prisma generate`).
>
> Como el siguiente job va a desplegar en un EC2, necesito que el resultado del build
> —la carpeta `dist/`, el `package.json`, el `package-lock.json` y la carpeta
> `prisma/`— quede disponible para ese segundo job. Explícame por qué no puedo simplemente
> usar los archivos del primer job en el segundo, y cuál es la forma correcta de
> pasarlos.

**Resultado:** cada job de GitHub Actions corre en una máquina nueva y vacía; nada se
comparte entre jobs a menos que se suba explícitamente como "artifact" con
`actions/upload-artifact` en un job y se baje con `actions/download-artifact` en el
siguiente. Añadimos un paso que arma una carpeta `release/` solo con lo necesario para
producción (sin código fuente `.ts` ni `node_modules`) y la sube como artifact.

## Prompt 3 — Despliegue en EC2

> Quiero un segundo job, `deploy`, que dependa de que el primero termine bien
> (`needs`) y que solo corra si el evento es un `pull_request`. Este job debe:
>
> - descargar el artifact del job anterior,
> - conectarse por SSH a mi EC2 usando los secrets `EC2_HOST`, `EC2_USERNAME` y
> `EC2_SSH_KEY`,
> - copiar los archivos al servidor,
> - instalar las dependencias ahí, generar el cliente de Prisma otra vez (en el
> servidor, no en el runner) y arrancar la app con PM2.
>
> No quiero pasar por S3 como en otros ejemplos que he visto — es un servidor propio,
> quiero copiar los archivos directamente por SSH con `scp`. Tampoco quiero usar
> `StrictHostKeyChecking=no`; prefiero verificar el host con `ssh-keyscan` como se
> hace correctamente.
>
> Una cosa más: el EC2 no tiene base de datos. ¿Es un problema para desplegar el
> backend, dado que usa Prisma?

**Resultado:** no es un problema para este ejercicio — `new PrismaClient()` no se
conecta a la base de datos al arrancar, solo al ejecutar la primera consulta real, así
que el servidor arranca y responde sin base de datos; solo fallarían las rutas que
consultan datos. Para el `pm2 start`/`pm2 restart`, usamos el patrón
`pm2 restart backend-app || pm2 start dist/index.js --name backend-app`: la primera
vez `restart` falla porque el proceso no existe todavía, y cae al `start`; a partir de
ahí, cada deploy hace `restart` con el código nuevo.

## Corrección posterior

Al revisar el código del backend encontramos que escucha en el **puerto 3010**, no en
el 8080 que había configurado inicialmente en el Security Group del EC2 (copiado sin
pensar de un ejemplo con otro puerto). Corregido en el Security Group antes de la
primera ejecución del pipeline.