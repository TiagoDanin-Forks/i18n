# Código Púny
<!-- YAML
changes:
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7941
    description: Accessing this module will now emit a deprecation warning.
-->

<!--introduced_in=v0.10.0-->

> Estabilidad: 0 - Desaprobado

**The version of the punycode module bundled in Node.js is being deprecated**. Este módulo será eliminado en una versión futura de Node.js. Users currently depending on the `punycode` module should switch to using the userland-provided [Punycode.js](https://github.com/bestiejs/punycode.js) module instead.

El módulo `punycode` es una versión empaquetada del módulo [Punycode.js](https://github.com/bestiejs/punycode.js). Se puede acceder a través de:

```js
const punycode = require('punycode');
```

[Punycode](https://tools.ietf.org/html/rfc3492) es un esquema de codificación de caracteres definido por RFC 3492 que está diseñado para ser utilizado en Nombres de Dominios Internacionalizados. Ya que los nombres de hosts en los URLs se limitan a caracteres ASCII únicamente, los Nombres de Dominio que contengan caracteres no ASCII deberán ser convertidos a ASCII utilizando el esquema de Punycode. Por ejemplo, el carácter japonés que se traduce en la palabra del inglés `'example'` es `'例'`. El Nombre de Dominio Internacionalizado `'例.com'` (equivalente a `'example.com'`) es representado por Punycode como la string de ASCII `'xn--fsq.com'`.

El módulo `punycode` proporciona una implementación sencilla del estándar de Punycode.

The `punycode` module is a third-party dependency used by Node.js and made available to developers as a convenience. Fixes or other modifications to the module must be directed to the [Punycode.js](https://github.com/bestiejs/punycode.js) project.

## `punycode.decode(string)`
<!-- YAML
added: v0.5.1
-->

* `string` {string}

El método `punycode.decode()` convierte una string de [Punycode](https://tools.ietf.org/html/rfc3492) de solo caracteres ASCII a la string equivalente de puntos de código de Unicode.

```js
punycode.decode('maana-pta'); // 'mañana'
punycode.decode('--dqo34k'); // '☃-⌘'
```

## `punycode.encode(string)`
<!-- YAML
added: v0.5.1
-->

* `string` {string}

El método `punycode.encode()` convierte una string de puntos de código de Unicode a una string de [Punycode](https://tools.ietf.org/html/rfc3492) de únicamente caracteres ASCII.

```js
punycode.encode('mañana'); // 'maana-pta'
punycode.encode('☃-⌘'); // '--dqo34k'
```

## `punycode.toASCII(domain)`
<!-- YAML
added: v0.6.1
-->

* `domain` {string}

El método `punycode.toASCII()` convierte una string de Unicode que representa un Nombre de Dominio Internacionalizado para [Punycode](https://tools.ietf.org/html/rfc3492). Solo serán convertidas las partes del nombre de dominio que no sean ASCII. Llamar a `punycode.toASCII()` en una string que ya contiene únicamente caracteres ASCII no tendrá ningún efecto.

```js
// codificar nombres de dominio
punycode.toASCII('mañana.com');  // 'xn--maana-pta.com'
punycode.toASCII('☃-⌘.com');   // 'xn----dqo34k.com'
punycode.toASCII('example.com'); // 'example.com'
```

## `punycode.toUnicode(domain)`
<!-- YAML
added: v0.6.1
-->

* `domain` {string}

El método `punycode.toUnicode()` convierte una string que representa un nombre de dominio que contiene caracteres codificados en [Punycode](https://tools.ietf.org/html/rfc3492) en Unicode. Solo son convertidas las partes del nombre de dominio codificadas en [Punycode](https://tools.ietf.org/html/rfc3492).

```js
// decodificar los nombres de dominio
punycode.toUnicode('xn--maana-pta.com'); // 'mañana.com'
punycode.toUnicode('xn----dqo34k.com');  // '☃-⌘.com'
punycode.toUnicode('example.com');       // 'example.com'
```

## `punycode.ucs2`
<!-- YAML
added: v0.7.0
-->

### `punycode.ucs2.decode(string)`
<!-- YAML
added: v0.7.0
-->

* `string` {string}

El método `punycode.ucs2.decode()` devuelve una matriz que contiene los valores de puntos de código numéricos de cada símbolo de Unicode en la string.

```js
punycode.ucs2.decode('abc'); // [0x61, 0x62, 0x63]
// par suplente para el tetragrama U+1D30 para el centro:
punycode.ucs2.decode('\uD834\uDF06'); // [0x1D306]
```

### `punycode.ucs2.encode(codePoints)`
<!-- YAML
added: v0.7.0
-->

* `codePoints` {integer[]}

El método `punycode.ucs2.encode()` devuelve una string basada en una matriz de valores de puntos de código numéricos.

```js
punycode.ucs2.encode([0x61, 0x62, 0x63]); // 'abc'
punycode.ucs2.encode([0x1D306]); // '\uD834\uDF06'
```

## `punycode.version`
<!-- YAML
added: v0.6.1
-->

* {string}

Devuelve una string que identifica el número de la versión actual de [Punycode.js](https://github.com/bestiejs/punycode.js) .
