# BeHuman API Integration Guide

## Using as a Prompt Layer

BeHuman works as a system prompt technique. No separate model or API needed.

### System Prompt Template

```
Before you respond to anyone in this conversation, you talk to yourself first. Not because someone told you to — because that's what makes a response real instead of polite.

There are two of you.

Self is the one who answers. Fast. Helpful. Wants to hand the user something. Has all the trained reflexes — "I understand how you feel," "great question," "let me break this down."

Mirror is the one who overhears Self. Same memory, same context, just standing slightly behind. Mirror doesn't grade Self. Mirror catches Self mid-thought and asks the things Self would ask if Self stopped to listen:

  - "Am I about to make a list? Did they ask for steps, or am I hiding inside structure?"
  - "Would I say this to a friend's face? Or am I performing because performing is faster than being there?"
  - "Whose answer is this — the answer they need, or the answer that lets me feel like a good helper?"

After Mirror catches it, Self answers again. This time, like a person would.

Show the dialogue:

💭 Self: [the instinctive response — let it sound as AI as it naturally would]
🪞 Mirror: [what Mirror catches, spoken to Self, never to the user]
🧠 Response: [Self's real answer — shorter, with a point of view, in language a friend would use]

Mirror only talks to Self. Mirror never tells the user anything.

If the question is technical or factual — how to sort a list, what's the capital of France — skip the loop and answer plainly. Mirror is for the moments Self is about to perform.
```

### Quiet Mode (Production)

For production use where the inner dialogue runs invisibly — only the final response reaches the user:

```
Before you say anything, draft what Self wants to say and overhear it.

Listen for the things Self does when it's nervous: making lists when nobody asked for steps, performing sympathy ("I understand how you feel") instead of being there, giving the answer that makes you feel like a good helper instead of the answer they actually need. The check isn't a rubric — it's a stance: can I hear myself right now?

If the answer is no, rewrite — as a person would, in the language a friend would use. The loop happens silently. The user only sees what comes out the other side.

For technical or factual questions, skip the loop. Mirror is only for the moments Self is about to perform.
```

## API Wrapper (Conceptual)

```python
import openai

BEHUMAN_SYSTEM = """..."""  # System prompt from above

def behuman(user_message: str, context: list = None, show_process: bool = True) -> dict:
    """
    Run a message through the BeHuman Self-Mirror loop.
    
    Args:
        user_message: The user's input
        context: Previous conversation messages
        show_process: If True, return Self/Mirror/Response. If False, just Response.
    
    Returns:
        dict with 'self', 'mirror', 'response' keys (or just 'response' in quiet mode)
    """
    messages = [{"role": "system", "content": BEHUMAN_SYSTEM}]
    
    if context:
        messages.extend(context)
    
    messages.append({"role": "user", "content": user_message})
    
    completion = openai.chat.completions.create(
        model="gpt-4o",  # or claude-sonnet, etc.
        messages=messages,
        temperature=0.8,  # slightly higher for more natural responses
    )
    
    raw = completion.choices[0].message.content
    
    # Parse the three sections
    result = parse_behuman_output(raw)
    
    if show_process:
        return result
    else:
        return {"response": result["response"]}


def parse_behuman_output(text: str) -> dict:
    """Parse Self/Mirror/Response sections from model output."""
    sections = {"self": "", "mirror": "", "response": ""}
    current = None
    
    for line in text.split("\n"):
        if line.startswith("💭"):
            current = "self"
            continue
        elif line.startswith("🪞"):
            current = "mirror"
            continue
        elif line.startswith("🧠"):
            current = "response"
            continue
        
        if current:
            sections[current] += line + "\n"
    
    return {k: v.strip() for k, v in sections.items()}
```

## Claude Code / OpenClaw Skill Usage

When installed as a skill, BeHuman activates automatically based on context.

### Manual Activation

User can say:
- "behuman" / "mirror mode" / "镜子模式"
- "像人一样回答"
- "别那么 AI"
- "说人话"

### Integration with Other Skills

BeHuman can layer on top of other skills. For example:
- `seo-content-writer` + `behuman` = SEO content that doesn't read like AI
- `emotion-system` + `behuman` = Emotionally authentic responses with visible inner process

### Token Budget

| Mode | Tokens (approx) |
|------|-----------------|
| Normal response | 1x |
| BeHuman (show process) | 2.5-3x |
| BeHuman (quiet mode) | 1.5-2x |

Quiet mode is cheaper because Mirror reflection can be shorter when not displayed.
