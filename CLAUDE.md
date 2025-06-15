# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Instructions

- Talk, chat in Korean as default.
- Write documents in Korean as default.
- Plan and check with a todo list to handle tasks.
- Write and edit the todo list in this document.

## Repository Overview

This is a comprehensive Korean-language documentation guide for Server-Sent Events (SSE) technology. The repository contains 13 markdown files covering SSE from W3C standards to production implementations, including real-world examples from major tech companies.

## Repository Structure

- **Main documentation**: 12 numbered parts (part1 through part12) covering different aspects of SSE
- **References directory**: 8 research documents that were used as source material
- **No code files**: This is a documentation-only repository with no build/test commands needed

## Documentation Organization

The guide follows a logical progression:

1. **Conceptual foundation** (Parts 1-2): W3C standards and working principles
2. **Industry patterns** (Parts 3-5): Big tech patterns, AI chatbot implementations, Next.js/Vercel
3. **Implementation details** (Parts 6-9): Browser data handling, error handling, performance, security
4. **Production readiness** (Parts 10-12): Testing, production examples, real-world cases

## Key Topics Covered

- W3C SSE specifications and EventSource API
- HTTP chunking mechanisms and streaming protocols
- OpenAI ChatGPT and Anthropic Claude SSE implementations
- Next.js App Router and Pages Router SSE patterns
- Vercel AI SDK integration
- Browser-side incomplete data handling
- Error handling and reconnection strategies
- Performance optimization techniques
- Security considerations (CORS, authentication, XSS/CSRF)
- Testing strategies with Jest, React Testing Library, and Storybook
- Production-level implementation examples
- Real-world case studies from GitHub, Vercel, Shopify, LinkedIn

## Working with This Repository

When modifying or adding to the documentation:

- Maintain consistency with the existing Korean language style
- Follow the established file naming pattern (partN-topic-name.md)
- Update the main README.md table of contents when adding new sections
- Cross-reference between related parts using relative markdown links
- Code examples should be practical and production-ready
