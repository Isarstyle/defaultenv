# defaultenv

[![Build Status](https://travis-ci.org/jcoreio/defaultenv.svg?branch=master)](https://travis-ci.org/jcoreio/defaultenv)
[![Coverage Status](https://coveralls.io/repos/github/jcoreio/defaultenv/badge.svg?branch=master)](https://coveralls.io/github/jcoreio/defaultenv?branch=master)
[![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release)
[![Commitizen friendly](https://img.shields.io/badge/commitizen-friendly-brightgreen.svg)](http://commitizen.github.io/cz-cli/)

Takes the suck out of launching your server with various sets of environment variables

## Table of Contents

* [Intro](#intro)
* [Usage](#usage)
  + [`dotenv`-style files](#dotenv-style-files)
  + [`.js` files](#js-files)
  + [`.json` files](#json-files)
  + [Single file](#single-file)
  + [Multiple files](#multiple-files)
  + [Notes](#notes)
  + [Exporting Values](#exporting-values)
  + [Options](#options)
  + [Example](#example)
  + [Node.js API](#node-js-api)

## Intro

Let's say you want to store development environment variables in `dev.env`, production environment variables in
`prod.env`, and then somehow use those to launch your server.  One common way to launch your server is with the `env`
command:

```sh
env $(< dev.env | xargs) runserver
```

This is a start, but it's pretty lackluster:
* it overwrites any variables that are already set in your environment
* it doesn't work on windows (and may not work in terminals besides `bash`...I have no idea)
* it would be nice avoid wrapping `dev.env` in `$(<` `| xargs)`
* if you want to source multiple files, you have to type even more punctuation
* you can't dynamically construct any values with code

`defaultenv` makes this easy:

```sh
defaultenv dev.env runserver
defaultenv devenv.js runserver
defaultenv prod.env staging.env -- runserver
defaultenv prod.env deployment.env -- runserver
```

You can use either `dotenv`-style files, `.js` modules that export an object, or `.json` files to provide environment
variable defaults.

## Usage

### `.env` file

If this is present in the root directory of your project, variables will be loaded from it using
[`dotenv`](https://www.npmjs.com/package/dotenv) unless you provide the `--no-dotenv` option.

### `dotenv`-style files

`defaultenv` uses [`dotenv`](https://www.npmjs.com/package/dotenv) to parse each file.  Here's an example of what
an env file should look like:
```sh
REDIS_HOST=localhost
REDIS_PORT=6379
DB_HOST=localhost
DB_USER=root
DYNAMO_ENDPOINT="http://localhost:8000"
```

### `.js` files

You may use a node `.js` script that exports an object where all property values are strings.  This is especially useful
if you need to dynamically create default values for environment variables with arbitrary code.
You are welcome to refer to `process.env` in the script; default values from prior files will already be applied.

```js
var path = require('path')
exports.BUILD_DIR = path.resolve(__dirname, '..', 'build')
exports.REDIS_HOST = 'localhost'
exports.REDIS_PORT = '6379'
if (process.env.TARGET) {
  exports.BUILD_DIR = path.resolve(exports.BUILD_DIR, process.env.TARGET)
}
```
(or)
```js
module.exports = {
  BUILD_DIR: path.resolve(__dirname, '..', 'build'),
  REDIS_HOST: 'localhost',
  REDIS_PORT: '6379',
}
if (process.env.TARGET) {
  module.exports.BUILD_DIR = path.resolve(module.exports.BUILD_DIR, process.env.TARGET)
}
```

### `.json` files

You may use a JSON file that contains an object where all property values are strings:
```json
{
  "REDIS_HOST": "localhost",
  "REDIS_PORT": "6379",
  "DB_HOST": "localhost",
  "DB_USER": "root",
  "DYNAMO_ENDPOINT": "http://localhost:8000"
}
```

### Single file

```sh
defaultenv <envfile> <command> [...command args]
```

### Multiple files

Just insert `--` between the env files and your command:

```sh
defaultenv <envfile1> <envfile2> [...more envfiles] -- <command> [...command args]
```

### Notes

* if `FOO` appears in more than one of the env files, the leftmost file in your arguments wins
* if `FOO` is already defined in the environment `defaultenv` gets run with, `defaultenv` won't overwrite it
* once an environment variable gets set to the empty string, `defaultenv` won't overwrite it

### Exporting values

Sometimes it's more convenient to just export a bunch of variables to your terminal session so you can run a bunch of
commands with those variables instead of running each command via `defaultenv`.

The `-p` or `--print` option will make `defaultenv` output a bash script that will export the values it loaded from
the envfiles:
```
> defaultenv -p foo.env
export FOO=foo
```

So if you wrap this in `$( )` it will export them to your terminal session:
```
> $(defaultenv -p foo.env)
> echo $FOO
foo
```

### Options

#### `-f`, `--force`
Overwrite existing values for environment variables (by default, existing values will be preserved)
**This does not currently apply to variables loaded via `--dotenv` option.**

#### `-p`, `--print`
See [Exporting Values](#exporting-values)

#### `--no-dotenv`
Prevents loading variables from a .env file in the root directory of your project

### Example

`foo.env`:
```
FOO=foo
```
`bar.env`:
```
FOO=hello
BAR=bar
```
```
> node -p 'process.env.FOO'

> defaultenv foo.env node -p 'process.env.FOO'
foo
> defaultenv bar.env node -p 'process.env.FOO'
hello
> defaultenv foo.env bar.env -- node -p 'process.env.FOO'
foo
> FOO=test defaultenv foo.env node -p 'process.env.FOO'
test
> defaultenv foo.env bar.env -- node -p 'process.env.FOO + process.env.BAR'
foobar
```

### Node.js API

```js
declare function defaultEnv(files: Array<string>, options: {
  force?: boolean,
  print?: boolean,
  noDotenv?: boolean,
  noExport?: boolean,
})
```

```js
var path = require('path')
var defaultEnv = require('defaultEnv')

var defaults = defaultEnv([path.resolve('foo.js')], {noExport: true})
```

#### Options

#### `force`
If true, overwrite existing values for environment variables (by default, existing values will be preserved)
**This does not currently apply to variables loaded via `--dotenv` option.**

#### `print`
See [Exporting Values](#exporting-values)

#### `noDotenv`
Unless this is true, also loads variables from a .env file in the root directory of your project
(by using `require('dotenv').config()`)

##### `noExport`
If true, doesn't write to `process.env`; only returns the values that would be written.

