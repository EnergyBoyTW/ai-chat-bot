# Website Chat Assistant

A Chrome extension that adds an AI chat bubble to any website and answers questions using that site's own pages. You scrape a page or crawl the whole site, the text is embedded and stored in Postgres, and each answer is grounded in the most relevant chunks from the site you are on.

The extension talks to a small Hono server that runs locally. The server holds your OpenAI key, so it never touches the browser.

## How it works

1. The content script injects a chat bubble and panel into every page.
2. Scraping reads the current page, or crawls same-origin links (up to 100 pages), strips scripts and styles, and splits the text into chunks of about 8,000 characters.
3. The server embeds each chunk with `text-embedding-3-large` and stores it in a Neon Postgres table (`scraped_content`) through LangChain's Neon vector store.
4. When you ask something, the server finds the 4 closest chunks for the current domain, adds the last few turns of that site's chat history, and streams a reply from OpenAI (`gpt-4o` by default).
5. Chat history is saved per site in a `messages` table and reloads when you open the bubble again.

## Features

- Chat bubble on every site, with streaming replies
- Scrape the current page, crawl the whole site, or crawl only pages that match your URL patterns (for example `*/docs/*`)
- Answers only use content from the site you are on
- Per-site chat history that the model can see, so follow-up questions work
- Nine bubble positions, saved with `chrome.storage.sync`

## Project structure

```
ai-chat-bot/
├── extension/              Chrome extension (Manifest V3)
│   ├── manifest.json
│   ├── content.js          chat UI, scraping and crawling
│   ├── popup.html          settings popup
│   ├── popup.js            position, scrape buttons, URL patterns
│   └── icons/
└── server/                 Hono API on Node
    ├── src/
    │   ├── index.ts        routes: chat, store, chat history
    │   ├── lib/vectorStore.ts
    │   └── db/             Drizzle schema and queries for chat history
    ├── drizzle/0000_messages.sql
    └── .env.example
```

## Setup

You need Node.js 20 or later, an OpenAI API key and a Neon Postgres database.

### Server

```bash
cd server
npm install
cp .env.example .env    # then fill in OPENAI_API_KEY and POSTGRES_URL
```

Create the chat history table once by running `drizzle/0000_messages.sql` in the Neon SQL editor (or with `psql "$POSTGRES_URL" -f drizzle/0000_messages.sql`). The vector table is created automatically on first use.

```bash
npm run dev             # http://localhost:8080
```

| Variable | Required | Purpose |
| --- | --- | --- |
| `OPENAI_API_KEY` | yes | Chat completions and embeddings |
| `POSTGRES_URL` | yes | Neon connection string for vectors and chat history |
| `OPENAI_MODEL` | no | Chat model, defaults to `gpt-4o` |
| `PORT` | no | Defaults to `8080` |

Other scripts: `npm start` runs the server without watch mode, and `npm run typecheck` runs the TypeScript compiler.

### Extension

1. Open `chrome://extensions` and turn on Developer mode.
2. Click Load unpacked and pick the `extension` folder.
3. Keep the server running on port 8080. The extension calls `http://localhost:8080`.

## Usage

1. Open any website and click the chat bubble.
2. Click Scrape Site in the chat header to crawl the site, or open the extension popup to scrape just the current page or only pages matching your URL patterns.
3. Ask questions about the site.

## API

| Method | Path | Body or params | Description |
| --- | --- | --- | --- |
| GET | `/` | | Health check |
| POST | `/store` | `{ documents: [{ url, domain, content }] }` | Embed and store page chunks |
| POST | `/` | `{ domain, messages: [{ role, content }] }` | Streamed chat reply as plain text |
| GET | `/chat-history/:domain` | | Saved messages for a site |

## License

MIT, see [LICENSE](LICENSE).
