# Symbolix AI - MVP Feature List
**Version:** 1.0 MVP  
**Timeline:** 12-16 weeks to launch  
**Goal:** Validate core learning loop with minimal viable feature set

---

## MVP Philosophy

**Core Hypothesis to Test:**
"STEM students will use and pay for an AI tool that turns their actual course PDFs (with equations) into personalized study materials."

**What We're NOT Building (Yet):**
- Mobile apps (web-only for MVP)
- Social/collaborative features
- Advanced document conversion (PDF → Word)
- Flashcard generators
- Study groups
- Browser extensions
- Desktop apps

**Success Metrics for MVP:**
- 100+ beta users
- 40%+ weekly active rate
- 10%+ conversion to paid
- Users completing 3+ study sessions per week

---

## Feature Priority Framework

### Must-Have (Launch Blockers)
Features without which the product doesn't work or provide core value.

### Should-Have (High Value)
Significantly improve experience but MVP can launch without them.

### Nice-to-Have (Defer to V1.1+)
Good ideas but can wait until after validation.

---

## MUST-HAVE FEATURES (MVP Core)

### 1. User Authentication & Onboarding

**Purpose:** Create accounts, set expectations, qualify users

**Features:**
- [ ] Sign up with email + password
- [ ] Sign up with Google OAuth (faster, preferred)
- [ ] Email verification
- [ ] Simple onboarding flow (3 screens max):
  - Screen 1: "What subject are you studying?" (dropdown: Calculus, Physics, Chemistry, Biology, Other)
  - Screen 2: "Upload your first document" (drag-drop or browse)
  - Screen 3: Brief tour of interface (skip-able)

**Technical Requirements:**
- NextAuth.js or Clerk for authentication
- PostgreSQL for user data
- Email service (SendGrid or Resend) for verification
- Store: user_id, email, hashed_password, created_at, subject_preference

**Time Estimate:** 1.5 weeks

**Acceptance Criteria:**
- User can create account in <2 minutes
- Email verification works
- Google sign-in works
- 80%+ complete onboarding flow

---

### 2. PDF Upload & Storage

**Purpose:** Allow students to upload course documents

**Features:**
- [ ] Drag-and-drop PDF upload interface
- [ ] File browser upload (fallback)
- [ ] Upload progress indicator
- [ ] File size limit: 50MB per file (prevents abuse)
- [ ] Page limit: 100 pages per document (MVP constraint)
- [ ] File format validation (PDF only for MVP)
- [ ] Upload history/library view (see all uploaded docs)

**Technical Requirements:**
- AWS S3 or Cloudflare R2 for storage
- Pre-signed upload URLs (secure, client-side upload)
- Database schema:
  ```
  documents table:
  - id (uuid)
  - user_id (foreign key)
  - filename (string)
  - s3_key (string)
  - file_size (integer, bytes)
  - page_count (integer)
  - upload_status (enum: uploading, processing, ready, failed)
  - uploaded_at (timestamp)
  ```
- Background job queue (BullMQ or Celery) for processing

**Restrictions (MVP):**
- Free tier: 3 documents per month
- Premium: Unlimited documents
- Max 50MB per file, 100 pages per document
- Only PDF format (no Word, images, etc.)

**Time Estimate:** 2 weeks

**Acceptance Criteria:**
- Upload completes in <30 seconds for typical document
- Progress bar shows accurate status
- Error handling for oversized/invalid files
- User can view all uploaded documents in library

---

### 3. Document Processing & Text Extraction

**Purpose:** Extract text, equations, and structure from PDFs

**Features:**
- [ ] Text extraction from PDF
- [ ] Equation detection and LaTeX extraction
- [ ] Image/diagram extraction (store separately)
- [ ] Table detection (basic)
- [ ] Maintain page numbers and structure
- [ ] Processing status updates

**Technical Requirements:**
- **Primary Parser:** Mathpix API for equation OCR
  - Cost: ~$0.02-0.05 per page
  - Handles LaTeX, handwritten equations, diagrams
  - API docs: https://mathpix.com/docs
  
- **Fallback Parser:** PyMuPDF (open source) for plain text
  - Use when no equations detected (save API costs)
  - Fast, handles most PDFs well
  
- **Processing Pipeline:**
  1. Upload PDF → S3
  2. Trigger background job
  3. Analyze first 3 pages to detect equations
  4. If equations found → use Mathpix
  5. If plain text → use PyMuPDF
  6. Extract and store:
     - Raw text by page
     - Detected equations (LaTeX format)
     - Images/diagrams (S3 URLs)
     - Document structure (headings, sections)
  7. Update status to "ready"

- **Database Schema:**
  ```
  document_content table:
  - id
  - document_id (foreign key)
  - page_number (integer)
  - raw_text (text)
  - equations (jsonb array of {latex, position})
  - images (jsonb array of {s3_url, position})
  - processed_at (timestamp)
  ```

**Processing Time Targets:**
- 10-page document: <30 seconds
- 50-page document: <2 minutes
- 100-page document: <5 minutes

**Error Handling:**
- Retry failed pages (max 2 retries)
- Partial success OK (e.g., 98/100 pages processed)
- Show user which pages failed
- Allow manual re-processing

**Time Estimate:** 3 weeks (most complex feature)

**Acceptance Criteria:**
- Successfully extracts text from 90%+ of test documents
- Equations render correctly in LaTeX
- Processing doesn't block upload (async)
- User sees clear status ("Processing page 15/50...")

---

### 4. Document Viewer

**Purpose:** Let users view uploaded PDFs and processed content

**Features:**
- [ ] Side-by-side view: Original PDF | Processed content
- [ ] Page navigation (prev/next, jump to page)
- [ ] Zoom controls for PDF
- [ ] Highlight/select text to ask AI tutor
- [ ] Show extracted equations rendered (MathJax)
- [ ] Mobile-responsive (basic)

**Technical Requirements:**
- **Frontend:** PDF.js for rendering PDFs in browser
- **Math Rendering:** MathJax or KaTeX for LaTeX equations
- React component structure:
  ```
  <DocumentViewer>
    <PDFPanel /> // Left side - original PDF
    <ContentPanel /> // Right side - processed text + equations
    <NavigationBar /> // Page controls, zoom
  </DocumentViewer>
  ```

**UI/UX Details:**
- Default view: Show both panels (split 50/50)
- Mobile: Stack vertically or tab between views
- Keyboard shortcuts: arrow keys for pages, +/- for zoom
- Persistent scroll position (user picks up where they left off)

**Time Estimate:** 1.5 weeks

**Acceptance Criteria:**
- PDF renders clearly on desktop and mobile
- Equations display properly (no broken LaTeX)
- Navigation is smooth (<500ms page change)
- Users can select text to copy or highlight

---

### 5. AI Quiz Generation

**Purpose:** Auto-generate practice questions from uploaded documents

**Features:**
- [ ] "Generate Quiz" button on document page
- [ ] Quiz settings (quick modal):
  - Number of questions: 5, 10, or 15 (default: 10)
  - Question types: Multiple choice, True/False, Short answer (default: mix)
  - Difficulty: Easy, Medium, Hard (default: Medium)
  - Focus area: Entire document or specific pages
- [ ] AI generates questions based on document content
- [ ] Show quiz in clean interface
- [ ] Multiple choice: 4 options, 1 correct
- [ ] Short answer: text input
- [ ] Submit quiz and see results
- [ ] Show correct answers with brief explanations
- [ ] Save quiz history (see past quizzes)

**Technical Requirements:**

**AI Prompt Engineering:**
```python
system_prompt = """You are an expert educational assessment creator for STEM subjects.
Generate quiz questions that test conceptual understanding, not just memorization.
Questions should be clear, unambiguous, and at appropriate difficulty level.
For multiple choice, ensure distractors are plausible but clearly wrong.
For math problems, show formulas in LaTeX format: $...$
"""

user_prompt = f"""Based on this document excerpt:
{document_text}

Generate {num_questions} questions with these settings:
- Difficulty: {difficulty}
- Question types: {question_types}
- Focus: {focus_area}

Return as JSON:
{{
  "questions": [
    {{
      "question": "What is...",
      "type": "multiple_choice",
      "options": ["A", "B", "C", "D"],
      "correct_answer": "B",
      "explanation": "...",
      "difficulty": "medium"
    }}
  ]
}}
"""
```

**LLM Choice:**
- **Primary:** Claude Sonnet 4.5 (better instruction following)
- **Cost:** ~$0.03-0.05 per quiz (5K input + 2K output tokens)
- **Fallback:** GPT-4o-mini if Claude has issues

**Database Schema:**
```
quizzes table:
- id
- user_id
- document_id
- num_questions (integer)
- difficulty (enum)
- created_at
- questions (jsonb array)

quiz_attempts table:
- id
- quiz_id
- user_id
- answers (jsonb array)
- score (float, 0-100)
- completed_at
```

**Rate Limiting (MVP):**
- Free tier: 20 quizzes per month
- Premium: Unlimited quizzes
- Cache generated quizzes (if same doc + settings, reuse)

**Time Estimate:** 2 weeks

**Acceptance Criteria:**
- Quizzes generate in <10 seconds
- Questions are relevant to document content
- 90%+ of generated questions are factually correct
- UI is clean and easy to use
- User can review incorrect answers

---

### 6. AI Tutor Chat

**Purpose:** Answer questions about uploaded documents

**Features:**
- [ ] Chat interface for each document
- [ ] User asks question about content
- [ ] AI provides explanation with page references
- [ ] Show relevant equations/diagrams inline
- [ ] Adjustable explanation depth:
  - "Explain like I'm 5"
  - "Standard explanation"
  - "Advanced/technical explanation"
- [ ] Chat history saved per document
- [ ] "Ask about this section" quick action (highlight text → ask)

**Technical Requirements:**

**AI System Prompt:**
```python
system_prompt = """You are an expert STEM tutor helping a student understand their course material.

Guidelines:
- Explain concepts clearly using analogies and examples
- Reference specific pages/sections from the document when relevant
- Use Socratic method: guide students to understanding, don't just give answers
- If student asks for solution to homework problem, help them understand the concept but don't solve it directly
- Format equations in LaTeX: $...$
- Adjust explanation based on student's indicated level
- Be encouraging and patient
"""

context = f"""
Document: {document_title}
Relevant excerpts: {relevant_pages}
Student's question: {user_question}
Explanation level: {difficulty_level}
"""
```

**Retrieval Approach:**
- **Simple MVP:** Send first 50 pages of document as context (within token limits)
- **Smarter (if time):** Use semantic search to find relevant pages:
  1. Embed document pages with OpenAI embeddings
  2. Embed user question
  3. Return top 5 most relevant pages
  4. Include those in context

**LLM Settings:**
- Model: Claude Sonnet 4.5
- Max tokens: 1000 per response (prevent runaway costs)
- Temperature: 0.3 (more focused, less creative)
- Cost per message: ~$0.10-0.30

**Rate Limiting:**
- Free tier: 50 messages per month
- Premium: Unlimited messages
- Show remaining message count

**Database Schema:**
```
chat_sessions table:
- id
- user_id
- document_id
- created_at

chat_messages table:
- id
- session_id
- role (enum: user, assistant)
- content (text)
- tokens_used (integer)
- created_at
```

**UI Details:**
- Floating chat button on document page (bottom right)
- Opens slide-out panel (doesn't cover document)
- Markdown rendering for AI responses
- LaTeX rendering for equations
- Copy response button
- "Ask follow-up" quick actions

**Time Estimate:** 2 weeks

**Acceptance Criteria:**
- Responses arrive in <5 seconds
- Explanations are relevant to document content
- Equations render properly
- Chat history persists between sessions
- Mobile-friendly chat interface

---

### 7. User Dashboard & Navigation

**Purpose:** Central hub for documents, quizzes, and activity

**Features:**
- [ ] Dashboard homepage after login:
  - Recent documents (last 5)
  - Recent quizzes (last 5)
  - Quick stats: Total documents, total quizzes, study time
  - "Upload new document" CTA
- [ ] Library page: All uploaded documents (sortable, searchable)
- [ ] Quiz history page: All past quizzes with scores
- [ ] Settings page: Profile, preferences, subscription
- [ ] Simple navigation bar (Logo, Dashboard, Library, Quizzes, Settings)

**Technical Requirements:**
- React Router for navigation
- Tailwind CSS for responsive layout
- Data fetching: React Query for caching
- Dashboard widgets show aggregated data from DB

**Time Estimate:** 1 week

**Acceptance Criteria:**
- Dashboard loads in <2 seconds
- Users can find their documents easily
- Navigation is intuitive (no confusion)
- Mobile responsive

---

### 8. Subscription & Payment System

**Purpose:** Monetize with freemium model

**Features:**
- [ ] Free tier limits enforced:
  - 3 documents per month
  - 20 quizzes per month
  - 50 AI tutor messages per month
- [ ] "Upgrade to Premium" prompts when limits reached
- [ ] Premium subscription: $9/month or $79/year
- [ ] Stripe integration for payments
- [ ] Subscription management page:
  - Current plan
  - Usage stats
  - Upgrade/cancel options
- [ ] Auto-renewal and cancellation handling
- [ ] Receipt emails

**Technical Requirements:**
- **Stripe Integration:**
  - Create Stripe customer on signup
  - Stripe Checkout for payment
  - Webhooks for subscription events
  - Store subscription status in DB
  
- **Database Schema:**
```
subscriptions table:
- id
- user_id
- stripe_customer_id
- stripe_subscription_id
- plan (enum: free, premium)
- status (enum: active, canceled, past_due)
- current_period_start
- current_period_end
- cancel_at_period_end (boolean)
```

- **Usage Tracking:**
```
usage_tracking table:
- id
- user_id
- month (date, e.g., 2026-02-01)
- documents_uploaded (integer)
- quizzes_generated (integer)
- ai_messages_sent (integer)
```

**Business Logic:**
- Check usage before allowing actions
- Reset monthly counters on subscription anniversary
- Handle grace period for failed payments (3 days)
- Downgrade to free tier if subscription canceled

**Time Estimate:** 2 weeks

**Acceptance Criteria:**
- Payment flow works smoothly (<3 clicks to upgrade)
- Limits are enforced accurately
- Subscription status updates in real-time
- Users can cancel and restart easily

---

### 9. Basic Settings & Preferences

**Purpose:** Let users customize experience

**Features:**
- [ ] Profile settings:
  - Name
  - Email (with verification if changed)
  - Password change
- [ ] Learning preferences:
  - Default explanation level (beginner, standard, advanced)
  - Preferred subjects (for better quiz generation)
- [ ] Notification settings:
  - Email for quiz results
  - Processing complete notifications
- [ ] Account deletion (GDPR compliance)

**Technical Requirements:**
- Form validation (Zod or Yup)
- Email verification for changes
- Soft delete for account deletion (keep data 30 days)
- Settings stored in user table

**Time Estimate:** 1 week

**Acceptance Criteria:**
- Users can update profile without issues
- Email/password changes work securely
- Account deletion complies with data privacy laws

---

### 10. Admin Panel (Internal Use)

**Purpose:** Monitor system health, manage users, view analytics

**Features:**
- [ ] User list with search/filter
- [ ] Document processing status dashboard
- [ ] Usage statistics (DAU, MAU, uploads, quizzes)
- [ ] Error logs and failed processing jobs
- [ ] Manual controls:
  - Retry failed document processing
  - Grant/revoke premium manually (for support)
  - View user activity logs

**Technical Requirements:**
- Protected route (admin-only access)
- React Admin or Retool for quick build
- Read-only access to database
- Aggregated queries for analytics

**Time Estimate:** 1 week

**Acceptance Criteria:**
- Admins can monitor system health
- Can troubleshoot user issues quickly
- Analytics update daily

---

## SHOULD-HAVE FEATURES (High Value, Not Blockers)

### 11. Document Summary Generation

**Purpose:** Quick overview of document content

**Features:**
- [ ] "Summarize" button on document page
- [ ] AI generates 200-300 word summary
- [ ] Highlights key concepts and main topics
- [ ] Extract key terms/definitions
- [ ] Show chapter structure if available

**Why Should-Have:**
Helps users quickly understand document before quizzing, but not essential for core value prop.

**Time Estimate:** 3 days

---

### 12. Progress Tracking

**Purpose:** Show learning progress over time

**Features:**
- [ ] Quiz performance over time (graph)
- [ ] Topics mastered vs. struggling
- [ ] Study streaks (days active)
- [ ] Weekly study time report

**Why Should-Have:**
Improves retention and engagement, but MVP can work without it.

**Time Estimate:** 5 days

---

### 13. Search Within Documents

**Purpose:** Find specific topics quickly

**Features:**
- [ ] Search bar for each document
- [ ] Highlight matches in text
- [ ] Jump to page with result

**Why Should-Have:**
Very useful for large documents, but users can scroll/navigate manually for MVP.

**Time Estimate:** 3 days

---

### 14. Export Quiz Results

**Purpose:** Save quiz data externally

**Features:**
- [ ] Download quiz as PDF
- [ ] Export to CSV for tracking
- [ ] Print-friendly format

**Why Should-Have:**
Nice for record-keeping, but not critical for core learning loop.

**Time Estimate:** 2 days

---

### 15. Dark Mode

**Purpose:** Reduce eye strain for night studying

**Features:**
- [ ] Toggle in settings
- [ ] Persistent preference
- [ ] All pages support dark mode

**Why Should-Have:**
Students love dark mode, but won't block usage if missing.

**Time Estimate:** 3 days

---

## NICE-TO-HAVE FEATURES (Defer to V1.1+)

### 16. Flashcard Generation
Auto-create Anki-style flashcards from document.

**Time:** 1 week  
**Why Defer:** Quizzes serve similar purpose, add later for variety.

---

### 17. Practice Problem Generator
Generate new problems similar to textbook examples.

**Time:** 2 weeks  
**Why Defer:** Complex to ensure accuracy, better after validation.

---

### 18. Collaborative Study Groups
Share documents and quizzes with classmates.

**Time:** 3 weeks  
**Why Defer:** Social features add complexity, focus on solo learning first.

---

### 19. Mobile Apps (iOS/Android)
Native mobile experience.

**Time:** 8+ weeks  
**Why Defer:** Web app works on mobile, native apps after PMF.

---

### 20. Browser Extension
Quick access from any website.

**Time:** 2 weeks  
**Why Defer:** Nice-to-have convenience, not essential for MVP.

---

### 21. Spaced Repetition System
Remind users to review concepts over time.

**Time:** 2 weeks  
**Why Defer:** Advanced retention feature, better once engagement is proven.

---

### 22. Handwritten Notes Upload
OCR for handwritten class notes.

**Time:** 3 weeks  
**Why Defer:** Very hard to do well, focus on typed PDFs first.

---

### 23. Video Lecture Integration
Upload lecture videos, generate notes/quizzes.

**Time:** 4 weeks  
**Why Defer:** Different content type, adds significant complexity.

---

### 24. Multi-Language Support
Interface in Spanish, French, etc.

**Time:** 2 weeks  
**Why Defer:** Target English-speaking students first, expand later.

---

### 25. Gamification
Points, badges, leaderboards.

**Time:** 2 weeks  
**Why Defer:** Can improve retention, but add after core features work well.

---

## Development Timeline (12-16 Weeks)

### Phase 1: Foundation (Weeks 1-4)
**Goal:** Set up infrastructure and basic user flows

**Week 1:**
- [ ] Set up development environment
- [ ] Initialize Next.js project
- [ ] Database schema design and setup (PostgreSQL)
- [ ] AWS S3 bucket setup

**Week 2:**
- [ ] User authentication (Clerk or NextAuth)
- [ ] Basic UI components (Tailwind)
- [ ] Landing page + sign up flow
- [ ] Simple dashboard layout

**Week 3:**
- [ ] PDF upload functionality
- [ ] File storage (S3 integration)
- [ ] Upload progress tracking
- [ ] Document library page

**Week 4:**
- [ ] Background job queue setup (BullMQ)
- [ ] Mathpix API integration
- [ ] Basic document processing pipeline
- [ ] Processing status updates

**Milestone 1:** Users can sign up and upload PDFs that get processed.

---

### Phase 2: Core Features (Weeks 5-8)
**Goal:** Build main learning features

**Week 5:**
- [ ] Document viewer (PDF.js)
- [ ] LaTeX rendering (MathJax)
- [ ] Page navigation
- [ ] Processed content display

**Week 6:**
- [ ] AI quiz generation (Claude API)
- [ ] Quiz UI (questions, answers, results)
- [ ] Quiz history page
- [ ] Basic prompt engineering

**Week 7:**
- [ ] AI tutor chat interface
- [ ] Message handling
- [ ] Context retrieval
- [ ] Chat history storage

**Week 8:**
- [ ] Stripe integration
- [ ] Subscription management
- [ ] Usage tracking
- [ ] Limit enforcement

**Milestone 2:** Complete learning loop works: upload → quiz → chat.

---

### Phase 3: Polish & Launch Prep (Weeks 9-12)
**Goal:** Refine UX and prepare for beta

**Week 9:**
- [ ] Settings and preferences
- [ ] Profile management
- [ ] Email notifications
- [ ] Error handling improvements

**Week 10:**
- [ ] Admin panel
- [ ] Analytics dashboard
- [ ] Monitoring and logging
- [ ] Performance optimization

**Week 11:**
- [ ] UI/UX polish
- [ ] Mobile responsiveness
- [ ] Loading states and animations
- [ ] Onboarding flow refinement

**Week 12:**
- [ ] Beta testing with 20 users
- [ ] Bug fixes
- [ ] Documentation
- [ ] Launch preparation

**Milestone 3:** Product ready for public beta.

---

### Phase 4: Optional Enhancements (Weeks 13-16)
**Goal:** Add should-have features based on feedback

**Week 13:**
- [ ] Document summaries
- [ ] Search within documents
- [ ] Dark mode

**Week 14:**
- [ ] Progress tracking
- [ ] Export quiz results
- [ ] Performance improvements

**Week 15-16:**
- [ ] Bug fixes from beta feedback
- [ ] A/B testing setup
- [ ] Analytics refinement
- [ ] Marketing site polish

**Milestone 4:** Polished MVP ready for growth phase.

---

## Technical Stack Summary

### Frontend
- **Framework:** Next.js 14 (React + TypeScript)
- **Styling:** Tailwind CSS
- **UI Components:** shadcn/ui (Radix primitives)
- **PDF Rendering:** PDF.js
- **Math Rendering:** MathJax or KaTeX
- **State Management:** React Query (server state) + Zustand (client state)
- **Forms:** React Hook Form + Zod validation

### Backend
- **Framework:** Next.js API routes (or FastAPI if prefer Python)
- **Database:** PostgreSQL (Supabase or Railway for hosting)
- **ORM:** Prisma (TypeScript) or SQLAlchemy (Python)
- **Background Jobs:** BullMQ (Redis-based)
- **File Storage:** AWS S3 or Cloudflare R2
- **Authentication:** Clerk or NextAuth.js
- **Payments:** Stripe

### AI/ML
- **LLM:** Claude 4.5 Sonnet (Anthropic API)
- **OCR:** Mathpix API
- **Embeddings:** (If adding semantic search) OpenAI text-embedding-3-small

### Infrastructure
- **Hosting:** Vercel (frontend + API) or Railway
- **Database:** Supabase or Railway
- **Redis:** Upstash or Railway
- **Monitoring:** Sentry (errors) + PostHog (analytics)
- **Email:** Resend or SendGrid

### DevOps
- **Version Control:** GitHub
- **CI/CD:** GitHub Actions
- **Environments:** Dev, Staging, Production
- **Secrets Management:** Vercel env vars or Doppler

---

## Cost Estimates (Monthly, MVP Scale)

### Infrastructure
| Service | Cost |
|---------|------|
| Vercel (Pro plan) | $20 |
| Supabase (Pro plan) | $25 |
| Upstash Redis | $10 |
| AWS S3 | $5-20 |
| **Total** | **$60-75** |

### APIs (100 active users)
| Service | Usage | Cost |
|---------|-------|------|
| Mathpix (OCR) | 300 docs × $0.03/page × 30 pages avg | $270 |
| Claude API (quizzes) | 2000 quizzes × $0.04 | $80 |
| Claude API (chat) | 5000 messages × $0.15 | $750 |
| **Total** | | **$1,100** |

### Other
| Service | Cost |
|---------|------|
| Clerk auth (free tier) | $0 |
| Stripe (fees) | 2.9% + $0.30 per transaction |
| Sentry (free tier) | $0 |
| PostHog (free tier) | $0 |
| **Total** | **~$0** |

### Total Monthly Costs (100 users): ~$1,200
- Revenue at 10% conversion ($9/mo × 10 users): $90
- Net: -$1,110/month (before hitting paid tier volume)

**Cost Optimization Strategies:**
1. Cache quiz questions (reuse for similar documents)
2. Start with GPT-4o-mini for quizzes ($0.15 vs $3 per 1M tokens)
3. Implement aggressive rate limiting on free tier
4. Use cheaper embedding models or skip semantic search initially

---

## Quality Assurance Checklist

### Before Beta Launch
- [ ] Security audit (basic): SQL injection, XSS, CSRF protection
- [ ] Data encryption at rest and in transit
- [ ] GDPR compliance: Privacy policy, data deletion, consent
- [ ] Rate limiting on all API endpoints
- [ ] Error handling and user-friendly messages
- [ ] Mobile responsive (test on iOS and Android browsers)
- [ ] Cross-browser testing (Chrome, Safari, Firefox)
- [ ] Load testing (can handle 100 concurrent users?)
- [ ] Backup strategy (automated DB backups)
- [ ] Monitoring and alerts (Sentry for errors, uptime monitoring)

### Test Cases (Critical Paths)
- [ ] Sign up → upload document → generate quiz → take quiz
- [ ] Sign up → upload document → ask AI tutor → get response
- [ ] Free user hits limit → sees upgrade prompt → pays → limit removed
- [ ] Premium user uploads 10 documents → all process successfully
- [ ] User uploads broken PDF → sees clear error message
- [ ] User uploads 200-page document → gets error about page limit

---

## Success Criteria for MVP

### Quantitative
- [ ] 100+ beta users signed up
- [ ] 60%+ activation rate (complete first action)
- [ ] 40%+ weekly active users
- [ ] 10%+ conversion to premium
- [ ] 3+ average sessions per week per active user
- [ ] <5% error rate in document processing
- [ ] <10 second average quiz generation time

### Qualitative
- [ ] Positive feedback from 70%+ of beta users
- [ ] Users describe it as "helpful" or "time-saving"
- [ ] Low support burden (<5 support requests per week)
- [ ] Users voluntarily recommend to friends

### Technical
- [ ] 99%+ uptime
- [ ] <3 second page load time (75th percentile)
- [ ] Zero security incidents
- [ ] <$15 per premium user in API costs

---

## Post-MVP Roadmap Preview

### V1.1 (Weeks 17-20)
Focus on retention and viral growth:
- Practice problem generation
- Referral program
- Progress tracking dashboard
- Mobile app (React Native)

### V1.2 (Weeks 21-28)
Expand capabilities:
- Multi-document context
- Flashcard generation
- Better equation handling
- Study groups (basic collaboration)

### V2.0 (Months 7-12)
Platform expansion:
- PDF-to-Word conversion (premium feature)
- API for third-party integrations
- University partnerships
- Advanced analytics

---

## Final Notes

**Development Team Recommendation:**
- **Minimum:** 1 full-stack developer (16 weeks full-time)
- **Ideal:** 2 developers (8-10 weeks, parallel work)
- **With design help:** +1 designer (can be contractor, 20-30 hours)

**Biggest Risks:**
1. **Document processing accuracy:** Mitigate with Mathpix API, extensive testing
2. **AI response quality:** Mitigate with prompt engineering, monitoring, human review
3. **User acquisition:** Mitigate with early beta program, student ambassadors
4. **Cost overruns:** Mitigate with aggressive rate limiting, caching, cost monitoring

**When to Launch Beta:**
After week 12 when core loop works. Don't wait for perfection—get feedback early.

**When to Launch Publicly:**
After 100+ beta users with positive feedback and 40%+ retention. Usually week 16-20.

---

## Next Steps

1. **Week 1:** Set up development environment, design database schema
2. **Week 2:** Build authentication and basic upload
3. **Week 3:** Integrate Mathpix and test document processing
4. **Week 4:** Review progress, adjust timeline based on learnings

Good luck building! 🚀
