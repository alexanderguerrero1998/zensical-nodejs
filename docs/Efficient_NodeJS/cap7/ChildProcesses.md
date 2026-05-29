# Procesos Hijo

Ejecutar una instancia de Node en un solo proceso funciona bien para una aplicación pequeña. A medida que la aplicación se vuelve más compleja y sirve a más usuarios, un solo proceso no será suficiente para manejar la carga de trabajo creciente. No importa qué tan potente sea tu servidor, un solo hilo puede soportar solo una carga limitada.

El hecho de que tu código Node se ejecute en un solo hilo no significa que no puedas aprovechar múltiples procesos y, por supuesto, también múltiples máquinas.

El uso de múltiples procesos es la mejor manera de escalar una aplicación Node. Node está diseñado para construir aplicaciones distribuidas con muchos nodos. ¡Por eso se llama Node! La escalabilidad está incorporada en la plataforma, y no es algo que empieces a considerar más adelante en la vida de una aplicación.

---

## Introducción a los Procesos Hijo

Podemos crear fácilmente un proceso hijo en un proceso Node principal usando el módulo incorporado `node:child_process`. Estos procesos hijos pueden comunicarse entre sí con un sistema de mensajería basado en eventos porque los objetos process implementan la estructura `EventEmitter` internamente.

El módulo `node:child_process` nos permite ejecutar eficientemente comandos del sistema operativo (como `find`, `grep`, etc.) desde un proceso Node, controlar de forma segura los argumentos pasados a estos comandos y hacer lo que queramos con la salida de los comandos. Esto le da a un proceso Node mucho más poder sobre lo que puede hacer.

Además, el módulo `node:child_process` soporta tanto buffers como streams para procesar la entrada y salida de los procesos hijo. Con streams, podemos combinar el poder de múltiples comandos y ejecutarlos juntos incluso si la tarea implica trabajar con grandes cantidades de datos. Podemos simplemente canalizar (pipe) la salida de un comando como la entrada de otro.

El módulo `node:child_process` ofrece cuatro formas diferentes de crear un proceso hijo: `spawn()`, `fork()`, `exec()` y `execFile()`. Vamos a ver las diferencias entre estas cuatro funciones y cuándo usar cada una.

---

## La Función spawn

La función `spawn` lanza un comando en un nuevo proceso, y podemos usarla para pasarle cualquier argumento a ese comando. Por ejemplo, aquí hay código para crear un nuevo proceso que ejecutará el comando `pwd` de Linux:

```js linenums="1"
import { spawn } from 'node:child_process';

const child = spawn('pwd');
```

Importamos la función `spawn` y la ejecutamos con el comando del SO como primer argumento.

El resultado de ejecutar la función `spawn` (el objeto `child` en este ejemplo) es una instancia de `ChildProcess`, que implementa la API de `EventEmitter`. Esto significa que podemos registrar manejadores para eventos en este objeto `child` directamente. Por ejemplo, podemos hacer algo cuando el proceso hijo termina registrando un manejador para el evento `exit`:

```js linenums="1"
child.on('exit', function(code, signal) {
  console.log(
    `Proceso hijo terminado. Código: ${code} - Señal: ${signal}`
  );
});
```

La función manejadora para un evento `exit` recibe el código de salida y la señal que se usó para terminar el proceso hijo. La variable `signal` es `null` cuando el proceso hijo termina normalmente.

Otro evento importante de manejar es el evento `error`, que se emite si el proceso no pudo ser creado (o terminado), o si tuvo algún problema durante la ejecución. Es una buena práctica tener siempre un manejador para este evento:

```js linenums="1"
child.on('error', (err) => {
  console.error(`El proceso hijo encontró un error: ${err.message}`);
});
```

Los otros eventos para los que podemos registrar manejadores con las instancias de `ChildProcess` son `message`, `spawn`, `disconnect` y `close`:

**El evento `message`**
: Se emite cuando el proceso hijo usa la función `process.send()` para enviar mensajes. Así es como los procesos padre/hijo pueden comunicarse entre sí. Veremos un ejemplo de eso en breve.

**El evento `spawn`**
: Se emite una vez que el proceso hijo se ha creado exitosamente.

**El evento `disconnect`**
: Se emite cuando el proceso padre llama manualmente al método `child.disconnect`.

**El evento `close`**
: Se emite cuando los streams de E/S de un proceso hijo se terminan. Cada proceso hijo recibe los tres streams stdio estándar, a los que podemos acceder usando `child.stdin`, `child.stdout` y `child.stderr`. Cuando estos streams se cierran, el proceso hijo que los estaba usando emite el evento `close`.

!!! note

    Este evento `close` es diferente del evento `exit` porque múltiples procesos hijos podrían compartir los mismos streams stdio, por lo que un proceso hijo que termina no significa que los streams se hayan cerrado.

Dado que todos los streams son emisores de eventos, podemos escuchar diferentes eventos en los streams stdio que están adjuntos a cada proceso hijo. Sin embargo, a diferencia de un proceso normal, en un proceso hijo los streams `stdout`/`stderr` son streams legibles mientras que el stream `stdin` es uno escribible. Esto es básicamente el inverso de esos tipos encontrados en un proceso principal.

Los eventos que podemos usar para los streams stdio son los estándar. Más importante aún, en los streams legibles podemos escuchar el evento `data`, que tendrá la salida del comando o cualquier error encontrado durante la ejecución del comando:

```js linenums="1"
child.stdout.on('data', data => {
  console.log(`stdout del hijo:\n${data}`);
});

child.stderr.on('data', data => {
  console.error(`stderr del hijo:\n${data}`);
});
```

Los dos manejadores en este ejemplo registrarán ambos casos en el stdout y stderr del proceso principal. Si ejecutamos el código tal como está, la salida del comando `pwd` se imprime y el proceso hijo termina con código 0, lo que significa que no ocurrió ningún error.

Podemos pasar argumentos al comando que es ejecutado por la función `spawn` usando el segundo argumento de la función `spawn`, que es un array de todos los argumentos a pasar al comando. Por ejemplo, para ejecutar el comando `find` en el directorio actual con un argumento `-type f` (para listar solo archivos), podemos hacer:

```js linenums="1"
const child = spawn('find', ['.', '-type', 'f']);
```

Si ocurre un error durante la ejecución del comando (por ejemplo, si le damos al comando `find` anterior un destino inválido), el manejador del evento `data` en el stream `child.stderr` se activará, y el manejador del evento `exit` en esa instancia reportará un código de salida de 1, que significa que ha ocurrido un error. Los valores de error realmente dependen del SO anfitrión y del tipo de error.

El `stdin` de un proceso hijo es un stream escribible. Podemos usarlo para enviar alguna entrada a un comando. Como cualquier stream escribible, la forma más fácil de consumirlo es usando la función `pipe`. Simplemente canalizamos un stream legible hacia un stream escribible. Dado que el `stdin` del proceso principal es un stream legible, podemos canalizarlo hacia el stream `stdin` de un proceso hijo. Aquí hay un ejemplo:

```js linenums="1"
import { spawn } from 'node:child_process';

const child = spawn('wc');

process.stdin.pipe(child.stdin);

child.stdout.on('data', data => {
  console.log(`stdout del hijo:\n${data}`);
});
```

En este ejemplo, el proceso hijo invoca el comando `wc`, que cuenta líneas, palabras y caracteres en Linux. Luego canalizamos el `stdin` del proceso principal (que es un stream legible) hacia el `stdin` del proceso hijo (que es un stream escribible). El resultado de esta combinación es que obtenemos un modo de entrada estándar donde podemos escribir algo, y cuando presionamos Ctrl + D, lo que escribimos se usará como entrada del comando `wc`.

También podemos canalizar la E/S estándar de múltiples procesos entre sí, igual que podemos hacer con los comandos de Linux. Por ejemplo, podemos canalizar el stdout del comando `find` hacia el stdin del comando `wc` para contar todos los archivos en el directorio actual:

```js linenums="1"
import { spawn } from 'node:child_process';

const find = spawn('find', ['.', '-type', 'f']);
const wc = spawn('wc', ['-l']);

find.stdout.pipe(wc.stdin);

wc.stdout.on('data', data => {
  console.log(`Número de archivos ${data}`);
});
```

Agregué el argumento `-l` al comando `wc` para hacer que cuente solo las líneas. Cuando se ejecuta, el código anterior mostrará un recuento de todos los archivos en todos los directorios bajo el actual.

---

## Sintaxis de Shell y la Función exec

Por defecto, la función `spawn` no crea un shell para ejecutar el comando que le pasamos. Esto la hace ligeramente más eficiente que la función `exec`, que sí crea un shell. La función `exec` tiene otra diferencia importante. Almacena en buffer la salida generada por el comando y pasa el valor completo de la salida a una función callback (en lugar de usar streams, que es lo que hace `spawn`).

Dado que la función `exec` usa un shell para ejecutar el comando, podemos usar cualquier sintaxis de shell directamente con ella. La sintaxis de shell es el conjunto de reglas que guían cómo se estructuran e interpretan los comandos en un entorno de shell. Por ejemplo, los comandos pueden tener opciones y argumentos.

También hay algunos caracteres especiales que podemos usar con los comandos. Por ejemplo, el símbolo de pipe (`|`) se puede usar para conectar la salida de un comando a la entrada de otro, el carácter `*` se puede usar en algunos comandos para coincidir archivos basados en patrones, y el carácter `>` se puede usar para redirigir la salida de un comando. Incluso puedes usar bucles y condicionales y muchas otras características avanzadas en comandos de shell.

Aquí está el ejemplo anterior de `find`/`wc` implementado con una función `exec`:

```js linenums="1"
import { exec } from 'node:child_process';

exec('find . -type f | wc -l', (err, stdout, stderr) => {
  if (err) {
    console.error(`exec error: ${err}`);
    return;
  }
  console.log(`Número de archivos ${stdout}`);
});
```

Nota cómo usé la función nativa de pipe del shell a través del símbolo de pipe (`|`) en este ejemplo.

!!! warning

    Ten cuidado al usar la sintaxis de shell en Node, ya que conlleva un riesgo de seguridad, especialmente si estás ejecutando cualquier entrada proporcionada externamente. Un usuario puede iniciar un ataque de inyección de comandos usando caracteres de sintaxis de shell como `;` (por ejemplo, `comando + '; rm -rf ~'`). Las estrategias para mitigar esto incluyen sanitizar y validar la entrada, hacer una lista de comandos aceptables y restringir el control de acceso.

La función `exec` almacena en buffer la salida y la pasa a la función callback (el segundo argumento de `exec`) como el argumento `stdout`. Este argumento `stdout` es la salida del comando que queremos imprimir.

La función `exec` es una buena opción si necesitas usar la sintaxis de shell y el tamaño de los datos esperados del comando es pequeño, ya que almacena en buffer todos los datos en memoria antes de devolverlos.

La función `spawn`, con sus objetos stream, es una opción mucho mejor cuando el tamaño de la salida esperada del comando es grande. Usar streams es compatible con los objetos de E/S estándar que a menudo se usan con procesos hijo. Incluso podemos hacer que un proceso hijo creado con `spawn` herede los objetos de E/S estándar de su padre si queremos.

Más importante aún, podemos hacer que la función `spawn` use la sintaxis de shell también. Aquí está el mismo comando `find | wc` implementado con la función `spawn`:

```js linenums="1"
const child = spawn('find . -type f | wc -l', {
  stdio: 'inherit',
  shell: true
});
```

Con la opción `stdio: 'inherit'`, cuando ejecutamos el código, el proceso hijo hereda el `stdin`, `stdout` y `stderr` del proceso principal. Esto hace que los manejadores de eventos de datos del proceso hijo se activen en el stream `process.stdout` del proceso principal, haciendo que el script muestre el resultado de inmediato.

La opción `shell: true` nos permite usar la sintaxis de shell en el comando ejecutado, igual que hicimos con `exec`. Pero con este código, todavía obtenemos la ventaja del streaming de datos que nos da la función `spawn`. Esto es realmente lo mejor de ambos mundos.

Hay algunas otras opciones útiles que podemos usar en el último argumento de las funciones de `node:child_process` además de `shell` y `stdio`. Por ejemplo, podemos usar la opción `cwd` para cambiar el directorio de trabajo del script. Aquí está el mismo ejemplo de contar todos los archivos hecho con una función `spawn` usando un shell y con un directorio de trabajo establecido en mi carpeta de Descargas:

```js linenums="1"
const child = spawn('find . -type f | wc -l', {
  stdio: 'inherit',
  shell: true,
  cwd: '/Users/samer/Downloads'
});
```

Otra opción que podemos usar es `env` para especificar las variables de entorno que serán visibles para el nuevo proceso hijo. El valor predeterminado para esta opción es `process.env`, que le da a cualquier comando acceso al entorno del proceso actual. Si queremos anular ese comportamiento, podemos simplemente pasar un objeto vacío como la opción `env` o nuevos valores allí para que sean considerados como las únicas variables de entorno:

```js linenums="1"
const child = spawn('echo $ANSWER', {
  stdio: 'inherit',
  shell: true,
  env: { ANSWER: 42 }
});
```

Dado que especificamos un objeto `env`, el comando `echo` no tendrá acceso a las variables de entorno del proceso padre. Por ejemplo, no puede acceder a `$HOME`, pero puede acceder a `$ANSWER` porque fue definida en la propiedad `env`.

Una última opción importante de proceso hijo para explicar aquí es la opción `detached`, que hace que el proceso hijo se ejecute independientemente de su proceso padre.

Supongamos que tenemos un archivo `timer.js` que mantiene ocupado el event loop:

```js linenums="1"
setTimeout(() => {
  // Mantener el event loop ocupado
}, 20_000);
```

Podemos ejecutar `timer.js` en segundo plano usando la opción `detached`:

```js linenums="1"
import { spawn } from 'node:child_process';

const child = spawn('node', ['timer.js'], {
  detached: true,
  stdio: 'ignore'
});

child.unref();
```

El comportamiento exacto de los procesos hijos detached depende del SO. En Windows, tendrán su propia ventana de consola, mientras que en Linux serán los líderes de nuevos grupos de procesos y sesiones.

Si se llama a la función `unref` en el proceso detached, el proceso padre puede terminar independientemente del hijo. Esto puede ser útil si el hijo está ejecutando un proceso de larga duración, pero para mantenerlo ejecutándose en segundo plano, las configuraciones de stdio del hijo también tienen que ser independientes del padre.

Este ejemplo ejecutará el script `timer.js` en segundo plano desvinculándolo (detaching) e ignorando también los descriptores de archivo stdio de su padre para que el padre pueda terminar mientras el hijo continúa ejecutándose en segundo plano.

---

## La Función execFile

La función `execFile` se comporta exactamente como la función `exec`, pero no usa un shell, lo que la hace un poco más eficiente y segura (porque no tiene sintaxis de shell). Podemos usarla para ejecutar un programa externo o un script. Aquí hay un ejemplo:

```js linenums="1"
import { execFile } from 'node:child_process';

execFile('ruby', ['-e', 'puts rand(1..100)'], (error, stdout, stderr) => {
  if (error) {
    console.error(`Error: ${error}`);
    return;
  }
  console.log(`Output: ${stdout}`);
});
```

Este ejemplo usa `execFile` para ejecutar el programa `ruby`, estableciendo los argumentos para ejecutar un one-liner que imprimirá un número aleatorio entre 1 y 100.

!!! warning

    En Windows, algunos archivos no se pueden ejecutar por sí solos, como los archivos `.bat` o `.cmd`. Esos archivos no se pueden ejecutar con `execFile`. Se necesita `exec` o `spawn` con `shell` establecido en `true` para ejecutarlos.

---

## Funciones Síncronas de Proceso Hijo

Las funciones `spawn`, `exec` y `execFile` del módulo `node:child_process` también tienen versiones síncronas bloqueantes que esperarán hasta que el proceso hijo termine:

```js linenums="1"
import {
  spawnSync,
  execSync,
  execFileSync
} from 'node:child_process';
```

Esas versiones síncronas son potencialmente útiles cuando se intenta simplificar tareas de scripting o cualquier tarea de procesamiento de inicio, pero deben evitarse en otros casos.

---

## La Función fork

La función `fork` es una variación de la función `spawn` para crear procesos node. La mayor diferencia entre `spawn` y `fork` es que se establece un canal para la comunicación entre procesos (IPC) con el proceso hijo cuando se usa `fork`, por lo que podemos usar la función `send` en el proceso bifurcado junto con el propio objeto `process` para intercambiar mensajes entre los procesos padre y bifurcados. Hacemos esto a través de la interfaz del módulo `EventEmitter`. Aquí hay un ejemplo.

En el archivo padre, `parent.js`, tenemos lo siguiente:

```js linenums="1"
import { fork } from 'node:child_process';

const forked = fork('child.js');

forked.on('message', msg => {
  console.log('Mensaje del hijo', msg);
});

forked.send({ hello: 'world' });
```

En el archivo hijo, `child.js`, tenemos lo siguiente:

```js linenums="1"
process.on('message', msg => {
  console.log('Mensaje del padre:', msg);
});

let counter = 0;

setInterval(() => {
  process.send({ counter: counter++ });
}, 1_000);
```

En el archivo padre, bifurcamos `child.js` (que ejecutará el archivo con el comando `node`), y luego escuchamos el evento `message`. El evento `message` se emitirá cada vez que el hijo use `process.send`, lo cual estamos haciendo cada segundo.

Para enviar mensajes del padre al hijo, podemos ejecutar la función `send` en el propio objeto bifurcado. Luego, en el script hijo, podemos escuchar el evento `message` en el objeto `process`.

Cuando se ejecuta el archivo `parent.js` anterior, primero enviará el objeto `{ hello: 'world' }` para que sea impreso por el proceso hijo bifurcado. Luego, el proceso hijo bifurcado enviará un valor de contador incrementado cada segundo para que sea impreso por el proceso padre.

Veamos un ejemplo más práctico del uso de la función `fork`. Digamos que tenemos un servidor web que maneja dos endpoints. Uno de estos endpoints (`/compute`) es computacionalmente costoso y tomará algunos segundos en completarse. Podemos usar un bucle `for` largo para simular eso:

```js linenums="1"
import { createServer } from 'node:http';

const longComputation = () => {
  let sum = 0;
  for (let i = 0; i < 1e9; i++) {
    sum += i;
  }
  return sum;
};

const server = createServer();

server.on('request', (req, res) => {
  if (req.url === '/compute') {
    const sum = longComputation();
    return res.end(`Sum is ${sum}`);
  } else {
    res.end('Ok');
  }
});

server.listen(3000, () => {
  console.log('Server is running...');
});
```

Este programa tiene un gran problema; cuando se solicita el endpoint `/compute`, el servidor no podrá manejar ninguna otra solicitud porque el hilo principal está ocupado con la operación del bucle `for` largo.

Hay varias formas de resolver este problema dependiendo de la naturaleza de la operación larga, pero una solución que funciona para todos los casos es simplemente mover la operación computacional a otro proceso usando la función `fork`.

Primero movemos toda la función `longComputation` a su propio archivo y hacemos que invoque esa función cuando se le indique a través de un evento `message` del proceso principal. En un nuevo archivo `compute.js`:

```js linenums="1"
const longComputation = () => {
  let sum = 0;
  for (let i = 0; i < 1e9; i++) {
    sum += i;
  }
  return sum;
};

process.on('message', msg => {
  const sum = longComputation();
  process.send(sum);
});
```

Ahora, en lugar de hacer la operación larga en el hilo principal, podemos bifurcar el archivo `compute.js` y usar la interfaz de mensajes para comunicar mensajes entre el servidor y el proceso bifurcado:

```js linenums="1"
import { createServer } from 'node:http';
import { fork } from 'node:child_process';

const server = createServer();

server.on('request', (req, res) => {
  if (req.url === '/compute') {
    const compute = fork('compute.js');
    compute.send('start');
    compute.on('message', sum => {
      res.end(`Sum is ${sum}`);
    });
  } else {
    res.end('Ok');
  }
});

server.listen(3000, () => {
  console.log('Server is running...');
});
```

Cuando ocurre una solicitud a `/compute` ahora con este código, simplemente enviamos un mensaje al proceso bifurcado para que comience a ejecutar la operación larga. El hilo principal no se bloqueará.

Una vez que el proceso bifurcado termina con esa operación larga, puede enviar su resultado de vuelta al proceso padre usando `process.send`.

En el proceso padre, escuchamos el evento `message` en el propio proceso hijo bifurcado. Cuando recibimos ese evento, tendremos un valor `sum` listo para enviar al usuario solicitante a través de HTTP.

Este método está limitado por el número de procesos que podemos bifurcar, pero cuando lo ejecutamos y solicitamos el endpoint de cómputo largo a través de HTTP, el servidor principal no se bloquea en absoluto y puede aceptar más solicitudes.

El módulo `cluster` de Node, que es el tema del Capítulo 9, está basado en esta idea de bifurcación de procesos hijo y equilibrio de carga de las solicitudes entre las múltiples bifurcaciones que podemos crear en cualquier sistema.

---

## Resumen

Podemos crear procesos hijo en Node para escalar aplicaciones más allá de un solo hilo y acceder a funcionalidades externas del SO y otros programas.

Los procesos hijo de Node nos permiten ejecutar comandos de shell externos, programas y scripts. También podemos establecer comunicación entre procesos hijo y su proceso padre cuando sea necesario.

El módulo `child_process` en Node tiene varias funciones asíncronas flexibles como `spawn`, `exec` y `execFile`, y versiones síncronas correspondientes que esperarán hasta que el proceso hijo termine. La función `spawn` se puede usar para transmitir (stream) la salida de la ejecución, lo que la hace adecuada para manejar salidas grandes. La sintaxis de shell se puede usar en `spawn` y `exec`.

La función `fork` se puede usar para delegar la ejecución de un script Node a un proceso diferente y liberar al proceso principal para hacer más cosas.

Los objetos de proceso hijo son emisores de eventos. Emiten eventos como `exit`, `error` y `message`. El evento `message` es cómo un proceso hijo puede comunicarse con su proceso padre.

En el próximo capítulo, aprenderemos sobre los módulos incorporados de Node para escribir tests y aserciones.


