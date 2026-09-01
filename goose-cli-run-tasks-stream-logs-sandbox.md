# Run Goose CLI Headlessly in Daytona and Stream Its Output

This guide demonstrates how to run [Goose](https://block.github.io/goose/) (Block's open source AI agent CLI) as a headless coding agent inside a Daytona sandbox. The agent can write code in any language, install dependencies, run scripts, and reason over a project, all inside a secure, isolated, disposable sandbox while its output streams back to your terminal in real time.

---

### 1. Workflow Overview

When you launch the main module, a Daytona sandbox is created and the Goose CLI is installed inside it with a fixed provider and model, so it never falls back to its interactive setup wizard. The agent is driven headlessly with `goose run --output-format stream-json --text "<prompt>"`, and its newline-delimited JSON events are parsed and printed as the agent works.

You interact with the main program via a command line chat interface. The program sends your prompts to the Goose CLI inside the sandbox, which writes code, runs commands, and streams the results back as it works. Each tool call surfaces as a `[tool]` line, followed by the assistant's response.

Goose has no dedicated session-init event, so instead of tracking a session ID, each turn after the first simply adds `--resume`, and Goose continues its most recent session, keeping context across the conversation. If the agent needs to start a server, it writes the start command to a script instead of running it directly, and the script runs it in the background so a preview URL stays available while the conversation continues. You can continue interacting with your agent until you are finished. When you exit the program, the sandbox is deleted automatically.

### 2. Project Setup

#### Clone the Repository

First, clone the daytona [repository](https://github.com/daytona/guides.git) and navigate to the example directory:

```bash
git clone https://github.com/daytona/guides.git
cd guides/typescript/goose/goose-cli
```

#### Configure Environment

Get your API keys:

- **Daytona API key:** [Daytona Dashboard](https://app.daytona.io/dashboard/keys)
- **OpenAI API key:** [OpenAI Platform](https://platform.openai.com/api-keys)

Copy `.env.example` to `.env` and add your keys:

```bash
DAYTONA_API_KEY=your_daytona_key
SANDBOX_OPENAI_API_KEY=your_openai_key
```

:::caution[API Key Security]
The `SANDBOX_OPENAI_API_KEY` key is passed into the Daytona sandbox environment and is accessible to any code executed inside the sandbox.
:::

#### Alternative: Inject the Key as a Daytona Secret

The default setup passes the OpenAI key into the sandbox as a plain environment variable, so anything running inside the sandbox - including the agent itself - can read the raw key with `env`. [Daytona Secrets](https://www.daytona.io/docs/en/secrets.md) keep the raw value out of the sandbox entirely: the environment variable holds only an opaque placeholder (`dtn_secret_<id>`), and Daytona's outbound proxy substitutes the real value into HTTPS request headers at egress - and only for requests to the hosts the Secret allows. An agent that dumps the environment or exfiltrates it never sees a usable key.

The Secret-based flow needs `@daytona/sdk` 0.192.0 or newer and a one-time Secret setup:

1. Create the Secret once for your organization - in the [Daytona Dashboard](https://app.daytona.io/dashboard/secrets) or with a one-off script (save as `create-secret.ts` next to this guide's `.env` and run `npx tsx create-secret.ts`):

   ```typescript
   import { Daytona } from '@daytona/sdk'
   import * as dotenv from 'dotenv'

   dotenv.config()

   async function main() {
     const value = process.env.SANDBOX_OPENAI_API_KEY
     if (!value) throw new Error('SANDBOX_OPENAI_API_KEY is not set')

     const daytona = new Daytona()
     await daytona.secret.create({
       name: 'openai-api-key',
       value,
       hosts: ['api.openai.com'], // the only host the real key may be sent to
     })
   }

   main()
   ```

2. In `src/index.ts`, swap the `OPENAI_API_KEY` env var for a `secrets:` mapping (environment variable name to Secret name). The other `GOOSE_*` variables carry no credentials and stay plain env vars:

   ```diff
    sandbox = await daytona.create({
      envVars: {
   -    OPENAI_API_KEY: process.env.SANDBOX_OPENAI_API_KEY,
        GOOSE_PROVIDER: 'openai',
        GOOSE_MODEL: 'gpt-4o',
        GOOSE_MODE: 'auto',
        GOOSE_DISABLE_KEYRING: '1',
      },
   +  secrets: {
   +    OPENAI_API_KEY: 'openai-api-key',
   +  },
    })
   ```

Inside the sandbox, `env` now shows `OPENAI_API_KEY=dtn_secret_...`, yet Goose still authenticates: it sends the key as the `Authorization` HTTPS request header to `api.openai.com`, where the proxy swaps in the real value. Substitution happens only in HTTPS request headers toward allowed hosts - requests to any other host carry the harmless placeholder. See the [Secrets documentation](https://www.daytona.io/docs/en/secrets.md#substitution-scope) for the full substitution scope.

#### Local Usage

:::note[Node.js Version]
Node.js 20 or newer is required to run this example. Please ensure your environment meets this requirement before proceeding.
:::

Install dependencies:

```bash
npm install
```

Run the agent:

```bash
npm run start
```

The agent will start and wait for your prompt.

### 3. Example Usage

Ask the agent to write and run some code. Here it generates a static ASCII-art 3D projection of a torus inside the sandbox and executes it, streaming the assistant's reasoning and the program output back to your terminal:

````
$ npm run start
Creating sandbox...
Installing Goose CLI...
Starting Goose CLI...

Agent ready. Press Ctrl+C at any time to exit.

User: Make a script that prints a static 3d projection of a torus in ASCII. Do not animate it. Just make the script (without printing the code) and print the result.
Let's begin by creating a Python script that generates a static ASCII representation of a 3D torus. I'll run the script to show you the result.

### Steps:
1. **Calculate the ASCII Art for Torus Projection**: We will use trigonometric functions to project a torus into 3D space.
2. **Implement the ASCII Art in a Python Script**: Write the script that translates mathematical calculations into ASCII characters.
3. **Run the Script to Display the Result**: Execute the script to generate and display the ASCII torus.

Let's write and execute the script.It seems there's an `IndexError` in the script due to the calculations exceeding the list indices for the size defined. Let's fix the index boundaries and adjust the parameters to avoid this issue.

I'll correct the script and try running it again.Here is the static ASCII representation of a 3D torus:

```

                     ~  ~  ~ ~  ~  -  -
                : :; :; :  : ~  ~  ~ -- -- -
             ;  ;  ;  =  ; ; ;  : ~  -  -  -  -
           ;  == !  !  = = = ;  ; : ~  -  - ,,  -
         ;  =! !* ** * !* != =  ;; :~ ~ -- ,, ,,  -
       =  =!**#  #  # * * * != = ; ; ~ ~  -  ,,,,,  -
        !*#  #  #$#$ # # #* *! = =; ; ~ ~--,  ,  .,,
     ;=!*  $ $$$  $ $## ####** *!!== ;:: -  ,,. .  ,,-~
    ;*#  #$@  @@$ $$$$$#####*# ****!=;;:~~ -,,  ...  ,,~
   =*   $$@  @@  @$$###              !==:~-  ,.  ...   ,~
     # $@ @ @@ @@$$$*                  =::~-, ,. . .. ,
  =!*#$ @@ @ @@ $$#!                    !:~- ,. . .. .,-~:
  *#$ @@ @@@@ @$#*!                      ;:~-, .... .. ,-~
            @ @$#*=                      =~-,. .
  *#$ @@ @@ @ @$#*;                      ;~-,... .. . .,-~
  *#$ @@ @@@@ $#*!=                      :~-,. .... .. ,-~
  !*#$@ @@ @ @$ #*!:                    :-,, .. . .. ,-~:;
     $ @@ @ @$ $ *!!;                  ~,,, . .. . ., -
   *$   @@@  @$ ##*!;;:              ---,,.. ..  .,,   ~;
    *$$   @@  $$# **!==;:~~~~  ~-----,,,,, ,..  .,   -~=
     *#$$@ @ @$$  # ! ! ;;::~ ~~---- --, ,  ,,, , -~:;=
        $$@  $  $##* * = =; ; ~~ ~- - - ,-,-  -  -~:
       #  $$$$$  #  * * = = ; ;: ~ ~ ~ -  -  -~~:;  ;
         #  $$ $$ ## * *! ==  ; ;: ~: ~ ~~ ~: :;  =
           #  $$ ## #  * ! =  = ; ; ;  :  : ;;  =
             #  #  #  #  * !  = = =  ;  =  =  =
                # ## ## *  *  ! !  ! =! =! !
                     #  #  *  * *  *  *

```

If there's anything more you'd like to do, please let me know!
User:
Cleaning up...
````

:::note[Why the extra step?]
The first version of the script raised an `IndexError` because the projection math walked past the bounds of the character grid. Goose noticed the failure from the tool's exit output, corrected the index boundaries itself, and reran the script before returning the final torus - no extra prompting was required.
:::

### 4. Understanding the Script

This example consists of two parts: a main program (`src/index.ts`) that manages the sandbox, the prompt loop, and background servers started by the agent, and a session class (`src/session.ts`) that drives the Goose CLI over a PTY and parses its streaming JSON output.

#### Initialization

On startup, the script:

1. Creates a new [Daytona sandbox](https://www.daytona.io/docs/en/sandboxes.md) with the OpenAI key and a fixed Goose provider/model injected as environment variables.
2. Installs the Goose CLI in the sandbox using the official install script.
3. Builds a Daytona-aware system prompt containing the sandbox's preview URL pattern.
4. Creates a PTY, starts a Goose session, and sends the system prompt on the first turn.
5. Enters a readline loop to send prompts, receive streamed responses, and start any server script the agent wrote.
6. On Ctrl+C, kills the PTY session, deletes the sandbox (and any background server sessions), and exits.

#### Creating the Sandbox

Goose defaults to an interactive `goose configure` wizard the first time it runs, which would hang a headless run. Passing `OPENAI_API_KEY`, `GOOSE_PROVIDER`, and `GOOSE_MODEL` as sandbox environment variables at create time gives Goose a working provider without ever prompting. `GOOSE_MODE=auto` auto-approves tool calls so a turn never blocks on a confirmation prompt, and `GOOSE_DISABLE_KEYRING=1` makes Goose use file-based secret storage instead of a desktop keyring that doesn't exist in the sandbox:

```ts
sandbox = await daytona.create({
  envVars: {
    OPENAI_API_KEY: process.env.SANDBOX_OPENAI_API_KEY,
    GOOSE_PROVIDER: 'openai',
    GOOSE_MODEL: 'gpt-4o',
    GOOSE_MODE: 'auto',
    GOOSE_DISABLE_KEYRING: '1',
  },
})

const install = await activeSandbox.process.executeCommand(
  'curl -fsSL https://github.com/block/goose/releases/download/stable/download_cli.sh | CONFIGURE=false bash',
)
if (install.exitCode !== 0) {
  throw new Error('Error installing Goose CLI: ' + install.result)
}
```

`CONFIGURE=false` skips the installer's trailing `goose configure` step. Without it, the installer tries to run that step interactively - `[ -r /dev/tty ]` reports true in the sandbox's exec environment even though there is no real controlling terminal attached, so the read fails instead of falling back cleanly.

#### Preview URLs for Background Servers

`goose run` is a one-shot command that blocks until it exits, so a foreground dev server started inside a turn would never let that turn - or the whole prompt loop - complete. To work around this, a Daytona-aware system prompt tells Goose to write server-start commands to a script instead of running them directly:

```ts
const previewLink = await activeSandbox.getPreviewLink(1234)
const previewUrlPattern = previewLink.url.replace(/1234/, '{PORT}')
const defaultSystemPrompt = [
  'You are running in a Daytona sandbox.',
  `When running services on localhost, they will be accessible as: ${previewUrlPattern}`,
  'When you need to start a server, DO NOT run it directly.',
  'Instead, write only the server start command to /home/daytona/start.sh (one command, no markdown).',
  'After writing the start command, provide the preview URL to the user.',
].join(' ')
```

After every turn, the script checks whether `/home/daytona/start.sh` exists and, if so, runs it in a separate Daytona session with `runAsync: true` - outside Goose entirely - so the server keeps running in the background while the conversation continues:

```ts
const startServerFromScript = async () => {
  const startScriptCheck = await activeSandbox.process.executeCommand('test -f /home/daytona/start.sh')
  if (startScriptCheck.exitCode !== 0) {
    return
  }

  const sessionId = `goose-server-session-${Date.now()}`
  await activeSandbox.process.createSession(sessionId)
  serverSessions.push(sessionId)

  await activeSandbox.process.executeSessionCommand(sessionId, {
    command: 'cd /home/daytona && chmod +x start.sh && ./start.sh',
    runAsync: true,
  })
}
```

Those server sessions are cleaned up alongside the sandbox on exit.

#### PTY Communication

The session uses a pseudo-terminal (PTY) for streaming output from the Goose CLI. Goose installs to `~/.local/bin`, which a fresh shell may not have on `PATH`, so it's exported once right after the PTY connects, since this PTY (and its shell) is reused for every turn:

```ts
async initialize(options?: { systemPrompt?: string }): Promise<void> {
  this.ptyHandle = await this.sandbox.process.createPty({
    id: `goose-pty-${Date.now()}`,
    cols: 120,
    rows: 30,
    onData: (data: Uint8Array) => this.handleData(data),
  })
  await this.ptyHandle.waitForConnection()
  await this.ptyHandle.sendInput('export PATH="$HOME/.local/bin:$PATH"\n')

  if (options?.systemPrompt?.trim()) {
    this.systemPrompt = options.systemPrompt.trim()
  }
}
```

#### Running Goose Commands

Each prompt is sent as a `goose run` command in headless mode. `--text` supplies a one-shot non-interactive prompt and `--output-format stream-json` emits newline-delimited JSON events. Goose has no dedicated session-init event and no session ID to capture, so instead of resuming a specific ID, `--resume` simply continues Goose's most recent session once the first turn has completed. The system prompt is only sent via `--system` on that first turn - Goose carries it forward as part of the session `--resume` continues:

```ts
async processPrompt(prompt: string): Promise<void> {
  const flags = ['run', '--output-format', 'stream-json', '--text', this.shellQuote(prompt)]
  if (this.resumable) {
    flags.push('--resume')
  } else if (this.systemPrompt) {
    flags.push('--system', this.shellQuote(this.systemPrompt))
  }
  const command = ['goose', ...flags].join(' ')

  await this.ptyHandle!.sendInput(`cd ${WORK_DIR} && ${command}\n`)
  await new Promise<void>((resolve) => {
    this.onResponseComplete = resolve
  })
}
```

#### Streaming JSON Messages

The Goose CLI outputs JSON lines that are parsed to track agent activity. The `handleData` method buffers incoming PTY bytes and processes each complete line, keeping any incomplete line for the next chunk. A stateful `TextDecoder` is reused across calls so partial multi-byte UTF-8 sequences split across PTY chunks are preserved instead of being corrupted:

```ts
private decoder = new TextDecoder('utf-8')

private handleData(data: Uint8Array): void {
  this.buffer += this.decoder.decode(data, { stream: true })
  const lines = this.buffer.split('\n')
  this.buffer = lines.pop() || ''
  for (const line of lines.map((l) => l.trim()).filter(Boolean)) {
    try {
      this.handleEvent(JSON.parse(line) as GooseEvent)
    } catch {
      debug('non-JSON line:', line)
    }
  }
}
```

Event types from the Goose CLI's streaming JSON:

- **message**: A turn's content blocks - `text` (assistant text, printed live), `tool_use` (printed as `[tool] <name>`), and `tool_result` (printed as `[tool error] ...` when `is_error` is set)
- **complete**: Marks the end of a turn - signals response completion, but does not by itself mean the turn succeeded
- **error**: A hard failure reported directly by the CLI

Goose has a quirk where provider/model failures (bad key, quota, etc.) aren't reported as an `error` event - they show up as ordinary assistant text starting with `"Ran into this error: ..."`, followed by a normal `complete` event. A regex matches that wrapped text so the `complete` handler can surface it as a real failure instead of printing it as if it were a reply:

```ts
private handleEvent(event: GooseEvent): void {
  switch (event.type) {
    case 'message': {
      const msg = (event as GooseMessageEvent).message
      if (msg.role !== 'assistant') return

      for (const block of msg.content) {
        if (block.type === 'text') {
          const errMatch = block.text.match(GOOSE_ERROR_TEXT)
          if (errMatch) {
            this.pendingError = errMatch[1].trim()
            continue
          }
          process.stdout.write(block.text)
        } else if (block.type === 'tool_use') {
          process.stdout.write(`\n[tool] ${block.name}\n`)
        } else if (block.type === 'tool_result' && block.is_error) {
          const content = block.content
          const output =
            typeof content === 'string'
              ? content
              : Array.isArray(content)
                ? content
                    .filter((part) => part.type === 'text' && part.text)
                    .map((part) => part.text)
                    .join('\n')
                : ''
          process.stdout.write(`\n[tool error] ${output}\n`)
        }
      }
      return
    }
    case 'complete': {
      this.resumable = true
      if (this.pendingError) {
        process.stderr.write(`\nFailed: ${this.pendingError}\n`)
        this.pendingError = null
      }
      process.stdout.write('\n')
      this.onResponseComplete?.()
      return
    }
    case 'error': {
      const err = (event as GooseErrorEvent).error
      const message = typeof err === 'string' ? err : err.message
      process.stderr.write(`\nFailed: ${message}\n`)
      this.onResponseComplete?.()
      return
    }
  }
}
```

When the `complete` event arrives, `onResponseComplete` resolves the promise that `processPrompt` is awaiting, so the readline loop can check for a server start script and then prompt for the next turn.

**Key advantages:**

- Secure, isolated execution in Daytona sandboxes
- Fully headless operation, no setup wizard and no permission prompts
- Streaming JSON output (`--output-format stream-json`) for real-time tool and message activity
- PTY-based communication for low-latency streaming
- Session-based conversation continuity across prompts (`--resume`)
- Preview URLs for servers the agent starts, run outside the blocking `goose run` turn
- All agent code execution happens inside the sandbox
- Automatic cleanup on exit
