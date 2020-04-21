# Docker Multi Stage para crear imágenes limpias

Docker Multi-Stage fue introducido en la versión 17.05 y te permite crear múltiples imágenes de Docker en el mismo Dockerfile, podrás utilizar múltiples sentencias FROM en el mismo Dockerfile.

Vamos a crear un pequeño proyecto con NestJS y a partir el proyecto crearemos el Dockerfile con las dependencias dev y sin ellas, para notar las diferencias entre ambas construcciones de imágenes.

### Creando el proyecto NestJS y construcción de una imagen Docker de forma básica

NestJS es un framework para crear aplicaciones del lado del servidor con Node.js con un conjunto de tecnologías como Express y TypeScript. Al iniciar debemos instalar de forma global el cli de nest y crear el proyecto

```
npm i -g @nestjs/cli
nest new docker-multi-state
```

Ya tenemos el proyecto iniciado, vamos al directorio del proyecto y creamos el Dockerfile de forma básica, dejando las dependencias dev

```
FROM node:12.16-alpine
 
WORKDIR /app/
COPY . /app/
RUN npm install &amp;&amp; npm run build
 
ENTRYPOINT [ "npm" ]
CMD [ "run", "start:prod" ]
EXPOSE 3000
```

Procedemos a hacer la construcción de la imagen Docker y verificamos su peso

```
docker build -t docker-without-multi-stage .
docker images
```

Ya podemos ver el peso de la imagen, ahora vamos a cambiar el Dockerfile para aplicar multi-stage.

### Haciendo el Dockerfile con Multi-Stage

Modificando el archivo Dockerfile utilizamos Multi-Stage para eliminar de la imagen final las dependencias dev

```
FROM node:12.16-alpine as builder
LABEL stage=builder
WORKDIR /app/
COPY . /app/
RUN npm install &amp;&amp; npm run build

FROM node:12.16-alpine
WORKDIR /app/
COPY --from=builder /app/package.json /app/package-lock.json ./
COPY --from=builder /app/dist ./dist/
RUN npm ci --only=production

ENTRYPOINT [ "npm" ]
CMD [ "run", "start:prod" ]
EXPOSE 3000
```

Implementando el Dockerfile anterior podemos ver que creamos la primera etapa con el nombre builder y le colocamos la etiqueta stage con el valor builder, eso nos servirá luego para poder filtrar y eliminar las imágenes intermedias que quedan luego de la creación de la imagen final.

En la siguiente etapa, copiamos los archivos necesarios y la versión transpilada para producción, notamos que la sentencia COPY tiene el párametro –from con la variable builder, eso es para indicar que de la etapa builder necesitamos los archivos o directorios.

Volvemos a hacer el build y verificamos el peso de la imagen Docker

```
docker build -t docker-with-multi-stage .
docker images
```

Se puede verificar la diferencia de más de 150MB entre las imágenes finales en su peso y se debe a solo omitir las dependencias dev a la hora de construir la imagen.

También vemos la imagen de la etapa builder, necesitamos eliminar esa imagen, ocupa espacio en nuestro local.

```
docker image prune -f --filter label=stage=builder
```

Ya tenemos las imágenes intermedias eliminadas y una imagen para compartir limpia en el container register y colocar en producción.

En mi página web se encuentra la misma explicación, con unas imagenes de nuestra [https://ajdelgados.com/2020/04/21/docker-multi-stage-para-crear-imagen-limpia/](https://ajdelgados.com/2020/04/21/docker-multi-stage-para-crear-imagen-limpia/)