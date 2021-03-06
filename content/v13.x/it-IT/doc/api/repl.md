# REPL

<!--introduced_in=v0.10.0-->

> Stabilità: 2 - Stable

Il modulo `repl` fornisce un'implementazione Read-Eval-Print-Loop (REPL) che è disponibile sia come programma standalone che incluso in altre applicazioni. Ci si può accedere utilizzando:

```js
const repl = require('repl');
```

## Design e Caratteristiche

The `repl` module exports the [`repl.REPLServer`][] class. While running, instances of [`repl.REPLServer`][] will accept individual lines of user input, evaluate those according to a user-defined evaluation function, then output the result. Input and output may be from `stdin` and `stdout`, respectively, or may be connected to any Node.js [stream](stream.html).

Instances of [`repl.REPLServer`][] support automatic completion of inputs, completion preview, simplistic Emacs-style line editing, multi-line inputs, [ZSH](https://en.wikipedia.org/wiki/Z_shell)-like reverse-i-search, [ZSH](https://en.wikipedia.org/wiki/Z_shell)-like substring-based history search, ANSI-styled output, saving and restoring current REPL session state, error recovery, and customizable evaluation functions. Terminals that do not support ANSI styles and Emacs-style line editing automatically fall back to a limited feature set.

### Comandi e Tasti Speciali

I seguenti comandi speciali sono supportati da tutte le istanze REPL:

* `.break`: When in the process of inputting a multi-line expression, entering the `.break` command (or pressing the `<ctrl>-C` key combination) will abort further input or processing of that expression.
* `.clear`: Resets the REPL `context` to an empty object and clears any multi-line expression currently being input.
* `.exit`: Close the I/O stream, causing the REPL to exit.
* `.help`: Show this list of special commands.
* `.save`: Save the current REPL session to a file: `> .save ./file/to/save.js`
* `.load`: Load a file into the current REPL session. `> .load ./file/to/load.js`
* `.editor`: Enter editor mode (`<ctrl>-D` to finish, `<ctrl>-C` to cancel).

```console
> .editor
// Entrata nella modalità editor (^D per finire, ^C per cancellare)
function welcome(name) {
  return `Hello ${name}!`;
}

welcome('Node.js User');

// ^D
'Hello Node.js User!'
>
```

Le seguenti combinazioni di tasti nel REPL hanno questi effetti speciali:

* `<ctrl>-C`: When pressed once, has the same effect as the `.break` command. Quando viene premuto due volte su una riga vuota, ha lo stesso effetto del comando `.exit`.
* `<ctrl>-D`: Has the same effect as the `.exit` command.
* `<tab>`: When pressed on a blank line, displays global and local (scope) variables. Quando viene premuto mentre si immette un altro input, visualizza le opzioni di completamento automatico rilevanti.

For key bindings related to the reverse-i-search, see [`reverse-i-search`][]. For all other key bindings, see [TTY keybindings](readline.html#readline_tty_keybindings).

### Valutazione Predefinita

By default, all instances of [`repl.REPLServer`][] use an evaluation function that evaluates JavaScript expressions and provides access to Node.js built-in modules. This default behavior can be overridden by passing in an alternative evaluation function when the [`repl.REPLServer`][] instance is created.

#### Espressioni JavaScript

Il programma di valutazione predefinito supporta la valutazione diretta delle espressioni JavaScript:

```console
> 1 + 1
2
> const m = 2
undefined
> m + 1
3
```

Unless otherwise scoped within blocks or functions, variables declared either implicitly or using the `const`, `let`, or `var` keywords are declared at the global scope.

#### Scope Globale e Locale

Il programma di valutazione predefinito fornisce l'accesso a qualsiasi variabile esistente nello scope globale. It is possible to expose a variable to the REPL explicitly by assigning it to the `context` object associated with each `REPLServer`:

```js
const repl = require('repl');
const msg = 'message';

repl.start('> ').context.m = msg;
```

Le proprietà nel `context` object vengono visualizzate come locali all'interno del REPL:

```console
$ node repl_test.js
> m
'message'
```

Le proprietà del contesto non sono di sola lettura per impostazione predefinita. To specify read-only globals, context properties must be defined using `Object.defineProperty()`:

```js
const repl = require('repl');
const msg = 'message';

const r = repl.start('> ');
Object.defineProperty(r.context, 'm', {
  configurable: false,
  enumerable: true,
  value: msg
});
```

#### Accesso ai Moduli Core Node.js

Quando viene utilizzato, il programma di valutazione predefinito caricherà automaticamente i moduli core Node.js nell'ambiente REPL. Ad esempio, se non diversamente dichiarato come variabile globale o con scope, l'input `fs` sarà valutato su richiesta come `global.fs = require('fs')`.

```console
> fs.createReadStream('./some/file');
```

#### Eccezioni Globali non Rilevate
<!-- YAML
changes:
  - version: v12.3.0
    pr-url: https://github.com/nodejs/node/pull/27151
    description: The `'uncaughtException'` event is from now on triggered if the
                 repl is used as standalone program.
-->

The REPL uses the [`domain`][] module to catch all uncaught exceptions for that REPL session.

Questo uso del modulo [`domain`][] nel REPL ha questi effetti collaterali:

* Uncaught exceptions only emit the [`'uncaughtException'`][] event in the standalone REPL. Adding a listener for this event in a REPL within another Node.js program throws [`ERR_INVALID_REPL_INPUT`][].
* Trying to use [`process.setUncaughtExceptionCaptureCallback()`][] throws an [`ERR_DOMAIN_CANNOT_SET_UNCAUGHT_EXCEPTION_CAPTURE`][] error.

As standalone program:

```js
process.on('uncaughtException', () => console.log('Uncaught'));

throw new Error('foobar');
// Uncaught
```

When used in another application:

```js
process.on('uncaughtException', () => console.log('Uncaught'));
// TypeError [ERR_INVALID_REPL_INPUT]: Listeners for `uncaughtException`
// cannot be used in the REPL

throw new Error('foobar');
// Thrown:
// Error: foobar
```

#### Assegnazione della variabile `_` (underscore)
<!-- YAML
changes:
  - version: v9.8.0
    pr-url: https://github.com/nodejs/node/pull/18919
    description: Added `_error` support.
-->

Per impostazione predefinita, il programma di valutazione predefinito assegnerà il risultato dell'espressione valutata più recentemente alla variabile speciale `_ ` (underscore). L'impostazione esplicita di `_` su un valore disabiliterà questo comportamento.

```console
> [ 'a', 'b', 'c' ]
[ 'a', 'b', 'c' ]
> _.length
3
> _ += 1
Expression assignment to _ now disabled.
4
> 1 + 1
2
> _
4
```

Similarly, `_error` will refer to the last seen error, if there was any. Explicitly setting `_error` to a value will disable this behavior.

```console
> throw new Error('foo');
Error: foo
> _error.message
'foo'
```

#### La parola chiave `await`

With the [`--experimental-repl-await`][] command line option specified, experimental support for the `await` keyword is enabled.

```console
> await Promise.resolve(123)
123
> await Promise.reject(new Error('REPL await'))
Error: REPL await
    at repl:1:45
> const timeout = util.promisify(setTimeout);
undefined
> const old = Date.now(); await timeout(1000); console.log(Date.now() - old);
1002
undefined
```

### Reverse-i-search
<!-- YAML
added: v13.6.0
-->

The REPL supports bi-directional reverse-i-search similar to [ZSH](https://en.wikipedia.org/wiki/Z_shell). It is triggered with `<ctrl> + R` to search backwards and `<ctrl> + S` to search forwards.

Duplicated history entires will be skipped.

Entries are accepted as soon as any button is pressed that doesn't correspond with the reverse search. Cancelling is possible by pressing `escape` or `<ctrl> + C`.

Changing the direction immediately searches for the next entry in the expected direction from the current position on.

### Funzioni di Valutazione Personalizzate

When a new [`repl.REPLServer`][] is created, a custom evaluation function may be provided. Questo può essere utilizzato, ad esempio, per implementare applicazioni REPL completamente personalizzate.

Di seguito viene illustrato un esempio ipotetico di un REPL che esegue la traduzione del testo da una lingua ad un'altra:

```js
const repl = require('repl');
const { Translator } = require('translator');

const myTranslator = new Translator('en', 'fr');

function myEval(cmd, context, filename, callback) {
  callback(null, myTranslator.translate(cmd));
}

repl.start({ prompt: '> ', eval: myEval });
```

#### Errori Recuperabili

Quando un utente digita l'input nel prompt REPL, premendo il tasto `&lt;enter&gt;` invierà la riga corrente di input alla funzione `eval`. Per supportare un input su più righe, la funzione eval può restituire un'istanza di `repl.Recoverable` alla funzione di callback fornita:

```js
function myEval(cmd, context, filename, callback) {
  let result;
  try {
    result = vm.runInThisContext(cmd);
  } catch (e) {
    if (isRecoverableError(e)) {
      return callback(new repl.Recoverable(e));
    }
  }
  callback(null, result);
}

function isRecoverableError(error) {
  if (error.name === 'SyntaxError') {
    return /^(Unexpected end of input|Unexpected token)/.test(error.message);
  }
  return false;
}
```

### Personalizzazione dell'Output REPL

By default, [`repl.REPLServer`][] instances format output using the [`util.inspect()`][] method before writing the output to the provided `Writable` stream (`process.stdout` by default). The `showProxy` inspection option is set to true by default and the `colors` option is set to true depending on the REPL's `useColors` option.

The `useColors` boolean option can be specified at construction to instruct the default writer to use ANSI style codes to colorize the output from the `util.inspect()` method.

If the REPL is run as standalone program, it is also possible to change the REPL's [inspection defaults][`util.inspect()`] from inside the REPL by using the `inspect.replDefaults` property which mirrors the `defaultOptions` from [`util.inspect()`][].

```console
> util.inspect.replDefaults.compact = false;
false
> [1]
[
  1
]
>
```

To fully customize the output of a [`repl.REPLServer`][] instance pass in a new function for the `writer` option on construction. The following example, for instance, simply converts any input text to upper case:

```js
const repl = require('repl');

const r = repl.start({ prompt: '> ', eval: myEval, writer: myWriter });

function myEval(cmd, context, filename, callback) {
  callback(null, cmd);
}

function myWriter(output) {
  return output.toUpperCase();
}
```

## Class: `REPLServer`
<!-- YAML
added: v0.1.91
-->

* `options` {Object|string} See [`repl.start()`][]
* Extends: {readline.Interface}

Instances of `repl.REPLServer` are created using the [`repl.start()`][] method or directly using the JavaScript `new` keyword.

```js
const repl = require('repl');

const options = { useColors: true };

const firstInstance = repl.start(options);
const secondInstance = new repl.REPLServer(options);
```

### Event: `'exit'`
<!-- YAML
added: v0.7.7
-->

L'evento `'exit'` viene emesso quando REPL viene chiuso ricevendo il comando `.exit` come input, con l'utente che preme `&lt;ctrl&gt;-C` due volte per segnalare `SIGINT` o premendo `&lt;ctrl&gt;-D` per segnalare `'end'` sullo stream di input. Il callback del listener viene invocato senza alcun argomento.

```js
replServer.on('exit', () => {
  console.log('Received "exit" event from repl!');
  process.exit();
});
```

### Event: `'reset'`
<!-- YAML
added: v0.11.0
-->

L'evento `'reset'` viene emesso quando il contesto di REPL viene reimpostato. This occurs whenever the `.clear` command is received as input *unless* the REPL is using the default evaluator and the `repl.REPLServer` instance was created with the `useGlobal` option set to `true`. Il callback del listener verrà chiamato con un riferimento al `context` object come unico argomento.

This can be used primarily to re-initialize REPL context to some pre-defined state:

```js
const repl = require('repl');

function initializeContext(context) {
  context.m = 'test';
}

const r = repl.start({ prompt: '> ' });
initializeContext(r.context);

r.on('reset', initializeContext);
```

Quando questo codice viene eseguito, la variabile globale `'m'` può essere modificata ma poi reimpostata al suo valore iniziale usando il comando `.clear`:

```console
$ ./node example.js
> m
'test'
> m = 1
1
> m
1
> .clear
Clearing context...
> m
'test'
>
```

### `replServer.defineCommand(keyword, cmd)`
<!-- YAML
added: v0.3.0
-->

* `keyword` {string} The command keyword (*without* a leading `.` character).
* `cmd` {Object|Function} La funzione da invocare quando il comando viene elaborato.

Il metodo `replServer.defineCommand()` viene utilizzato per aggiungere nuovi comandi `.`-prefissati all'istanza REPL. Tali comandi vengono invocati digitando un `.` seguito dalla `keyword`. The `cmd` is either a `Function` or an `Object` with the following properties:

* `help` {string} Testo di aiuto da visualizzare quando viene inserito `.help` (Opzionale).
* `action` {Function} La funzione da eseguire, accettando facoltativamente un argomento a stringa singola.

L'esempio seguente mostra due nuovi comandi aggiunti all'istanza REPL:

```js
const repl = require('repl');

const replServer = repl.start({ prompt: '> ' });
replServer.defineCommand('sayhello', {
  help: 'Say hello',
  action(name) {
    this.clearBufferedCommand();
    console.log(`Hello, ${name}!`);
    this.displayPrompt();
  }
});
replServer.defineCommand('saybye', function saybye() {
  console.log('Goodbye!');
  this.close();
});
```

I nuovi comandi possono quindi essere utilizzati dall'interno dell'istanza REPL:

```console
> .sayhello Node.js User
Hello, Node.js User!
> .saybye
Goodbye!
```

### `replServer.displayPrompt([preserveCursor])`
<!-- YAML
added: v0.1.91
-->

* `preserveCursor` {boolean}

Il metodo `replServer.displayPrompt()` prepara l'istanza REPL per l'input dall'utente, stampando il `prompt` configurato su una nuova riga nell'`output` e ripristinando l'`input` per accettare un nuovo input.

Quando viene immesso un input su più righe, vengono stampati dei puntini di sospensione anziché il "prompt".

Quando `preserveCursor` è `true`, la posizione del cursore non verrà reimpostata su `0`.

Il metodo `replServer.displayPrompt` è destinato principalmente a essere chiamato dall'interno della funzione di azione per i comandi registrati utilizzando il metodo `replServer.defineCommand()`.

### `replServer.clearBufferedCommand()`
<!-- YAML
added: v9.0.0
-->

The `replServer.clearBufferedCommand()` method clears any command that has been buffered but not yet executed. This method is primarily intended to be called from within the action function for commands registered using the `replServer.defineCommand()` method.

### `replServer.parseREPLKeyword(keyword[, rest])`
<!-- YAML
added: v0.8.9
deprecated: v9.0.0
-->

> Stabilità: 0 - Obsoleto.

* `keyword` {string} the potential keyword to parse and execute
* `rest` {any} any parameters to the keyword command
* Restituisce: {boolean}

An internal method used to parse and execute `REPLServer` keywords. Returns `true` if `keyword` is a valid keyword, otherwise `false`.

### `replServer.setupHistory(historyPath, callback)`
<!-- YAML
added: v11.10.0
-->

* `historyPath` {string} the path to the history file
* `callback` {Function} called when history writes are ready or upon error
  * `err` {Error}
  * `repl` {repl.REPLServer}

Initializes a history log file for the REPL instance. When executing the Node.js binary and using the command line REPL, a history file is initialized by default. However, this is not the case when creating a REPL programmatically. Use this method to initialize a history log file when working with REPL instances programmatically.

## `repl.start([options])`
<!-- YAML
added: v0.1.91
changes:
  - version: v13.4.0
    pr-url: https://github.com/nodejs/node/pull/30811
    description: The `preview` option is now available.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26518
    description: The `terminal` option now follows the default description in
                 all cases and `useColors` checks `hasColors()` if available.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19187
    description: The `REPL_MAGIC_MODE` `replMode` was removed.
  - version: v5.8.0
    pr-url: https://github.com/nodejs/node/pull/5388
    description: The `options` parameter is optional now.
-->

* `options` {Object|string}
  * `prompt` {string} Il prompt dell'input da visualizzare. **Default:** `'> '` (with a trailing space).
  * `input` {stream.Readable} The `Readable` stream from which REPL input will be read. **Default:** `process.stdin`.
  * `output` {stream.Writable} The `Writable` stream to which REPL output will be written. **Default:** `process.stdout`.
  * `terminal` {boolean} If `true`, specifies that the `output` should be treated as a TTY terminal. **Default:** checking the value of the `isTTY` property on the `output` stream upon instantiation.
  * `eval` {Function} La funzione da utilizzare quando si valuta ogni riga specifica di input. **Default:** an async wrapper for the JavaScript `eval()` function. Una funzione `eval` può generare un errore con `repl.Recoverable` nel tentativo di indicare che l'input era incompleto e può richiedere linee aggiuntive.
  * `useColors` {boolean} Se `true`, specifica che la funzione `writer` predefinita deve includere lo stile dei colori ANSI per l'output REPL. Se viene fornita una funzione `writer` personalizzata, ciò non ha alcun effetto. **Default:** checking color support on the `output` stream if the REPL instance's `terminal` value is `true`.
  * `useGlobal` {boolean} Se `true`, specifica che la funzione di valutazione predefinita utilizzerà il `global` di JavaScript come contesto anziché creare un nuovo contesto separato per l'istanza REPL. The node CLI REPL sets this value to `true`. **Default:** `false`.
  * `ignoreUndefined` {boolean} Se `true`, specifica che il writer predefinito non genererà il valore restituito di un comando se si valuta `undefined`. **Default:** `false`.
  * `writer` {Function} La funzione da invocare per formattare l'output di ciascun comando prima di scrivere sull'`output`. **Default:** [`util.inspect()`][].
  * `completer` {Function} Una funzione opzionale utilizzata per il completamento automatico di Tab. Vedi [`readline.InterfaceCompleter`][] per un esempio.
  * `replMode` {symbol} A flag that specifies whether the default evaluator executes all JavaScript commands in strict mode or default (sloppy) mode. I valori accettabili sono:
    * `repl.REPL_MODE_SLOPPY` to evaluate expressions in sloppy mode.
    * `repl.REPL_MODE_STRICT` to evaluate expressions in strict mode. Questo è equivalente all'anteporre `'use strict'` davanti ad ogni istruzione repl.
  * `breakEvalOnSigint` {boolean} Stop evaluating the current piece of code when `SIGINT` is received, such as when `Ctrl+C` is pressed. This cannot be used together with a custom `eval` function. **Default:** `false`.
  * `preview` {boolean} Defines if the repl prints autocomplete and output previews or not. **Default:** `true` with the default eval function and `false` in case a custom eval function is used. If `terminal` is falsy, then there are no previews and the value of `preview` has no effect.
* Returns: {repl.REPLServer}

The `repl.start()` method creates and starts a [`repl.REPLServer`][] instance.

Se `options` è una stringa, allora specifica il prompt di input:

```js
const repl = require('repl');

// un prompt in stile Unix
repl.start('$ ');
```

## Il REPL di Node.js

Node.js stesso utilizza il modulo `repl` per fornire la propria interfaccia interattiva per l'esecuzione di JavaScript. Questo può essere utilizzato eseguendo il binario di Node.js senza passare alcun argomento (o passando l'argomento `-i`):

```console
$ node
> const a = [1, 2, 3];
undefined
> a
[ 1, 2, 3 ]
> a.forEach((v) => {
...   console.log(v);
...   });
1
2
3
```

### Opzioni Variabili d'Ambiente

I vari comportamenti del REPL di Node.js possono essere personalizzati utilizzando le seguenti variabili di ambiente:

* `NODE_REPL_HISTORY`: When a valid path is given, persistent REPL history will be saved to the specified file rather than `.node_repl_history` in the user's home directory. Setting this value to `''` (an empty string) will disable persistent REPL history. Lo spazio bianco sarà aggiustato dal valore. On Windows platforms environment variables with empty values are invalid so set this variable to one or more spaces to disable persistent REPL history.
* `NODE_REPL_HISTORY_SIZE`: Controls how many lines of history will be persisted if history is available. Deve essere un numero positivo. **Default:** `1000`.
* `NODE_REPL_MODE`: May be either `'sloppy'` or `'strict'`. **Default:** `'sloppy'`, which will allow non-strict mode code to be run.

### Cronologia Persistente

Per impostazione predefinita, il REPL di Node.js manterrà la cronologia tra le sessioni REPL del `node` salvando gli input in un file `.node_repl_history` situato nella home directory dell'utente. Questo può essere disabilitato impostando la variabile d'ambiente `NODE_REPL_HISTORY=''`.

### Utilizzo del REPL di Node.js con editor di riga avanzati

Per gli editor di riga avanzati, avviare Node.js con la variabile di ambiente `NODE_NO_READLINE=1`. This will start the main and debugger REPL in canonical terminal settings, which will allow use with `rlwrap`.

Ad esempio, ciò che segue può essere aggiunto a un file `.bashrc`:

```text
alias node="env NODE_NO_READLINE=1 rlwrap node"
```

### Avvio di più istanze REPL su una singola istanza in esecuzione

È possibile creare ed eseguire più istanze REPL su una singola istanza di Node.js in esecuzione che condivide un singolo `global` object ma possiede interfacce I/O separate.

Il seguente caso, ad esempio, fornisce REPL separati su `stdin`, un socket Unix ed un socket TCP:

```js
const net = require('net');
const repl = require('repl');
let connections = 0;

repl.start({
  prompt: 'Node.js via stdin> ',
  input: process.stdin,
  output: process.stdout
});

net.createServer((socket) => {
  connections += 1;
  repl.start({
    prompt: 'Node.js via Unix socket> ',
    input: socket,
    output: socket
  }).on('exit', () => {
    socket.end();
  });
}).listen('/tmp/node-repl-sock');

net.createServer((socket) => {
  connections += 1;
  repl.start({
    prompt: 'Node.js via TCP socket> ',
    input: socket,
    output: socket
  }).on('exit', () => {
    socket.end();
  });
}).listen(5001);
```

L'esecuzione di questa applicazione dalla riga di comando avvierà un REPL su stdin. Altri client REPL possono connettersi tramite il socket Unix o il socket TCP. `telnet`, ad esempio, è utile per la connessione ai socket TCP, mentre `socat` può essere utilizzato per connettersi sia ai socket Unix sia ai TCP.

Avviando un REPL da un server basato su socket Unix invece di stdin, è possibile connettersi a un processo Node.js con esecuzione prolungata senza riavviarlo.

For an example of running a "full-featured" (`terminal`) REPL over a `net.Server` and `net.Socket` instance, see: <https://gist.github.com/TooTallNate/2209310>.

For an example of running a REPL instance over [curl(1)](https://curl.haxx.se/docs/manpage.html), see: <https://gist.github.com/TooTallNate/2053342>.
