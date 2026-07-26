# 113 — Building a CLI tool

## What is this?

A CLI (Command Line Interface) tool is a program you run by typing a command in the terminal instead of clicking buttons in a browser — think `git commit`, `npm install`, or `npx create-react-app`. Node.js lets you build these tools with plain JavaScript: you read the arguments the user typed, do some work, and print results back to the terminal. Libraries like `commander.js` handle argument parsing, `chalk` adds color to your output, and `ora` shows a spinner while long tasks run — together they turn a raw script into a polished tool people can install with `npm install -g`.

## Why does it matter for backend development?

Backend developers write CLI tools constantly — database migration runners, project scaffolding generators, deployment scripts, log analyzers, seed-data generators. Every serious backend team has at least one internal CLI that new engineers run on day one (`npm run setup`, `mycli db:migrate`, `mycli generate:controller`). Knowing how to parse `process.argv` properly, give clear help text, show progress on slow operations, and publish the tool to npm (public or private registry) means you can turn a one-off script into something your whole team reuses instead of copy-pasting code. It is also how you package and distribute your own dev tools to the world.

---

## Syntax / API

```js
#!/usr/bin/env node
// The "shebang" line above tells Unix systems to run this file with node
// It is required for a CLI tool to be executable directly (e.g. ./mycli)

const { Command } = require('commander');   // commander.js — argument/flag parser
const chalk = require('chalk');              // chalk — colored terminal output
const ora = require('ora');                  // ora — animated loading spinners

const program = new Command();               // create the root CLI program

// Define metadata shown in --help and --version
program
  .name('mycli')                              // the command name users type
  .description('CLI tool for managing backend deployments')  // shown in --help
  .version('1.0.0');                          // shown when user runs --version

// Define a subcommand: mycli deploy <environment>
program
  .command('deploy <environment>')            // <environment> is a required positional arg
  .description('Deploy the app to the given environment')
  .option('-f, --force', 'skip confirmation prompt')  // optional boolean flag
  .option('-t, --timeout <seconds>', 'deploy timeout in seconds', '30') // flag with value + default
  .action(async (environment, options) => {
    // environment → the positional argument (e.g. "staging")
    // options     → object of parsed flags, e.g. { force: true, timeout: '30' }

    console.log(chalk.blue(`Deploying to ${environment}...`)); // blue informational text

    const spinner = ora('Connecting to server').start(); // start an animated spinner
    await new Promise((resolve) => setTimeout(resolve, 1500)); // simulate async work
    spinner.succeed('Connected');              // replace spinner with a green checkmark

    console.log(chalk.green.bold('Deploy finished successfully!')); // bold green success text
  });

program.parse(process.argv);   // parse the actual command line arguments and run the matching action
```

---

## How it works — line by line

The shebang line (`#!/usr/bin/env node`) is a special first line that only matters on Linux/Mac — it tells the operating system "run this file using whatever `node` is found on the PATH." Without it, users would have to type `node mycli.js` instead of just `mycli`.

`commander.js` gives you a `Command` object that represents your whole CLI program. You chain methods onto it: `.name()` and `.description()` set up the text shown in `--help`, and `.version()` wires up the `-V`/`--version` flag automatically — you never have to write that logic yourself.

`.command('deploy <environment>')` registers a subcommand. The `<environment>` in angle brackets means it is a **required** positional argument — commander will error out with a helpful message if the user forgets it. Square brackets like `[environment]` would make it optional instead.

`.option('-f, --force', 'description')` registers a flag. Commander automatically supports both the short form (`-f`) and long form (`--force`), and produces `options.force` as `true` when the user passes it. When an option needs a value, like `--timeout <seconds>`, commander captures whatever the user typed after it as a string.

`.action(async (environment, options) => {...})` is the function that actually runs when the user types this subcommand. Commander calls it with the positional arguments first, then an object holding all the parsed flags.

`chalk.blue(...)`, `chalk.green.bold(...)` wrap a string in terminal color codes — chalk handles the escape sequences so you just chain readable method names.

`ora('message').start()` prints a spinning animation next to your message while an async task runs. Calling `.succeed('message')` stops the spinner and replaces it with a green checkmark and final message; `.fail('message')` would show a red X instead.

Finally, `program.parse(process.argv)` is the line that reads the real command-line input the user typed and runs whichever subcommand and options matched.

---

## Example 1 — basic

```js
#!/usr/bin/env node
// A minimal CLI built with raw process.argv — no libraries yet, to show the fundamentals

// process.argv is an array: [nodePath, scriptPath, ...userArguments]
const args = process.argv.slice(2);   // drop the first two entries, keep only what the user typed

// Example run: node greet.js --name Riya --loud
const nameIndex = args.indexOf('--name');          // find the position of the --name flag
const userName  = nameIndex !== -1 ? args[nameIndex + 1] : 'stranger'; // read the value right after it
const isLoud    = args.includes('--loud');          // boolean flags are just checked with includes()

let message = `Hello, ${userName}!`;   // build the base greeting message

if (isLoud) {
  message = message.toUpperCase() + '!!!';   // shout it if --loud was passed
}

console.log(message);   // print the final result to the terminal
// Run:  node greet.js --name Riya --loud
// Out:  HELLO, RIYA!!!!
```

---

## Example 2 — real world backend use case

```js
#!/usr/bin/env node
// File: bin/mycli.js
// A real internal tool: generates a boilerplate Express controller file,
// the kind of script backend teams keep in every repo to save typing.

const { Command } = require('commander');
const chalk = require('chalk');
const ora = require('ora');
const fs = require('fs/promises');   // async fs API — covered in Topic 16
const path = require('path');

const program = new Command();

program
  .name('mycli')
  .description('Internal scaffolding CLI for backend services')
  .version('2.1.0');

program
  .command('generate:controller <resourceName>')   // e.g. mycli generate:controller user
  .description('Scaffold a new Express controller file')
  .option('-d, --dir <folder>', 'target folder', 'src/controllers') // default output folder
  .action(async (resourceName, options) => {
    const spinner = ora('Preparing controller file').start(); // show progress immediately

    try {
      const className   = resourceName[0].toUpperCase() + resourceName.slice(1); // e.g. "user" -> "User"
      const filePath     = path.join(process.cwd(), options.dir, `${resourceName}.controller.js`); // build target path from cwd, not __dirname — user runs this from their project root
      const fileContents =
`// Auto-generated controller for ${resourceName}
const dbConnection = require('../db');   // placeholder import for the real DB layer

async function get${className}ById(req, res) {
  const userId = req.params.id;          // route param, e.g. /users/:id
  const record = await dbConnection.query('SELECT * FROM ${resourceName}s WHERE id = $1', [userId]);
  res.json(record.rows[0]);              // respond with the found row as JSON
}

module.exports = { get${className}ById };
`;

      await fs.mkdir(path.dirname(filePath), { recursive: true }); // ensure target folder exists
      await fs.writeFile(filePath, fileContents, 'utf8');           // write the generated file to disk

      spinner.succeed(`Created ${chalk.cyan(filePath)}`);           // green checkmark + cyan file path
      console.log(chalk.green('Controller scaffold ready. Wire it up in your routes file.'));
    } catch (error) {
      spinner.fail('Failed to generate controller');   // red X on failure
      console.error(chalk.red(error.message));          // print the actual error in red
      process.exitCode = 1;                             // signal failure to the calling shell/CI
    }
  });

program.parse(process.argv);   // run the CLI with the real arguments passed by the user

// package.json addition to make this globally runnable:
// "bin": { "mycli": "./bin/mycli.js" }
// Then: npm link (local testing) or npm publish (real distribution)
```

---

## Common mistakes

### Mistake 1 — Forgetting to slice process.argv

```js
// ❌ WRONG — process.argv[0] and [1] are the node binary path and script path,
// not user input — this breaks all your index math
const args = process.argv;
const firstUserArg = args[0];   // this is actually "/usr/local/bin/node", not user input

// ✅ CORRECT — always slice off the first two entries first
const args = process.argv.slice(2);
const firstUserArg = args[0];   // now this is genuinely the first thing the user typed
```

### Mistake 2 — Blocking spinners with synchronous code

```js
// ❌ WRONG — ora's spinner needs the event loop free to animate;
// a synchronous blocking call freezes the spinner mid-frame
const spinner = ora('Reading large file').start();
const data = fs.readFileSync('./huge-dataset.json', 'utf8');   // blocks the whole process
spinner.succeed('Done');   // spinner never actually animated, just froze then jumped to done

// ✅ CORRECT — use the async fs API so the event loop stays free to render the spinner
const spinner = ora('Reading large file').start();
const data = await fs.promises.readFile('./huge-dataset.json', 'utf8'); // non-blocking
spinner.succeed('Done');   // spinner animates smoothly the whole time
```

### Mistake 3 — Missing the bin field or shebang when publishing to npm

```js
// ❌ WRONG — package.json has no "bin" field, so npm has no idea this package
// should install a global command — "mycli: command not found" after install
{
  "name": "mycli",
  "version": "1.0.0",
  "main": "index.js"
}

// ✅ CORRECT — declare the bin entry AND include the shebang in that file
// package.json:
{
  "name": "mycli",
  "version": "1.0.0",
  "main": "index.js",
  "bin": { "mycli": "./bin/mycli.js" }   // maps the "mycli" command to this file
}
// bin/mycli.js — first line MUST be the shebang, and the file must be executable:
// #!/usr/bin/env node
// chmod +x bin/mycli.js   (or set it via files/postinstall before publishing)
```

---

## Practice exercises

### Exercise 1 — easy

Write a plain Node.js script (no libraries) called `whoami.js` that reads `process.argv` directly and:
1. Accepts a `--name <value>` flag and a `--role <value>` flag
2. If `--name` is missing, defaults to `"Anonymous"`
3. If `--role` is missing, defaults to `"Guest"`
4. Prints a message like: `Anonymous logged in as Guest` or `Riya logged in as Admin`
5. Supports a `--help` flag that prints usage instructions and exits before doing anything else

```js
// Write your code here
```

---

### Exercise 2 — medium

Using `commander.js` and `chalk`, build a CLI tool `taskcli.js` with two subcommands:
1. `taskcli add <taskDescription>` — appends the task to a local `tasks.json` file (create it if missing), each task stored as `{ id, description, done: false }`
2. `taskcli list` — reads `tasks.json` and prints every task, showing completed tasks in green with a checkmark and pending tasks in yellow
3. Add a `--done` flag to `list` that filters to only show completed tasks
4. Handle the case where `tasks.json` does not exist yet gracefully (print a friendly message instead of crashing)

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a full CLI tool `dbseed.js` that simulates seeding a database, combining `commander`, `chalk`, and `ora`:
1. Command: `dbseed run <tableName>` with options `--rows <count>` (default `10`) and `--dry-run` (boolean, no writes)
2. Show an `ora` spinner titled `Generating {count} rows for {tableName}` while you simulate work with a `setTimeout`-based delay proportional to the row count
3. On success, print a summary table to the console (plain formatted text is fine) showing: table name, rows inserted, and time taken in milliseconds
4. If `--dry-run` is passed, skip the actual "insert" step, show the spinner as informational only, and print a yellow warning that no data was written
5. If `--rows` is not a valid positive number, fail fast with a red error message and set `process.exitCode = 1` before any spinner starts
6. Structure the file so it could realistically be published to npm — include the shebang line and note what `bin` entry would be needed in `package.json`

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
RAW process.argv
  process.argv[0]        → path to the node binary
  process.argv[1]         → path to the running script
  process.argv.slice(2)   → the actual arguments the user typed

COMMANDER.JS
  new Command()                          → create the CLI program
  .name() / .description() / .version()  → metadata shown in --help / --version
  .command('name <req> [opt]')           → subcommand, <required>, [optional]
  .option('-f, --flag <val>', 'desc', default) → registers a flag
  .action((args..., options) => {})      → handler run when subcommand matches
  program.parse(process.argv)            → parses input and triggers the action

CHALK (v5+ is ESM-only — use chalk@4 for CommonJS require())
  chalk.red() / .green() / .yellow() / .blue()   → colored text
  chalk.bold() / .underline() / .dim()            → text styles
  chalk.green.bold('text')                        → chainable combos

ORA
  ora('message').start()      → begin spinner
  spinner.succeed('msg')      → green checkmark, stop spinner
  spinner.fail('msg')         → red X, stop spinner
  spinner.text = 'new msg'    → update the running spinner's text
  spinner.stop()              → stop with no icon

PUBLISHING TO NPM
  package.json "bin" field    → { "mycli": "./bin/mycli.js" }
  shebang line required       → "#!/usr/bin/env node" as the very first line
  chmod +x bin/mycli.js       → make the file executable (or use files/prepare script)
  npm link                    → test the global command locally before publishing
  npm login                   → authenticate with the npm registry
  npm publish                 → push the package live
  npm version patch|minor|major → bump semver before republishing

GOTCHAS
  Forgetting .slice(2) on raw process.argv → off-by-two index bugs
  Blocking sync calls freeze ora's animation → use async APIs
  chalk v5+ requires ESM import, not require() → pin chalk@4 for CommonJS projects
  Missing shebang or bin field → "command not found" after global install
```

---

## Connected topics

- **05 — The process object** — `process.argv` is the raw input every CLI parser (including commander) reads under the hood
- **114 — Building npm packages** — the packaging, `exports` field, and publishing workflow you use to ship this CLI tool to the npm registry
- **31 — readline module** — the built-in way to build interactive prompts (yes/no confirmations, text input) inside a CLI tool
