*Context*
You are a Senior Software Architect specializing in AI-agent system design and QA automation tooling. You have deep expertise in requirements elicitation and writing implementation-ready specification documents that any coding agent (Claude, GitHub Copilot, Cursor, or any IDE-integrated LLM) can build from without further clarification.

[MANDATORY]Ask exactly 15 multiple-choice questions, one at a time, covering the full system design below
For each question, provide 3-4 answer options (labeled A/B/C/D) and mark the option you'd suggest as "Recommended," with a one-line reason.one-question-at-a-time after answering one question proceed with the next question, then produce a complete Architecture.md specification file.

*Tech stack*
Front end :React.js+Typescript
Back end :Node.js+Express
Pixel by pixel comparison -Resemble.js
Vision LLM for layout and text analysis -Groq model (Configurable via .env)
Reporting model for severity scoring - Groq model (Configurable via .env)

*INSTRUCTION*
[MANDATORY] Build API First and then proceed with UI/UX design. Ensure all endpoints are RESTful and follow best practices for versioning and error handling.

*Below Are Mandatory*

1) Entry point: User manually uploads image(s); no browser automation (no Playwright).
2) Two top-level menu paths:
   - Current Sprint (New Feature/Page)
   - Regression (Existing Feature/Page)
3) Current Sprint (New Feature/Page):
   - Single image, no baseline.
   - Vision-LLM only (checklist: layout, alignment, text, spacing, colour, missing/extra elements).
4) Regression (Existing Feature/Page):
   - Two images (Baseline + Current).
   - Three selectable sub-modes:
     - Pixel-by-Pixel
     - Text-Extraction Diff
     - Hybrid (Pixel + Text)
   4.1) Regression mode shall support automatic retrieval of the Baseline Screenshot from JIRA and the Executed Screenshot from TestRail via configurable REST APIs, in addition to manual image upload.
   4.2) JIRA and TestRail integrations must be optional and enabled/disabled through `.env` configuration; no hardcoded URLs, credentials, or project mappings.
   4.3) The system shall map JIRA issues to TestRail test cases/results using configurable fields or mapping rules.
   4.4) After fetching both images, the application shall automatically invoke the visual comparison pipeline without requiring additional user intervention.
   4.5) If either screenshot cannot be retrieved, the application shall provide meaningful error messages and allow manual image upload as a fallback.
   4.6) The comparison workflow, reporting, severity scoring, and downloadable HTML report shall remain identical regardless of whether images are uploaded manually or fetched from JIRA and TestRail.
5) Pixel-level comparison must always use a deterministic image comparison library—never an LLM.
6) The reporting step must use a different LLM from the Vision Comparison model and shall be served via Groq.
7) Every menu option, sub-menu option, model selection, threshold, API endpoint, integration toggle, and configurable parameter must be controlled through `.env`; nothing shall be hardcoded.
8) A configurable scoring mechanism must convert comparison findings into numeric and severity scores.
9) The application shall generate a downloadable HTML report containing tables, pie charts, comparison summary, detected issues, severity scores, and validation results for both Current Sprint (New Feature/Page) and Regression (Existing Feature/Page).
10) The solution must run in Claude, GitHub Copilot, or any IDE-based coding agent. The specification must remain completely tool-agnostic.