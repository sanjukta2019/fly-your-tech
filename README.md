# Fly Your Tech — AI Chatbot with Tool Calling

An AI-powered customer-support and lead-generation chatbot for **Fly Your Tech**. The application uses **LangChain**, **LangGraph**, **FastAPI**, and a **React** frontend to answer company-related questions, provide service-pricing estimates, capture leads, and optionally schedule consultations.

The chatbot uses LLM-driven tool calling to automatically choose the correct tool based on the user’s intent.

---

## Features

- AI chatbot with tool calling.
- FastAPI backend.
- React-based chat interface.
- LangChain for LLM integration and tool definitions.
- LangGraph for agent workflow orchestration.
- Company knowledge-base tool.
- Lead-management tool.
- Service-pricing tool.
- Optional appointment-scheduling tool.
- Automatic tool selection by the LLM.
- JSON-based storage for prototype data.
- Easy extension to a database, vector store, authentication, and conversation memory.

---

## System Architecture

### Backend

- Python
- FastAPI
- LangChain
- LangGraph
- Gemini, OpenAI, or another compatible LLM provider
- JSON files for proof-of-concept data storage

### Frontend

- React.js
- Simple chat interface
- REST API communication with the FastAPI backend

### Request Flow

```text
User
  ↓
React Frontend
  ↓
FastAPI POST /chat
  ↓
LangGraph Agent Workflow
  ↓
LLM Determines Whether a Tool Is Needed
  ↓
Tool Execution
  ↓
Tool Result Sent Back to LLM
  ↓
Final LLM Response
  ↓
React Frontend
```

---

## Tools

### Tool 1: Company Knowledge Base

This tool retrieves company information from a JSON knowledge base.

Create the file:

```text
backend/data/knowledge_base.json
```

```json
{
  "company_name": "Fly Your Tech",
  "address": "2nd Floor, Tech Park, Bengaluru, India",
  "phone": "+91-9876543210",
  "email": "contact@flyyourtech.com",
  "services": [
    "AI Solutions",
    "Web Development",
    "Mobile App Development",
    "Cloud Consulting",
    "Machine Learning Consulting"
  ],
  "starting_price": "₹25,000"
}
```

---

### Tool 2: Lead Management

This tool stores customer leads for future follow-up.

Create the file:

```text
backend/data/leads.json
```

```json
[]
```

Each saved lead follows this structure:

```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "service": "AI Solutions"
}
```

> This project uses JSON storage for demonstration purposes. In production, use a real database such as PostgreSQL, MySQL, MongoDB, or Supabase.

---

### Tool 3: Service Pricing

This tool provides a starting price estimate for selected Fly Your Tech services.

Example pricing information:

```text
AI Solutions: ₹50,000 onwards
Web Development: ₹30,000 onwards
Mobile App Development: ₹40,000 onwards
Cloud Consulting: ₹35,000 onwards
```

If the requested service is not available in the pricing list, the chatbot responds with a custom-quote recommendation.

---

### Tool 4: Appointment Scheduling (Optional)

This optional tool allows users to schedule a demo or consultation.

Create the file:

```text
backend/data/appointments.json
```

```json
[]
```

Each appointment follows this structure:

```json
{
  "name": "abcd",
  "email": "abcd@gmail.com",
  "date": "2026-02-20",
  "time": "15:00"
}
```

> For a production system, validate date/time input, confirm available time slots, handle time zones, and store appointments in a database or calendar service.

---

## Backend Setup

### Install Dependencies

From the `backend` directory, create and activate a virtual environment.

#### Windows Command Prompt

```cmd
python -m venv .venv
.venv\Scripts\activate
```

#### Windows PowerShell

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

#### macOS / Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install dependencies:

```bash
python -m pip install --upgrade pip
python -m pip install fastapi uvicorn langchain langgraph pydantic python-dotenv
```

Depending on the chosen LLM provider, also install the appropriate package.

For OpenAI:

```bash
python -m pip install langchain-openai
```

For Gemini:

```bash
python -m pip install langchain-google-genai
```

---

## Environment Variables

Create a `.env` file inside the `backend` folder.

### OpenAI Example

```env
OPENAI_API_KEY=your_openai_api_key_here
```

### Gemini Example

```env
GOOGLE_API_KEY=your_google_api_key_here
```

> Never commit `.env` files, API keys, tokens, or credentials to GitHub. Add `.env` to `.gitignore`.

---

## Tool Implementation

Create:

```text
backend/app/tools.py
```

```python
import json
from pathlib import Path

from langchain_core.tools import tool


BASE_DIR = Path(__file__).resolve().parent.parent
DATA_DIR = BASE_DIR / "data"

KNOWLEDGE_BASE_PATH = DATA_DIR / "knowledge_base.json"
LEADS_PATH = DATA_DIR / "leads.json"
APPOINTMENTS_PATH = DATA_DIR / "appointments.json"


@tool
def knowledge_base_tool(query: str) -> str:
    """Retrieve Fly Your Tech company information from the knowledge base."""

    with open(KNOWLEDGE_BASE_PATH, "r", encoding="utf-8") as file:
        knowledge_base = json.load(file)

    query_lower = query.lower()

    if "company" in query_lower or "name" in query_lower:
        return f"Company Name: {knowledge_base['company_name']}"

    if "address" in query_lower or "location" in query_lower:
        return f"Address: {knowledge_base['address']}"

    if "phone" in query_lower or "contact number" in query_lower:
        return f"Phone: {knowledge_base['phone']}"

    if "email" in query_lower or "contact email" in query_lower:
        return f"Email: {knowledge_base['email']}"

    if "service" in query_lower:
        services = ", ".join(knowledge_base["services"])
        return f"Services: {services}"

    if "starting price" in query_lower or "base price" in query_lower:
        return f"Starting Price: {knowledge_base['starting_price']}"

    return "Sorry, I could not find that company information."


@tool
def lead_management_tool(name: str, email: str, service: str) -> str:
    """Save a customer lead with name, email, and requested service."""

    try:
        with open(LEADS_PATH, "r", encoding="utf-8") as file:
            leads = json.load(file)
    except (FileNotFoundError, json.JSONDecodeError):
        leads = []

    new_lead = {
        "name": name,
        "email": email,
        "service": service
    }

    leads.append(new_lead)

    with open(LEADS_PATH, "w", encoding="utf-8") as file:
        json.dump(leads, file, indent=2)

    return f"Lead saved successfully for {name} regarding {service}."


@tool
def pricing_tool(service: str) -> str:
    """Return an estimated starting price for a Fly Your Tech service."""

    pricing = {
        "ai solutions": "₹50,000 onwards",
        "web development": "₹30,000 onwards",
        "mobile app development": "₹40,000 onwards",
        "cloud consulting": "₹35,000 onwards",
        "machine learning consulting": "Please contact sales for a custom quote."
    }

    service_key = service.strip().lower()

    if service_key in pricing:
        return f"Estimated price for {service}: {pricing[service_key]}"

    return "Please contact our sales team for a custom quote."


@tool
def scheduling_tool(name: str, email: str, date: str, time: str) -> str:
    """Schedule a demo or consultation appointment for a customer."""

    try:
        with open(APPOINTMENTS_PATH, "r", encoding="utf-8") as file:
            appointments = json.load(file)
    except (FileNotFoundError, json.JSONDecodeError):
        appointments = []

    appointment = {
        "name": name,
        "email": email,
        "date": date,
        "time": time
    }

    appointments.append(appointment)

    with open(APPOINTMENTS_PATH, "w", encoding="utf-8") as file:
        json.dump(appointments, file, indent=2)

    return f"Appointment scheduled for {name} on {date} at {time}."
```

---

## LangGraph Agent Setup

Create:

```text
backend/app/agent.py
```

```python
import os

from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from langgraph.prebuilt import create_react_agent

from app.tools import (
    knowledge_base_tool,
    lead_management_tool,
    pricing_tool,
    scheduling_tool
)

load_dotenv()

llm = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0,
    api_key=os.getenv("OPENAI_API_KEY")
)

tools = [
    knowledge_base_tool,
    lead_management_tool,
    pricing_tool,
    scheduling_tool
]

agent = create_react_agent(
    model=llm,
    tools=tools
)


def run_agent(message: str) -> str:
    """Run the LangGraph agent and return the final assistant response."""

    result = agent.invoke(
        {
            "messages": [
                HumanMessage(content=message)
            ]
        }
    )

    return result["messages"][-1].content
```

> This example uses `langchain-openai` and OpenAI's `gpt-4o-mini` model. If you use Gemini, replace the LLM setup with the appropriate Gemini LangChain integration.

---

## FastAPI Server

Create:

```text
backend/app/main.py
```

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel

from app.agent import run_agent


app = FastAPI(
    title="Fly Your Tech AI Chatbot API",
    version="1.0.0"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"]
)


class ChatRequest(BaseModel):
    message: str


class ChatResponse(BaseModel):
    response: str


@app.get("/")
def health_check():
    return {
        "status": "ok",
        "message": "Fly Your Tech AI Chatbot API is running."
    }


@app.post("/chat", response_model=ChatResponse)
def chat(request: ChatRequest):
    response = run_agent(request.message)
    return {
        "response": response
    }
```

---

## Run the Backend

From the `backend` directory, run:

```bash
uvicorn app.main:app --reload
```

The FastAPI backend will start at:

```text
http://127.0.0.1:8000
```

Open the API documentation at:

```text
http://127.0.0.1:8000/docs
```

---

## React Frontend

Create or update:

```text
frontend/src/App.js
```

```jsx
import React, { useState } from "react";

function App() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);

  const sendMessage = async () => {
    const trimmedInput = input.trim();

    if (!trimmedInput || loading) {
      return;
    }

    setMessages((currentMessages) => [
      ...currentMessages,
      {
        role: "user",
        content: trimmedInput
      }
    ]);

    setInput("");
    setLoading(true);

    try {
      const response = await fetch("http://localhost:8000/chat", {
        method: "POST",
        headers: {
          "Content-Type": "application/json"
        },
        body: JSON.stringify({
          message: trimmedInput
        })
      });

      if (!response.ok) {
        throw new Error("Unable to receive a response from the chatbot.");
      }

      const data = await response.json();

      setMessages((currentMessages) => [
        ...currentMessages,
        {
          role: "assistant",
          content: data.response
        }
      ]);
    } catch (error) {
      setMessages((currentMessages) => [
        ...currentMessages,
        {
          role: "assistant",
          content: `Error: ${error.message}`
        }
      ]);
    } finally {
      setLoading(false);
    }
  };

  const handleKeyDown = (event) => {
    if (event.key === "Enter") {
      sendMessage();
    }
  };

  return (
    <div style={{ maxWidth: "800px", margin: "0 auto", padding: "20px" }}>
      <h2>Fly Your Tech AI Chatbot</h2>

      <div
        style={{
          height: "400px",
          overflowY: "auto",
          border: "1px solid #cccccc",
          borderRadius: "8px",
          padding: "16px",
          marginBottom: "16px"
        }}
      >
        {messages.length === 0 && (
          <p>Ask me about Fly Your Tech services, pricing, or consultations.</p>
        )}

        {messages.map((message, index) => (
          <div key={index}>
            <p>
              <strong>
                {message.role === "user" ? "You" : "Assistant"}:
              </strong>{" "}
              {message.content}
            </p>
          </div>
        ))}

        {loading && <p><strong>Assistant:</strong> Thinking...</p>}
      </div>

      <div style={{ display: "flex", gap: "8px" }}>
        <input
          type="text"
          value={input}
          onChange={(event) => setInput(event.target.value)}
          onKeyDown={handleKeyDown}
          placeholder="Type your message..."
          style={{ flex: 1, padding: "10px" }}
        />

        <button onClick={sendMessage} disabled={loading}>
          Send
        </button>
      </div>
    </div>
  );
}

export default App;
```

---

## Run the Frontend

From the `frontend` directory:

```bash
npm install
npm start
```

The React application usually runs at:

```text
http://localhost:3000
```

---

## Example Queries

### Company Information

```text
What services does Fly Your Tech offer?
```

```text
What is the starting price?
```

```text
Give me your contact email.
```

```text
Where is Fly Your Tech located?
```

### Service Pricing

```text
How much does web development cost?
```

```text
What is the estimated price for AI solutions?
```

```text
What is the cost of mobile app development?
```

### Lead Creation

```text
I am interested in AI Solutions. 
```

```text
I need web development services. 
```

### Appointment Scheduling

```text
Book a demo for me on 2026-02-20 at 15:00. 
```

```text
Schedule a consultation for Rahul at 11:00 tomorrow. 
```

> For real deployment, the chatbot should ask for clarification when a relative date such as “tomorrow” is used. It should convert dates to a standard format and validate availability before confirming an appointment.

---

## Project Structure

```text
fly-your-tech-chatbot/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                  # FastAPI application and API routes
│   │   ├── agent.py                 # LangGraph agent configuration
│   │   ├── tools.py                 # Knowledge, lead, pricing, and scheduling tools
│   │   ├── schemas.py               # Optional Pydantic request/response models
│   │   └── config.py                # Optional application configuration
│   │
│   ├── data/
│   │   ├── knowledge_base.json      # Company knowledge base
│   │   ├── leads.json               # Lead storage
│   │   └── appointments.json        # Appointment storage
│   │
│   ├── .env                         # API keys; do not commit
│   ├── requirements.txt             # Python dependencies
│   └── README.md                    # Backend documentation
│
├── frontend/
│   ├── public/
│   │   └── index.html
│   ├── src/
│   │   ├── App.js                   # Chat interface
│   │   ├── index.js                 # React entry point
│   │   ├── api.js                   # Optional API helper functions
│   │   └── styles.css               # Frontend styles
│   ├── package.json
│   └── README.md                    # Frontend documentation
│
├── docs/
│   ├── architecture.md               # System-design explanation
│   └── demo_queries.md               # Demonstration queries
│
├── .gitignore
├── README.md
└── run.sh                            # Optional startup script
```

---

## Evaluation Checklist

| Requirement | Status |
|---|---|
| Python backend | ✅ |
| FastAPI API | ✅ |
| LangChain integration | ✅ |
| LangGraph workflow | ✅ |
| Knowledge-base tool | ✅ |
| Lead-management tool | ✅ |
| Pricing tool | ✅ |
| Automatic tool selection | ✅ |
| React frontend | ✅ |
| Company knowledge base | ✅ |
| Simulated lead storage | ✅ |
| Appointment-scheduling tool | ✅ Optional enhancement |

---

## Optional Enhancements

- Add a vector database such as FAISS, Chroma, or Pinecone for semantic knowledge-base retrieval.
- Replace JSON data storage with PostgreSQL, MongoDB, Supabase, or Firebase.
- Add conversation history and persistent chat sessions.
- Add user authentication.
- Add structured logging and tracing for tool calls.
- Add user-friendly frontend styling with Tailwind CSS, Material UI, or Bootstrap.
- Add response streaming.
- Add input validation for email addresses, dates, and appointment times.
- Integrate Google Calendar or Microsoft Outlook for real appointment booking.
- Add rate limiting and role-based tool permissions.
- Deploy the backend with Railway, Render, AWS, or Docker.
- Deploy the frontend with Vercel, Netlify, or GitHub Pages.

---

## Security Notes

- Do not commit `.env`, API keys, tokens, or credentials to GitHub.
- Keep LLM API keys only on the backend; never expose them in the React frontend.
- Validate user input before writing leads or appointments to storage.
- Validate email formats and sanitize fields before storing them.
- Add authentication, rate limiting, logging, and database access controls before production deployment.
- Treat LLM tool calls as untrusted until their inputs are validated.
- Use a production database instead of JSON files for concurrent or real-user usage.

---

## Disclaimer

This project is a proof of concept for educational, academic, and portfolio use. The JSON-based lead and appointment storage is not designed for production workloads, concurrent access, or sensitive customer data.
