# Building a Website-Aware AI Chatbot with n8n

This guide walks you through creating an AI chatbot in **n8n** that can "read" a specific website and answer questions based on its content.

This architecture is known as **RAG (Retrieval-Augmented Generation)**. It involves two main workflows:
1.  **Ingestion Workflow**: Scrapes the website, splits the text, and stores it in a database (Vector Store).
2.  **Chat Workflow**: An AI Agent that searches the database for relevant info before answering a user's question.

---

## 🏗️ Architecture

```mermaid
graph LR
    subgraph "1. Ingestion Workflow"
    Schedule[Schedule Trigger] --> A[Manual Trigger]
    A --> B[Website Content Crawler]
    B --> C[Text Splitter]
    C --> D[Embeddings (OpenAI)]
    D --> E[Vector Store (Upsert)]
    end

    subgraph "2. Chat Workflow"
    F[Chat Trigger] --> G[AI Agent Node]
    G <--> H[OpenAI Model]
    G <--> I[Vector Store Tool]
    I -.-> E
    end
```

---

## ✅ Prerequisites

1.  **n8n Instance**: Self-hosted or n8n Cloud (version 1.0+ recommended for AI nodes).
2.  **OpenAI API Key**: For generating embeddings and powering the LLM.
3.  **Vector Store**: A database to store the website knowledge.
    *   *Recommendation*: **Pinecone** (easy setup, generous free tier) or **Supabase** (Postgres with pgvector).

---

## 🛠️ Step 1: The Ingestion Workflow

This workflow is run **once** (or periodically) to "teach" the AI about your website.

1.  **Create a new Workflow** in n8n.
2.  **Add a "Manual Trigger"** node.
3.  **Add a "Website Content Crawler"** node (or *Firecrawl* / *Crawl4AI* if available).
    *   **URL**: Enter the website URL you want to scrape.
    *   **Return All**: Toggle this on if you want to crawl sub-pages (be careful with large sites). or keep it off to just scrape the single page.
    *   **Output**: Select "Text" or "Markdown".
4.  **Add a "Recursive Character Text Splitter"** node.
    *   Connect it to the output of the crawler.
    *   **Chunk Size**: `1000` (adjustable).
    *   **Chunk Overlap**: `100`.
    *   *Note*: This node isn't a standalone step in the main flow but is often an input to the Vector Store node in newer n8n versions, or a processing step. In the modern **Vector Store** node, you often connect the splitter to the "Text Splitter" input of the store node.
5.  **Add a "Vector Store"** node (e.g., Pinecone Vector Store).
    *   **Mode**: `Insert Documents` (or Upsert).
    *   **Embeddings**: Connect an **OpenAI Embeddings** node to the "Embeddings" input.
    *   **Text Splitter**: Connect the **Recursive Character Text Splitter** node to the "Text Splitter" input.
    *   **Document**: Select the `text` or `content` field from the previous node.

**Run this workflow**. You should see success messages indicating documents were added to your Vector Store index.

### 🔥 Alternative: Using Firecrawl (Recommended)

If you have installed the **Firecrawl** community node, it is often better than the standard crawler because it converts websites into perfectly formatted Markdown specifically for AI.

1.  **Replace** the "Website Content Crawler" with the **Firecrawl** node.
2.  **Operation**: `Crawl`.
3.  **Credential**: Create a "Firecrawl API" credential using the key from [firecrawl.dev](https://firecrawl.dev).
4.  **Include Sub-pages**: Set a limit (e.g., 10 or 20) so you don't scrape thousands of pages accidentally.
5.  **Clean Data**: Firecrawl automatically removes HTML tags, navbars, and footers, which drastically improves the AI's accuracy.

---

## 💬 Step 2: The Chat Workflow

This workflow handles the actual conversation.

1.  **Create a new Workflow**.
2.  **Add a "Chat Trigger"** node (or *On Webhook* for custom frontends).
3.  **Add an "AI Agent"** node.
    *   **Mode**: `Chat` or `Auto-GPT`. "Chat" is usually faster.
    *   **Model**: Connect an **OpenAI Chat Model** node (choose `gpt-4o` or `gpt-3.5-turbo`).
4.  **Add "Window Buffer Memory"**.
    *   **Session Management**: n8n uses a `Session ID` to keep different users' conversations separate. Ensure the "Session ID" field in the node is set to the default: `{{ $json.sessionId }}`.
    *   **How it works**: This ensures that if ten people use your website chat at once, each person has their own private "memory" bank, while all sharing the same Vector Store data.
    *   Connect this to the "Memory" input of the AI Agent.
5.  **Add a "Vector Store Tool"**.
    *   Connect this to the "Tools" input of the AI Agent.
    *   **Vector Store**: Select the SAME Vector Store node type(e.g., Pinecone) you used in Step 1.
    *   **Operation**: `Retrieve` (or Get Many).
    *   **Embeddings**: Connect the same **OpenAI Embeddings** node.
    *   **System Prompt**: In the AI Agent configuration, add a system message:
        > "You are a helpful assistant. You must answer the user's questions using ONLY the context provided by the Vector Store Tool. If the answer is not in the context, say you don't know."

---

## 🌐 Step 3: Adding the Chat Box to Your Website

Once your workflow is ready, you need to embed the chat interface on your site so users can interact with it.

### 1. Enable CORS in n8n
To allow your website to talk to your n8n instance, you may need to set an environment variable in your n8n setup (especially if self-hosting):
`N8N_CORS_ALLOWED_ORIGINS=https://yourwebsite.com`

### 2. Embed the Chatbot
n8n provides a built-in chat interface that you can embed using a simple `<script>` tag.

1.  In your **Chat Workflow**, click on the **Chat Trigger** node.
2.  Go to the **"Embed"** tab.
3.  Copy the code snippet provided. It will look something like this:

```html
<script src="https://<your-n8n-instance>/chat.js"></script>
<script>
  createChat({
    webhookUrl: 'https://<your-n8n-instance>/webhook/chat',
    title: 'AI Assistant',
    welcomeMessage: 'How can I help you today?',
  });
</script>
```

4.  Paste this code into the `<body>` of your website's HTML.
5.  **Session Persistence**: The `createChat` script automatically stores the user's session ID in their browser. If a user refreshes the page, their conversation history will remain intact.

---

## ⏰ Step 4: Automating Daily Content Updates

To ensure your chatbot always has the latest information from your website, you can automate the scraping process.

1.  Open your **Ingestion Workflow**.
2.  Add a **Schedule Trigger** node.
3.  Set the **Interval** to `Every Day`.
4.  Set the **Time of Day** (e.g., `03:00` AM) when traffic is low.
5.  **Important: Prevent Duplicates**
    *   In your **Vector Store** node, ensure the mode is set to **Upsert** (if supported by your provider like Pinecone or Supabase).
    *   Use the **URL** or a **Hash** of the content as the unique ID so n8n updates existing records instead of creating new ones every day.

---

## 🧪 How to Test

1.  Open the **Chat** interface in n8n (part of the Chat Trigger / AI Agent preview).
2.  Ask a question that is specific to the website you scraped.
    *   *Example*: "What is the pricing on this website?" or "Who are the team members?"
3.  **Observation**:
    *   The Agent should pause to "use the tool" (query the vector store).
    *   It should retrieve the relevant chunk of text.
    *   It should generate a natural language answer based on that chunk.

## 🚀 Going Live: Activating Your Workflow

Before your website can talk to n8n, you must move the workflow from "Testing" to "Production."

1.  **Toggle the Active Switch**: In the top right of the n8n editor, toggle the workflow to **Active**.
2.  **Use the Production Webhook**: 
    *   n8n has two types of URLs: **Test** and **Production**.
    *   Once active, make sure your website snippet uses the **Production URL**.
3.  **Check Executions**: You can click the "Executions" tab in the left sidebar of n8n to see a history of every time a user has interacted with your bot.

## � Data & Privacy: Where is memory stored?

By default, n8n stores all chat history in its internal database.

*   **Self-Hosted**: Stored in your `database.sqlite` file.
*   **Separation**: History is indexed by `Session ID`.
*   **Retention**: You can configure n8n to automatically delete execution history after X days (in n8n Settings) to save disk space.

## 🚀 Pro Tips

*   **Filter Metadata**: If you scrape multiple sites, save the `url` as metadata. You can then configure the Vector Store Tool to filter answers by specific URLs.
*   **Human-in-the-loop**: For complex queries, you can add a step to notify a human via Slack or Discord if the AI is unsure of the answer.

---

## 💾 JSON Import Templates

To get started quickly, you can copy the JSON content from the blocks below and paste it directly into a new n8n workflow canvas.

### 1. Ingestion Workflow
<details>
<summary>Click to view JSON</summary>

```json
{
  "nodes": [
    { "parameters": {}, "id": "1", "name": "Manual Trigger", "type": "n8n-nodes-base.manualTrigger", "typeVersion": 1, "position": [0, 0] },
    { "parameters": { "rule": { "interval": [{ "field": "days" }] } }, "id": "2", "name": "Schedule Trigger", "type": "n8n-nodes-base.scheduleTrigger", "typeVersion": 1, "position": [0, 200] },
    { "parameters": { "url": "https://yourwebsite.com" }, "id": "3", "name": "Website Content Crawler", "type": "n8n-nodes-base.websiteContentCrawler", "typeVersion": 1, "position": [250, 100] },
    { "parameters": { "mode": "upsert" }, "id": "4", "name": "Vector Store", "type": "n8n-nodes-base.vectorStorePinecone", "typeVersion": 1, "position": [550, 100] },
    { "parameters": {}, "id": "5", "name": "OpenAI Embeddings", "type": "n8n-nodes-base.embeddingsOpenAi", "typeVersion": 1, "position": [450, 300] },
    { "parameters": {}, "id": "6", "name": "Recursive Character Text Splitter", "type": "n8n-nodes-base.textSplitterRecursiveCharacter", "typeVersion": 1, "position": [650, 300] }
  ],
  "connections": {
    "Manual Trigger": { "main": [[{ "node": "Website Content Crawler", "type": "main", "index": 0 }]] },
    "Schedule Trigger": { "main": [[{ "node": "Website Content Crawler", "type": "main", "index": 0 }]] },
    "Website Content Crawler": { "main": [[{ "node": "Vector Store", "type": "main", "index": 0 }]] },
    "OpenAI Embeddings": { "ai_embedding": [[{ "node": "Vector Store", "type": "ai_embedding", "index": 0 }]] },
    "Recursive Character Text Splitter": { "ai_textSplitter": [[{ "node": "Vector Store", "type": "ai_textSplitter", "index": 0 }]] }
  }
}
```
</details>

### 2. Chat Workflow
<details>
<summary>Click to view JSON</summary>

```json
{
  "nodes": [
    { "parameters": {}, "id": "1", "name": "Chat Trigger", "type": "n8n-nodes-base.chatTrigger", "typeVersion": 1, "position": [0, 0] },
    { "parameters": { "options": { "systemMessage": "You are a helpful assistant..." } }, "id": "2", "name": "AI Agent", "type": "n8n-nodes-base.aiAgent", "typeVersion": 1, "position": [250, 0] },
    { "parameters": { "model": "gpt-4o" }, "id": "3", "name": "OpenAI Chat Model", "type": "n8n-nodes-base.chatModelOpenAi", "typeVersion": 1, "position": [150, 200] },
    { "parameters": {}, "id": "4", "name": "Window Buffer Memory", "type": "n8n-nodes-base.memoryWindowBuffer", "typeVersion": 1, "position": [300, 200] },
    { "parameters": { "name": "website_knowledge" }, "id": "5", "name": "Vector Store Tool", "type": "n8n-nodes-base.vectorStoreTool", "typeVersion": 1, "position": [450, 200] }
  ],
  "connections": {
    "Chat Trigger": { "main": [[{ "node": "AI Agent", "type": "main", "index": 0 }]] },
    "OpenAI Chat Model": { "ai_languageModel": [[{ "node": "AI Agent", "type": "ai_languageModel", "index": 0 }]] },
    "Window Buffer Memory": { "ai_memory": [[{ "node": "AI Agent", "type": "ai_memory", "index": 0 }]] },
    "Vector Store Tool": { "ai_tool": [[{ "node": "AI Agent", "type": "ai_tool", "index": 0 }]] }
  }
}
```
</details>
