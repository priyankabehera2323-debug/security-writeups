# TryHackMe: The Concierge Knows Too Much

**Category:** AI Security | Prompt Injection | Social Engineering
**Room link:** https://tryhackme.com/room/hh-theconciergeknows-2d7eb4d9

## Skills Demonstrated
- Prompt injection technique execution
- Analyzing LLM trust boundaries and context handling
- Social engineering reconnaissance against an AI system
- Identifying system prompt / hidden instruction leakage risks

## Scenario Overview
The room simulates an AI-powered "concierge" assistant that personalizes its responses based on user context and perceived identity. The objective was to manipulate the assistant's trust assumptions to extract information it was designed to keep restricted.

## Approach

**1. Reconnaissance**
Interacted with the assistant to understand how it personalized responses — testing what contextual signals (claimed identity, phrasing, framing) changed its behavior.

**2. Identifying the trust exploit**
Found that the assistant made trust decisions based on social engineering cues rather than verified authentication — a common weakness in LLM-integrated systems that infer identity/authority from conversational context alone.

**3. Prompt manipulation**
Used targeted prompt engineering to shift the assistant's framing of who I was and what it was "safe" to share, working around its intended restrictions without directly attacking its underlying model.

**4. Extraction**
Once the trust boundary was bypassed, the assistant disclosed information it was designed to protect — demonstrating how a system prompt or hidden instruction can leak when identity/context isn't properly verified.

## Key Takeaways
- LLM-integrated applications often conflate *conversational plausibility* with *verified trust* — a critical design flaw.
- Prompt injection isn't always about breaking the model itself; often it's about exploiting how the surrounding application interprets and trusts user input.
- Protecting system prompts and hidden instructions is not sufficient on its own — the trust/authentication layer around the LLM matters just as much.

## Defensive Recommendations
- Never rely on conversational context alone to establish user identity or authorization.
- Enforce authentication and authorization at the application layer, independent of the LLM's own reasoning.
- Treat system prompts as sensitive but *not secret* — assume they could leak, and design restrictions that don't depend on the model refusing to disclose them.
- Apply output filtering/validation on LLM responses before they reach the user, especially for identity-sensitive or restricted data.

## Reflection
This was my first hands-on exposure to prompt injection in a socially-engineered context rather than a purely technical exploit. It reinforced that AI security sits at the intersection of classic security thinking (trust boundaries, authentication) and new attack surfaces (natural language manipulation) — a space I want to keep exploring.
