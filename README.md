# Symbolix AI

Symbolix AI is a tool that turns course PDFs (with equations) into personalized study materials.

## Project Structure

- `web/`: Next.js application (Frontend & API)
- `symbolix_mvp_features.md`: Detailed feature specifications and roadmap

## Getting Started

### Prerequisites

- Node.js (v20.9.0 or later recommended)
- npm or yarn

### Running the Web Application

1. Navigate to the web directory:
   ```bash
   cd web
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Run the development server:
   ```bash
   npm run dev
   ```

4. Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.

## Features (MVP)

See [symbolix_mvp_features.md](./symbolix_mvp_features.md) for the detailed feature list.

### Core Features
- User Authentication & Onboarding
- PDF Upload & Storage
- Document Processing & Text Extraction (Mathpix)
- Document Viewer with LaTeX support
- AI Quiz Generation
- AI Tutor Chat
