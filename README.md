# Monarch Agent Tutorial

We will be walking through the process of creating an ontology agent.

## Pre-requisites

1. Some knowledge of Python
2. `uv` installed
3. An IDE or text editor (cursor recommended)
4. An OpenAI API key, with `OPENAI_API_KEY` set

## Initializing the project

### Install uv

Follow the instructions here: [https://docs.astral.sh/uv/getting-started/installation/](https://docs.astral.sh/uv/getting-started/installation/)

On a mac:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Create the repo

```bash
uv init --package agent-tutorial
```

### Add dependencies

```
cd agent-tutorial
```

(or open the terminal in your IDE)

```
uv add pydantic-ai
```

## PART 1: Create a basic agent

### Create `hello_world.py`

Create this in [src/agent_tutorial/hello_world.py](src/agent_tutorial/hello_world.py)

Copy the example from here:

https://ai.pydantic.dev/#hello-world-example

Adapt it to use `openai:gpt-4o` (unless you have a )

```python
from pydantic_ai import Agent

agent = Agent(  
    'openai:gpt-4o',
    system_prompt='Be concise, reply with one sentence.',  
)

result = agent.run_sync('Where does "hello world" come from?')  
print(result.data)
"""
The first known use of "hello, world" was in a 1974 textbook about the C programming language.
"""
```

### Run it

```
uv run src/agent_tutorial/hello_world.py 
```

__TROUBLESHOOTING__ 

* do you have `OPENAI_API_KEY` set?

## PART 2: Create an agent with tools

### Add OAK as a dependency

```
uv add oaklib
```

__TROUBLESHOOTING__

You may need to do this first:

```
uv add fastobo --binary fastobo==0.12.3
```

Then create a file [src/agent_tutorial/oak_agent.py](src/agent_tutorial/oak_agent.py)

We will make an agent, as before

```python
oak_agent = Agent(  
    'openai:gpt-4o',
    system_prompt="""
    You are an expert ontology curator. Use the ontologies at your disposal to
    answer the users questions.
    """,  
)```

But this time we will give an agent a *tool*

```python
@oak_agent.tool_plain
async def search_uberon(term: str) -> List[Tuple[str, str]]:
    """
    Search the UBERON ontology for a term.

    Note that search should take into account synonyms, but synonyms may be incomplete,
    so if you cannot find a concept of interest, try searching using related or synonymous
    terms.

    If you are searching for a composite term, try searching on the sub-terms to get a sense
    of the terminology used in the ontology.

    Args:
        term: The term to search for.

    Returns:
        A list of tuples, each containing an UBERON ID and a label.
    """
    adapter = get_adapter("ols:uberon")
    results = adapter.basic_search(term)
    labels = list(adapter.labels(results))
    print(f"## Query: {term} -> {labels}")
    return labels
```

We can test this with

```python
query = "What is the UBERON ID for the CNS?"
result = oak_agent.run_sync(query)
```

And run it:

```
uv run src/agent_tutorial/oak_agent.py 
```

## PART 3: Create an annotator agent

Then create a file [src/agent_tutorial/annotator_agent.py](src/agent_tutorial/annotator_agent.py)

We will create a bespoke data model for this agent:

```python
class TextAnnotation(BaseModel):
    """
    A text annotation is a span of text and the UBERON ID and label for the anatomical structure it mentions.
    Use `text` for the source text, and `uberon_id` and `uberon_label` for the UBERON ID and label of the anatomical structure in the ontology.
    """
    text: str
    uberon_id: Optional[str] = None
    uberon_label: Optional[str] = None

class TextAnnotationResult(BaseModel):
    annotations: List[TextAnnotation]
```

Now we'll create an agent:

```python
annotator_agent = Agent(  
    'claude-3-7-sonnet-latest',
    system_prompt="""
    Extract all uberon terms from the text. Return the as a list of annotations.
    Be sure to include all spans mentioning anatomical structures; if you cannot
    find a UBERON ID, then you should still return a TextAnnotation, just leave
    the uberon_id field empty.

    However, before giving up you should be sure to try different combinations of
    synonyms with the `search_uberon` tool.
    """,
    tools=[search_uberon],
    result_type=TextAnnotationResult,  
)
```

Note this reuses the tool from the previous example

And run it:

```
uv run src/agent_tutorial/annotator_agent.py 
```