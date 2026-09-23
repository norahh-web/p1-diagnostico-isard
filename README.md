# P1 - publica y diagnostica una web en isard

## datos de la vm

- usuario: ikasle
- hostname: daw-lubuntu-base
- ip de la vm: 192.168.123.249 (interfaz enp1s0)
- puerto: 8000

## esquema

```
pc del aula -> red de isard -> vm (ip real) -> puerto 8000 -> proceso python3 -> archivos de ~/catalogo-web
```

## 127.0.0.1, 0.0.0.0 e ip real

- 127.0.0.1: interfaz local (loopback). solo se puede llegar desde la propia vm.
- ip real de la vm: la de la interfaz conectada a la red de isard. es la que usan otras máquinas para llegar a la vm.
- 0.0.0.0: no es una dirección a la que se entre, es una forma de decir que el servidor escucha en todas las interfaces (127.0.0.1 y la ip real).

## resultados con el servidor en 127.0.0.1

| dato | resultado |
| --- | --- |
| código http index.html | 200 OK |
| código http productos.html | 200 OK |
| dirección de escucha | 127.0.0.1 |
| puerto | 8000 |
| proceso | python3 |

## dirección de la vm e interfaces

| pregunta | respuesta |
| --- | --- |
| ip real de la interfaz activa | 192.168.123.249 (interfaz enp1s0) |
| ¿funciona curl con esa ip si el servidor escucha en 127.0.0.1? | no, curl: (7) Failed to connect to 192.168.123.249 port 8000 after 0 ms: Couldn't connect to server |
| diferencia entre 127.0.0.1 y la ip de la interfaz | 127.0.0.1 es solo local a la vm, la ip de la interfaz es la que se ve desde la red |

## servidor en 0.0.0.0

| comprobación | resultado |
| --- | --- |
| dirección que muestra ss | 0.0.0.0:8000 (python3, pid=8341) |
| acceso con 127.0.0.1 desde la vm | HTTP/1.0 200 OK |
| acceso con la ip real desde la vm | HTTP/1.0 200 OK (curl -I http://192.168.123.249:8000/) |
| acceso desde el pc del aula | no lo he podido comprobar (desde la vm la ip real responde 200 OK) |

## resolución de nombres

- no he configurado ningún servidor dns ni he llegado a añadir catalogo.local al archivo /etc/hosts. al probar curl -I http://catalogo.local:8000/css/estilos.css me salió curl: (6) Could not resolve host: catalogo.local, porque sin una entrada en /etc/hosts (o un dns) el nombre no se asocia a ninguna ip.

## tabla de diagnóstico

| caso | síntoma y evidencia | componente sospechoso | causa comprobada y solución |
| --- | --- | --- | --- |
| A puerto 8080 | curl: (7) Failed to connect to 127.0.0.1 port 8080 after 0 ms: Couldn't connect to server | puerto / proceso | no hay ningún proceso escuchando en el 8080, el servidor escucha en el 8000. se usa el puerto correcto (8000) |
| B producto.html | curl -I http://127.0.0.1:8000/producto.html devuelve HTTP/1.0 404 File not found | recurso web (archivo) | el servidor sí responde pero el archivo producto.html no existe, el correcto es productos.html |
| C servidor detenido | curl: (7) Failed to connect to 127.0.0.1 port 8000 after 0 ms: Couldn't connect to server | proceso | con ctrl+c el proceso python3 se ha parado y ya no hay nadie escuchando en el 8000. se vuelve a arrancar el servidor |
| D ip real con escucha local | curl: (7) Failed to connect to 192.168.123.249 port 8000 after 0 ms: Couldn't connect to server (con ss mostrando 127.0.0.1:8000) | interfaz / dirección de escucha | el servidor solo escucha en 127.0.0.1 y no en la interfaz de la ip real. se arranca con --bind 0.0.0.0 |
| E acceso desde el pc | no comprobado desde el pc del aula. desde la vm, con el servidor en 0.0.0.0, curl a la ip real da 200 OK | red entre el pc y la vm (sin comprobar) | sin comprobar |
| F hoja de estilos | index.html llamaba a css/styles.css (se ve en el git diff) pero el archivo que existe es css/estilos.css. los curl del css con catalogo.local dieron curl: (6) Could not resolve host porque todavía no había configurado el nombre | recurso web (ruta en index.html) | la ruta del href estaba mal, el archivo se llama estilos.css. corregido en la rama fix/css cambiando css/styles.css por css/estilos.css |

## conexión rechazada vs http 404

- conexión rechazada (curl: (7)): ni siquiera se llega a establecer la conexión, porque no hay ningún proceso escuchando en esa ip y puerto. no hay respuesta http.
- 404: la conexión se ha establecido y el servidor http ha respondido, pero dice que ese recurso no existe.

## acceso desde el pc del aula

no he podido probar el acceso desde el pc del aula. lo único comprobado es que desde la propia vm, con el servidor en 0.0.0.0, la ip real (192.168.123.249) responde 200 OK.

## git log

```
* 28118a3 (HEAD -> main, fix/css) fix: corrige la ruta de la hoja de estilos
* 2201172 feat: crea catalogo web inicial
```

## conclusión individual

**¿qué comprobaciones haría y en qué orden si una web no se abre?**

primero miraría si hay un proceso escuchando y en qué dirección y puerto (ss -ltnp | grep 8000), porque sin proceso no va a funcionar nada. después probaría con curl -I desde la propia vm, primero con 127.0.0.1 y luego con la ip real: si con la ip real falla, el servidor solo escucha en local y hay que arrancarlo con 0.0.0.0. si desde la vm va bien, probaría desde otra máquina para ver si el problema es la red. si uso un nombre, comprobaría también que se resuelve. al final miraría el código http, porque un 404 ya no es un problema de conexión sino del archivo o la ruta.

**¿por qué un 404 demuestra que parte de la infraestructura sí funciona?**

porque para recibir un 404 la petición ha tenido que llegar hasta el servidor: se ha resuelto la ip, la red ha llevado la petición hasta la vm, el puerto estaba abierto y el proceso python3 estaba escuchando y ha contestado. lo único que falla es que el recurso pedido no existe (por ejemplo producto.html), así que el problema es del archivo y no de la conexión.

