# N8N-Fees-Charges-Explainer-Agent-
Build a conversational agent in N8N that explains HDFC mutual fund fees using official sources, with chat memory, approval gates, and audit trails.

## 🎯 Project Overview
Build a conversational agent in N8N that explains HDFC mutual fund fees using official sources, with chat memory, approval gates, and audit trails.

---

## 📋 Stage-by-Stage Implementation Plan

### **Stage 1: Environment Setup & Configuration** ⚙️
**Goal:** Set up project structure and configuration files

**What Cursor Will Build:**
- Project folder structure
- Fund registry JSON (6 HDFC funds with URLs)
- Environment variables template
- N8N workflow stubs (empty workflows ready to populate)

**Key Files:**
- `config/fund-registry.json` - Hardcoded 6 funds
- `.env.example` - All configuration
- `README.md` - Setup instructions

**Time:** 1 day

---

### **Stage 2: Chat Memory System** 🧠
**Goal:** Implement persistent conversation memory within N8N

**What Cursor Will Build:**
- Session state management (using N8N variables or simple JSON storage)
- Conversation context tracking:
  - Previous questions/answers
  - Fund selection
  - Fee type selection
  - User preferences
- Memory retrieval functions
- Session timeout logic (30 min inactivity)

**N8N Implementation:**
- Use **N8N Sticky Notes** or **Set Variable** nodes for session data
- Store conversation history in temporary JSON structure
- Pass context forward through workflow

**Key Functions:**
```javascript
// Store conversation turn
storeConversationTurn(sessionId, userMessage, agentResponse)

// Retrieve conversation history
getConversationHistory(sessionId)

// Extract entities from conversation
extractEntities(conversationHistory) // Returns: { fund, feeType, investmentType }
```

**Time:** 2 days

---

### **Stage 3: Preprocessing Workflow - Web Scraping** 🕷️
**Goal:** Scrape and cache fund data from Groww.in

**What Cursor Will Build:**
- HTTP Request node configuration for Groww.in
- Cheerio-based HTML parser for extracting:
  - Expense Ratio
  - Minimum SIP
  - Exit Load
  - NAV + date
- Caching mechanism (24-hour TTL)
- Rate limiting (2-second delay between requests)

**N8N Nodes Used:**
- **HTTP Request** - Fetch Groww.in pages
- **Code (JavaScript)** - Parse HTML with Cheerio
- **IF** - Check cache validity
- **Set Variable** - Store cache

**Anti-Rate-Limit Strategy:**
- Cache all 6 funds on first run
- Refresh cache only once per day (3 AM)
- Use cached data for 99% of queries
- Max 6 scrapes/day = well under free limits

**Key Functions:**
```javascript
// Scrape single fund
scrapeFundData(fundUrl)

// Parse Groww.in HTML
parseGrowwPage(html)

// Cache management
getCachedData(fundId)
setCachedData(fundId, data)
isCacheValid(timestamp)
```

**Time:** 2 days

---

### **Stage 4: Main Orchestration - Conversation Flow** 💬
**Goal:** Handle user queries with contextual understanding

**What Cursor Will Build:**
- Intent detection (What is the user asking?)
  - Fund identification from query
  - Fee type identification
  - Ambiguity detection
- Clarifying question logic
  - Only ask if genuinely unclear
  - Use conversation memory to avoid repeat questions
- Response generation with citations
- Approval gate before MCP actions

**N8N Nodes Used:**
- **Webhook** - Receive user messages
- **Code (JavaScript)** - Intent detection, NLU logic
- **IF** - Route based on clarity
- **AI Agent** (optional) - Use N8N AI Agent for better NLU
- **Wait for Webhook** - Approval gate

**Conversation Flow:**
```
User: "What's the expense ratio for ELSS?"
↓
[Check memory: No fund ambiguity, clear fee type]
↓
[Fetch cached data for HDFC ELSS]
↓
[Generate response with citation]
↓
Agent: "• Expense Ratio: 0.64% annually [Source: groww.in/...] Last checked: 04-Jan-2025"
↓
[Ask for approval: "Save this info?"]
```

**Key Functions:**
```javascript
// Detect intent from user message + history
detectIntent(userMessage, conversationHistory)

// Check if clarification needed
needsClarification(intent)

// Generate clarifying question
generateClarifyingQuestion(missingInfo)

// Format response with citations
formatResponse(fundData, feeType, extractedDate)
```

**Time:** 3 days

---

### **Stage 5: MCP Actions Workflow** 📝
**Goal:** Execute approved actions (notes, email, audit)

**What Cursor Will Build:**

#### 5A: **Google Sheets Integration** (Notes Entry)
- N8N Google Sheets node setup
- Append row with: `{date, fund, fee_type, answer, source_link}`
- Error handling

**N8N Node:** Google Sheets → Append Row

#### 5B: **Gmail Integration** (Email Draft)
- N8N Gmail node setup
- Draft email template with bullet points
- **Do NOT send** - create draft only
- Include: Fund name, fee details, source links

**N8N Node:** Gmail → Create Draft

#### 5C: **Audit Trail** (Free Option)
**Recommended Free Tool:** **Google Sheets** (separate sheet for audit)

**Alternative:** JSON file stored in N8N workflow data

**Structure:**
```json
{
  "timestamp": "2025-01-04T10:30:00Z",
  "user": "anonymous",
  "action": "fees_query",
  "fund": "HDFC ELSS",
  "fee_type": "expense_ratio",
  "approved": true,
  "note_created": true,
  "email_drafted": true
}
```

**N8N Implementation:**
- **Option 1:** Google Sheets → Append to "Audit_Log" sheet
- **Option 2:** MongoDB Atlas Free Tier (512MB)
- **Option 3:** Supabase Free Tier (500MB PostgreSQL)

**Recommendation:** Use **Google Sheets** for both notes and audit - simpler, free, familiar interface.

**Time:** 2 days

---

### **Stage 6: Approval Gate & Manual Controls** ✅
**Goal:** User must explicitly approve before actions execute

**What Cursor Will Build:**
- Approval request message with preview
- Wait for user response (approve/cancel)
- Timeout handling (5 minutes)
- Post-approval action execution
- Cancellation flow

**N8N Implementation:**
```
[Generate Response]
↓
[Show Preview to User]
↓
[Ask: "Type APPROVE to save, CANCEL to discard"]
↓
[Wait for Webhook Response] ← Pause here
↓
IF approved:
  → [Execute MCP Actions]
  → [Send confirmation]
IF cancelled:
  → [Skip actions]
  → [Send "Nothing saved" message]
IF timeout (5 min):
  → [Auto-cancel]
```

**N8N Nodes:**
- **Wait for Webhook** - Pause workflow
- **IF** - Check approval status
- **Code** - Handle timeout

**Time:** 1 day

---

### **Stage 7: Error Handling & Edge Cases** 🛡️
**Goal:** Graceful failure and clear error messages

**What Cursor Will Build:**
- Network error handling (Groww.in unreachable)
- Parsing errors (page structure changed)
- Rate limit detection
- Invalid fund/fee requests
- Out-of-scope query rejection
- Friendly error messages

**Error Scenarios:**
```javascript
// Groww.in unreachable
→ "Sorry, I can't fetch live data right now. Using cached data from [date]."

// Fund not in registry
→ "I only have information for HDFC funds: Flexi Cap, Large Cap, Mid Cap, Small Cap, Multi Cap, ELSS."

// Recommendation request
→ "I can only provide factual fee information, not recommendations."

// Missing data on page
→ "• NAV: [Not Available - Check Source]"
```

**Time:** 2 days

---

### **Stage 8: Testing & Documentation** ✅
**Goal:** Comprehensive testing and user guides

**What Cursor Will Build:**
- Test cases document (10+ scenarios)
- User guide (how to use the agent)
- Admin guide (N8N workflow import instructions)
- Troubleshooting guide
- API usage monitoring script

**Test Cases:**
1. Happy path: Full query → approval → actions
2. Clarification: Ambiguous query → clarify → answer
3. Memory: Multi-turn conversation
4. Out-of-scope: Non-HDFC fund request
5. Network error: Groww.in down
6. Approval timeout: User doesn't respond
7. Cancellation: User cancels request

**Time:** 2 days

---

## 🚀 Implementation Order

| Week | Stages | Focus |
|------|--------|-------|
| **Week 1** | Stages 1-2 | Setup + Memory |
| **Week 2** | Stages 3-4 | Scraping + Conversation |
| **Week 3** | Stages 5-6 | MCP Actions + Approval |
| **Week 4** | Stages 7-8 | Error Handling + Testing |

**Total Time:** ~16 days

---

## 🔧 Free Tools Recommended

| Purpose | Tool | Free Tier |
|---------|------|-----------|
| **N8N Hosting** | N8N Cloud | 5,000 executions/month |
| **Notes Storage** | Google Sheets | Unlimited (reasonable use) |
| **Email Drafts** | Gmail API | 1B quota units/day |
| **Audit Trail** | Google Sheets | Same as above |
| **Web Scraping** | HTTP Request (N8N built-in) | No API key needed |
| **HTML Parsing** | Cheerio (JavaScript library) | Free, no limits |

**Total API Calls/Day:**
- Groww.in scrapes: ~6/day (once per fund per day)
- Google Sheets writes: ~10-20/day
- Gmail drafts: ~5-10/day

**All well within free limits!** ✅

---

## ⚠️ Pitfalls to Avoid

### 1. **Chat Memory Pitfall**
❌ **Don't:** Store memory in external database (adds complexity)
✅ **Do:** Use N8N workflow variables or simple JSON in Set nodes
   - Persist for session only (30 min)
   - Clear on session end

### 2. **Rate Limiting Pitfall**
❌ **Don't:** Scrape on every query
✅ **Do:** Cache aggressively
   - Pre-scrape all 6 funds once daily (3 AM)
   - Serve from cache 99% of time
   - Only 6 scrapes/day total

### 3. **Approval Gate Pitfall**
❌ **Don't:** Auto-execute MCP actions
✅ **Do:** ALWAYS wait for explicit "APPROVE"
   - Use N8N "Wait for Webhook" node
   - Timeout after 5 minutes
   - Show preview before approval

### 4. **Error Message Pitfall**
❌ **Don't:** Show technical errors to users ("Error 500", "Parse failed")
✅ **Do:** User-friendly messages
   - "I'm having trouble right now. Using data from [date]."
   - Always have a fallback response

### 5. **Scope Creep Pitfall**
❌ **Don't:** Try to answer everything (recommendations, comparisons)
✅ **Do:** Politely decline out-of-scope
   - "I can only provide factual fee information."
   - "I only cover HDFC funds."

### 6. **Citation Pitfall**
❌ **Don't:** Make up numbers or dates
✅ **Do:** Always cite exact source
   - Quote verbatim from Groww.in
   - Include "Last checked: DD-MMM-YYYY"
   - If data missing: "[Not Available - Check Source]"

### 7. **N8N Workflow Complexity Pitfall**
❌ **Don't:** Build one massive workflow
✅ **Do:** Split into 3 workflows
   - Main Orchestration (user-facing)
   - Preprocessing (scraping, runs separately)
   - MCP Actions (triggered by approval)

### 8. **Testing Pitfall**
❌ **Don't:** Test only happy paths
✅ **Do:** Test all failure modes
   - Network down
   - Invalid input
   - Timeout scenarios
   - Partial data

---

## 🎯 Success Metrics

1. ✅ Agent answers all 4 fee types for 6 funds (24 combinations)
2. ✅ Chat memory works across multi-turn conversations
3. ✅ No repeat questions when context is clear
4. ✅ Approval gate blocks unauthorized actions
5. ✅ All sources cited with dates
6. ✅ Stays under 100 API calls/day
7. ✅ 100% of MCP actions require approval
8. ✅ Graceful error handling (no crashes)

---

## 📝 Next Steps

**For each stage, you'll provide Cursor with:**
```
Context: Building Stage X - [Stage Name]
Requirements: [Detailed specs from this plan]
Constraints: [Free tier limits, N8N specifics]
Output: [Exact code/config needed]
```

**Example Prompt for Stage 2 (Chat Memory):**
```
Context: Building Stage 2 - Chat Memory System for N8N Fees Explainer Agent

Requirements:
- Store conversation history per session (user messages + agent responses)
- Extract entities: fund name, fee type, investment type
- Persist for 30 minutes
- Use N8N Set Variable nodes (no external database)

Constraints:
- Must work in N8N workflow
- No external APIs
- Simple JSON structure

Output:
- JavaScript functions for N8N Code nodes
- Session state structure
- Memory retrieval logic
```

---

This plan is optimized for **portfolio projects** with emphasis on:
- ✅ Free tools only
- ✅ Minimal API calls
- ✅ Clear stage-by-stage progression
- ✅ Real-world features (memory, approval, audit)
- ✅ Professional error handling

