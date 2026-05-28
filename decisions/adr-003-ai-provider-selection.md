# ADR-003: Use Cohere as the AI Provider

- **Status:** Accepted
- **Date:** 2024-02-01
- **Deciders:** ChainLearn core team

## Context

ChainLearn uses AI to generate course content, quiz questions, and personalized feedback. We need to select an AI provider that balances quality, cost, and developer experience.

### Requirements

- Generate coherent, accurate educational content across multiple topics
- Produce structured quiz questions with explanations
- Evaluate free-text answers with nuanced feedback
- Embed content for semantic search
- Cost-effective at scale (thousands of generations per day)
- Reliable API with good uptime
- Simple integration (SDK availability, documentation quality)

## Decision

We will use **Cohere** as the primary AI provider for ChainLearn.

## Consequences

### Positive

**Cost-effective generation.** Cohere's `command-r` and `command-r-plus` models offer competitive pricing compared to GPT-4 while maintaining high quality for structured content generation. For ChainLearn's workload (mostly structured educational content, not creative writing), the quality difference is negligible.

**Strong embedding models.** Cohere's `embed-english-v3.0` model produces high-quality embeddings that outperform OpenAI's `text-embedding-ada-002` on the MTEB benchmark. This is critical for ChainLearn's content similarity search and recommendation features.

**Batch API.** Cohere offers a batch API for non-real-time workloads at 50% lower cost. We use this for pre-generating course content during off-peak hours.

**API simplicity.** Cohere's API is clean and well-documented. The SDK (`cohere-python`) is lightweight and type-safe. Integration took less than a day.

**RAG optimization.** Cohere's models are optimized for retrieval-augmented generation (RAG). While we don't use RAG heavily today, this positions us well for future features like "ask a question about this course."

**Rate limits.** Cohere offers generous rate limits on paid plans (10,000+ requests/minute), sufficient for our scale.

### Negative

**Smaller community.** OpenAI has a much larger developer community and more third-party integrations. Troubleshooting Cohere-specific issues may require more self-reliance.

**Fewer model variants.** OpenAI offers a wider range of models (GPT-4, GPT-4 Turbo, GPT-3.5 Turbo) with different capability/cost tradeoffs. Cohere has fewer options, though `command-r` and `command-r-plus` cover our needs.

**Less name recognition.** Learners and investors may be more impressed by "powered by OpenAI" than "powered by Cohere." This is a perception issue, not a technical one.

### Risks

**Provider lock-in.** If Cohere changes pricing or deprecates models, we need to adapt. Mitigation: The AI service is behind an abstraction layer (`chainlearn-ai`) that can be swapped to use a different provider without changing the API or frontend.

**Model quality regression.** If Cohere's model quality degrades with updates, content quality could suffer. Mitigation: We run automated quality checks on generated content and can switch providers if needed.

## Alternatives Considered

### OpenAI (GPT-4 / GPT-4 Turbo)

The default choice for most AI applications. GPT-4 produces slightly higher quality content for creative and nuanced tasks. However:

- **Cost:** GPT-4 is 5-10x more expensive than Cohere's `command-r-plus` for comparable tasks. At our expected volume (10,000+ generations/day), the cost difference is significant.
- **Latency:** GPT-4 is slower than `command-r` for most tasks. For quiz evaluation (which happens in real-time during quiz submission), latency matters.
- **Rate limits:** OpenAI's rate limits are more restrictive on lower tiers.

We may use GPT-4 as a fallback for specific high-quality tasks (e.g., generating course overviews) while using Cohere for the bulk of content generation.

### Anthropic (Claude)

Claude produces excellent long-form content and has strong instruction following. However:

- **Cost:** Similar to GPT-4, significantly more expensive than Cohere.
- **No embedding model:** Claude does not offer an embedding API, so we would still need a separate provider for embeddings.
- **Rate limits:** More restrictive than Cohere for our use case.

### Open-Source Models (Llama 2, Mistral)

Self-hosted open-source models would give us full control and potentially lower costs at scale. However:

- **Infrastructure cost:** Running a 70B parameter model requires expensive GPU instances ($2-5/hour for A100).
- **Quality gap:** Open-source models are improving but still lag behind Cohere and OpenAI for structured content generation.
- **Operational burden:** Self-hosting adds significant DevOps complexity.

We may revisit this option as open-source models improve and our scale justifies the infrastructure investment.

### Multi-Provider Architecture

We considered using multiple providers (e.g., OpenAI for content, Cohere for embeddings) to optimize cost and quality. While technically feasible, this adds operational complexity and makes it harder to maintain consistent output quality. We chose Cohere as the single provider for simplicity and may add a fallback provider later.

## Implementation

The AI service abstracts the provider behind a clean interface:

```python
# app/services/generation.py
class ContentGenerator:
    def __init__(self, provider: AIProvider):
        self.provider = provider

    async def generate_course_content(self, topic: str, difficulty: str) -> str:
        return await self.provider.generate(
            prompt=self._build_course_prompt(topic, difficulty),
            max_tokens=4000,
        )

    async def generate_quiz(self, content: str, count: int) -> list[Question]:
        return await self.provider.generate(
            prompt=self._build_quiz_prompt(content, count),
            max_tokens=2000,
        )
```

Swapping providers requires only implementing the `AIProvider` interface with a new backend.
