---
title: 'Configuración de kubernetes con un registry privado'
date: 2020-03-09T13:41:16-04:00
hero: "/images/kubernetes.jpg"
excerpt: Pasos para configurar un registry privado con un clúster kubernetes.
authors:
  - Ever Favio Argollo Ticona
categories: [Docker]
---

### Prerequisitos

Se necesita un clúster Kubernetes, y la línea de comandos kubectl habilitada y configurada.

### Identificación con docker

El primer paso es autenticarse con en el registro privado
```shell
  $ docker login  https://registry.privado.com
```
Si el registry no cuenta con ssl debe registrarlo en el archivo daemon.json y reiniciar el servicio de docker
```shell
  $ sudo vim /etc/docker/daemon.json
  ---
  {
    "insecure-registries":["http://registry.privado.com"]
  }

```

Cuando ejecute el comando anterior, debe ingresar su usuario y contraseña.

El proceso de login crea(o actualiza) el archivo config.json donde se generará un token de autorización

Verifique el archivo config.json, deberia generarse algo similar a esto:
```json
  {
    "auths": {
		"registry.privado.com": {
			"auth": "dWludGVyb3B..........GFkOkFCQ2FiYzEyM3Vp"
		}
	}
```

### creando un secreto basado en una credencial existente de docker

Un cluster de kubernetes usa el secret de docker-registry para autenticarse con un registry privado para recuperar imágenes docker

Si el paso anterior se realizó satisfactoriamente, podemos copiar esa credencial generada dentro de kubernetes:
```shell
  $ kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=<~/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson
```

Para verificar que el token se haya registrado correctamente podemos verlo con:

```shell
  $ kubectl get secret regcred --output=yaml
```

### Verificación y uso del registry privado

Crearemos un pod de prueba para verificar que nuestro token funcione correctamente.

```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: test-private-registry
  spec:
    containers:
    - name: private-reg-container
      image: registry.privado.com/imagenes/mi-imagen:v1
    imagePullSecrets:
    - name: regcred

```