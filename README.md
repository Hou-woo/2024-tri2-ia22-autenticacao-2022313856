# Autenticação de Usuários (single server)

# Autenticação VS Autorização

- Autenticação é o ato de identificar um usuário ou dispositivo, enquanto autorização é o ato de permitir ou negar direitos de acesso a usuários e dispositivos.
A autenticação pode ser usada como um fator nas decisões de autorização.
Artefatos de autorização não são úteis para identificar usuários ou dispositivos.

# Autenticação com Token (JWT)

- O JWT (JSON Web Token) é uma forma de autenticação que permite que um servidor verifique a identidade de um usuário sem precisar armazenar informações sobre ele.

# Projeto (Objeto de Estudos)

- O objeto de estudo é uma especificação que deriva de temáticas mais amplas.

# gitgnore

node_modules/
dist/

database.sqlite

# Database

...

import { open, Database } from 'sqlite'
import sqlite3 from 'sqlite3'

let instance: Database | null = null

export async function connect() {
  if (instance) 
    return instance

  const db = await open({
    filename: 'database.sqlite',
    driver: sqlite3.Database
  })
  
  await db.exec(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT,
      email TEXT
    )
  `)

  instance = db
  return db
}

...

# Index.ts

import express from 'express'
import { connect } from './database'

const port = 3000
const app = express()

app.use(express.json())
app.use(express.static(__dirname + '/../public'))

app.get('/users', async (req, res) => {
  const db = await connect()
  const users = await db.all('SELECT * FROM users')
  res.json(users)
})

app.post('/users', async (req, res) => {
  const db = await connect()
  const { name, email } = req.body
  const result = await db.run('INSERT INTO users (name, email) VALUES (?, ?)', [name, email])
  const user = await db.get('SELECT * FROM users WHERE id = ?', [result.lastID])
  res.json(user)
})

app.put('/users/:id', async (req, res) => {
  const db = await connect()
  const { name, email } = req.body
  const { id } = req.params
  await db.run('UPDATE users SET name = ?, email = ? WHERE id = ?', [name, email, id])
  const user = await db.get('SELECT * FROM users WHERE id = ?', [id])
  res.json(user)
})

app.delete('/users/:id', async (req, res) => {
  const db = await connect()
  const { id } = req.params
  await db.run('DELETE FROM users WHERE id = ?', [id])
  res.json({ message: 'User deleted' })
})

app.listen(port, () => {
  console.log(`Server running on port ${port}`)
})

...

# Index.html

<!DOCTYPE html>
<html lang="pt-br">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Cadastro de Usuários</title>
  <link rel="stylesheet" href="main.css">
  <script src="main.js" defer></script>
</head>
<body>
  <h1>Cadastro de Usuários</h1>
  <main>
    <form>
      <div class="input-container">
        <label for="name">Nome</label>
        <input type="text" name="name">
      </div>
      <div class="input-container">
        <label for="email">E-mail</label>
        <input type="email" name="email">
      </div>
      <div class="actions-container">
        <button data-action="delete">Excluir</button>
        <button data-action="update">Alterar</button>
        <button data-action="create">Cadastrar</button>
      </div>
    </form>
  </main>
</body>
</html>

...

# Main.css

* {
  box-sizing: border-box;
}

:root {
  font-family: Arial, sans-serif;
}

body {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  margin: 0;
  padding: 1rem;
  min-height: 100vh;
  background-color: #f0f0f0;
}

main {
  display: flex;
  gap: .5rem;
  flex-wrap: wrap;
  justify-content: center;
}

form {
  background-color: #fff;
  display: inline-block;
  padding: 1.5rem;
  margin: 0;
  border-radius: 5px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  width: 350px;
  flex: none;
}

form[data-id] {
  & button[data-action=create] {
    display: none;
  }
}

form:not([data-id]) {
  & button[data-action=update],
  & button[data-action=delete] {
    display: none;
  }
}

form .input-container {
  margin-bottom: 1rem;
}

form label {
  display: block;
  font-weight: bold;
  margin-bottom: .25rem;
}

form input {
  width: 100%;
  padding: .6rem;
  border: 1px solid #ccc;
  border-radius: 5px;
}

form .actions-container {
  display: flex;
  justify-content: flex-end;
  gap: .5rem;
}

form button {
  padding: 10px 20px;
  border: 0;
  border-radius: 5px;
  color: #fff;
  font-weight: bold;
  cursor: pointer;
}

form button[data-action=cancel] {
  background-color: transparent;
  color: #dc3545;
  margin-right: auto;
  padding-inline: .5rem;
}

form button[data-action=update] {
  background-color: #007bff;
}

form button[data-action=delete] {
  background-color: #dc3545;
}

form button[data-action=create] {
  background-color: #28a745;
}

...

# Main.js

const mainForm = document.querySelector('form')

void async function () {
  const response = await fetch('/users')
  const users = await response.json()
  users.forEach(user => {
    const newForm = mainForm.cloneNode(true)
    newForm.name.value = user.name
    newForm.email.value = user.email
    newForm.dataset.id = user.id
    newForm.id.readOnly = true
    mainForm.before(newForm)
  })
  console.log(users)
}()

document.addEventListener('submit', async (event) => {
  event.preventDefault()
  const action = event.submitter.dataset.action ?? null
  const currentForm = event.target
  
  if (action === 'delete') {
    const id = currentForm.dataset.id
    const method = 'DELETE'
    const url = `/users/${id}`
    const response = await fetch(url, { method })
    if (!response.ok)
      return console.error('Error:', response.statusText)
    currentForm.remove()
    return
  }
  
  if (action === 'update') {
    const id = currentForm.dataset.id
    const method = 'PUT'
    const url = `/users/${id}`
    const headers = { 'Content-Type': 'application/json' }
    const name = currentForm.name.value
    const email = currentForm.email.value
    const body = JSON.stringify({ name, email })
    const response = await fetch(url, { method, headers, body, })
    if (!response.ok)
      return console.error('Error:', response.statusText)
    const responseData = await response.json()
    currentForm.name.value = responseData.name
    currentForm.email.value = responseData.email
    return
  }
  
  if (action === 'create') {
    const method = 'POST'
    const url = '/users'
    const headers = { 'Content-Type': 'application/json' }
    const name = currentForm.name.value
    const email = currentForm.email.value
    const body = JSON.stringify({ name, email })
    const response = await fetch(url, { method, headers, body })
    if (!response.ok)
      return console.error('Error:', response.statusText)
    const responseData = await response.json()
    const newForm = mainForm.cloneNode(true)
    newForm.name.value = responseData.name
    newForm.email.value = responseData.email
    newForm.dataset.id = responseData.id
    newForm.id.readOnly = true
    mainForm.reset()
    mainForm.before(newForm)
    return
  }
})

...

# Tsconfig.js

{
  "compilerOptions": {
    /* Visit https://aka.ms/tsconfig to read more about this file */

    /* Projects */
    // "incremental": true,                              /* Save .tsbuildinfo files to allow for incremental compilation of projects. */
    // "composite": true,                                /* Enable constraints that allow a TypeScript project to be used with project references. */
    // "tsBuildInfoFile": "./.tsbuildinfo",              /* Specify the path to .tsbuildinfo incremental compilation file. */
    // "disableSourceOfProjectReferenceRedirect": true,  /* Disable preferring source files instead of declaration files when referencing composite projects. */
    // "disableSolutionSearching": true,                 /* Opt a project out of multi-project reference checking when editing. */
    // "disableReferencedProjectLoad": true,             /* Reduce the number of projects loaded automatically by TypeScript. */

    /* Language and Environment */
    "target": "es2016",                                  /* Set the JavaScript language version for emitted JavaScript and include compatible library declarations. */
    // "lib": [],                                        /* Specify a set of bundled library declaration files that describe the target runtime environment. */
    // "jsx": "preserve",                                /* Specify what JSX code is generated. */
    // "experimentalDecorators": true,                   /* Enable experimental support for legacy experimental decorators. */
    // "emitDecoratorMetadata": true,                    /* Emit design-type metadata for decorated declarations in source files. */
    // "jsxFactory": "",                                 /* Specify the JSX factory function used when targeting React JSX emit, e.g. 'React.createElement' or 'h'. */
    // "jsxFragmentFactory": "",                         /* Specify the JSX Fragment reference used for fragments when targeting React JSX emit e.g. 'React.Fragment' or 'Fragment'. */
    // "jsxImportSource": "",                            /* Specify module specifier used to import the JSX factory functions when using 'jsx: react-jsx*'. */
    // "reactNamespace": "",                             /* Specify the object invoked for 'createElement'. This only applies when targeting 'react' JSX emit. */
    // "noLib": true,                                    /* Disable including any library files, including the default lib.d.ts. */
    // "useDefineForClassFields": true,                  /* Emit ECMAScript-standard-compliant class fields. */
    // "moduleDetection": "auto",                        /* Control what method is used to detect module-format JS files. */

    /* Modules */
    "module": "commonjs",                                /* Specify what module code is generated. */
    // "rootDir": "./",                                  /* Specify the root folder within your source files. */
    // "moduleResolution": "node10",                     /* Specify how TypeScript looks up a file from a given module specifier. */
    // "baseUrl": "./",                                  /* Specify the base directory to resolve non-relative module names. */
    // "paths": {},                                      /* Specify a set of entries that re-map imports to additional lookup locations. */
    // "rootDirs": [],                                   /* Allow multiple folders to be treated as one when resolving modules. */
    // "typeRoots": [],                                  /* Specify multiple folders that act like './node_modules/@types'. */
    // "types": [],                                      /* Specify type package names to be included without being referenced in a source file. */
    // "allowUmdGlobalAccess": true,                     /* Allow accessing UMD globals from modules. */
    // "moduleSuffixes": [],                             /* List of file name suffixes to search when resolving a module. */
    // "allowImportingTsExtensions": true,               /* Allow imports to include TypeScript file extensions. Requires '--moduleResolution bundler' and either '--noEmit' or '--emitDeclarationOnly' to be set. */
    // "resolvePackageJsonExports": true,                /* Use the package.json 'exports' field when resolving package imports. */
    // "resolvePackageJsonImports": true,                /* Use the package.json 'imports' field when resolving imports. */
    // "customConditions": [],                           /* Conditions to set in addition to the resolver-specific defaults when resolving imports. */
    // "resolveJsonModule": true,                        /* Enable importing .json files. */
    // "allowArbitraryExtensions": true,                 /* Enable importing files with any extension, provided a declaration file is present. */
    // "noResolve": true,                                /* Disallow 'import's, 'require's or '<reference>'s from expanding the number of files TypeScript should add to a project. */

    /* JavaScript Support */
    // "allowJs": true,                                  /* Allow JavaScript files to be a part of your program. Use the 'checkJS' option to get errors from these files. */
    // "checkJs": true,                                  /* Enable error reporting in type-checked JavaScript files. */
    // "maxNodeModuleJsDepth": 1,                        /* Specify the maximum folder depth used for checking JavaScript files from 'node_modules'. Only applicable with 'allowJs'. */

    /* Emit */
    // "declaration": true,                              /* Generate .d.ts files from TypeScript and JavaScript files in your project. */
    // "declarationMap": true,                           /* Create sourcemaps for d.ts files. */
    // "emitDeclarationOnly": true,                      /* Only output d.ts files and not JavaScript files. */
    // "sourceMap": true,                                /* Create source map files for emitted JavaScript files. */
    // "inlineSourceMap": true,                          /* Include sourcemap files inside the emitted JavaScript. */
    // "outFile": "./",                                  /* Specify a file that bundles all outputs into one JavaScript file. If 'declaration' is true, also designates a file that bundles all .d.ts output. */
    "outDir": "./dist",                                  /* Specify an output folder for all emitted files. */
    // "removeComments": true,                           /* Disable emitting comments. */
    // "noEmit": true,                                   /* Disable emitting files from a compilation. */
    // "importHelpers": true,                            /* Allow importing helper functions from tslib once per project, instead of including them per-file. */
    // "downlevelIteration": true,                       /* Emit more compliant, but verbose and less performant JavaScript for iteration. */
    // "sourceRoot": "",                                 /* Specify the root path for debuggers to find the reference source code. */
    // "mapRoot": "",                                    /* Specify the location where debugger should locate map files instead of generated locations. */
    // "inlineSources": true,                            /* Include source code in the sourcemaps inside the emitted JavaScript. */
    // "emitBOM": true,                                  /* Emit a UTF-8 Byte Order Mark (BOM) in the beginning of output files. */
    // "newLine": "crlf",                                /* Set the newline character for emitting files. */
    // "stripInternal": true,                            /* Disable emitting declarations that have '@internal' in their JSDoc comments. */
    // "noEmitHelpers": true,                            /* Disable generating custom helper functions like '__extends' in compiled output. */
    // "noEmitOnError": true,                            /* Disable emitting files if any type checking errors are reported. */
    // "preserveConstEnums": true,                       /* Disable erasing 'const enum' declarations in generated code. */
    // "declarationDir": "./",                           /* Specify the output directory for generated declaration files. */

    /* Interop Constraints */
    // "isolatedModules": true,                          /* Ensure that each file can be safely transpiled without relying on other imports. */
    // "verbatimModuleSyntax": true,                     /* Do not transform or elide any imports or exports not marked as type-only, ensuring they are written in the output file's format based on the 'module' setting. */
    // "isolatedDeclarations": true,                     /* Require sufficient annotation on exports so other tools can trivially generate declaration files. */
    // "allowSyntheticDefaultImports": true,             /* Allow 'import x from y' when a module doesn't have a default export. */
    "esModuleInterop": true,                             /* Emit additional JavaScript to ease support for importing CommonJS modules. This enables 'allowSyntheticDefaultImports' for type compatibility. */
    // "preserveSymlinks": true,                         /* Disable resolving symlinks to their realpath. This correlates to the same flag in node. */
    "forceConsistentCasingInFileNames": true,            /* Ensure that casing is correct in imports. */

    /* Type Checking */
    "strict": true,                                      /* Enable all strict type-checking options. */
    // "noImplicitAny": true,                            /* Enable error reporting for expressions and declarations with an implied 'any' type. */
    // "strictNullChecks": true,                         /* When type checking, take into account 'null' and 'undefined'. */
    // "strictFunctionTypes": true,                      /* When assigning functions, check to ensure parameters and the return values are subtype-compatible. */
    // "strictBindCallApply": true,                      /* Check that the arguments for 'bind', 'call', and 'apply' methods match the original function. */
    // "strictPropertyInitialization": true,             /* Check for class properties that are declared but not set in the constructor. */
    // "noImplicitThis": true,                           /* Enable error reporting when 'this' is given the type 'any'. */
    // "useUnknownInCatchVariables": true,               /* Default catch clause variables as 'unknown' instead of 'any'. */
    // "alwaysStrict": true,                             /* Ensure 'use strict' is always emitted. */
    // "noUnusedLocals": true,                           /* Enable error reporting when local variables aren't read. */
    // "noUnusedParameters": true,                       /* Raise an error when a function parameter isn't read. */
    // "exactOptionalPropertyTypes": true,               /* Interpret optional property types as written, rather than adding 'undefined'. */
    // "noImplicitReturns": true,                        /* Enable error reporting for codepaths that do not explicitly return in a function. */
    // "noFallthroughCasesInSwitch": true,               /* Enable error reporting for fallthrough cases in switch statements. */
    // "noUncheckedIndexedAccess": true,                 /* Add 'undefined' to a type when accessed using an index. */
    // "noImplicitOverride": true,                       /* Ensure overriding members in derived classes are marked with an override modifier. */
    // "noPropertyAccessFromIndexSignature": true,       /* Enforces using indexed accessors for keys declared using an indexed type. */
    // "allowUnusedLabels": true,                        /* Disable error reporting for unused labels. */
    // "allowUnreachableCode": true,                     /* Disable error reporting for unreachable code. */

    /* Completeness */
    // "skipDefaultLibCheck": true,                      /* Skip type checking .d.ts files that are included with TypeScript. */
    "skipLibCheck": true                                 /* Skip type checking all .d.ts files. */
  }
}

...

# Package.json

{
  "scripts": {
    "start": "node dist/index.js",
    "dev": "nodemon src/index.ts",
    "build": "tsc -p ."
  },
  "name": "2024-tri2-ia22-autenticacao-2022313856",
  "description": "- Autenticação é o ato de identificar um usuário ou dispositivo, enquanto autorização é o ato de permitir ou negar direitos de acesso a usuários e dispositivos.\r A autenticação pode ser usada como um fator nas decisões de autorização.\r Artefatos de autorização não são úteis para identificar usuários ou dispositivos.",
  "version": "1.0.0",
  "main": "main.js",
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.19.2",
    "sqlite": "^5.1.1",
    "sqlite3": "^5.1.7"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "nodemon": "^3.1.4",
    "ts-node": "^10.9.2",
    "typescript": "^5.5.4"
  }
}
