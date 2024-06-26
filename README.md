# The Graph Maker

A Python tool that can convert any text into a graph of knowedge given an ontology.



## What is a knowledge graph?

A knowledge graph, also known as a semantic network, represents a network of real-world entities—i.e. objects, events, situations, or concepts—and illustrates the relationship between them. This information is usually stored in a graph database and visualized as a graph structure, prompting the term knowledge “graph.”

Source: https://www.ibm.com/topics/knowledge-graph

## Why Graph?

KG can be used for a multitude of purposes. We can run graph algorithms and calculate centralities of any node, to understand how important a concept (node) is to this body of work. We can calculate communities to bunch the concepts together to better analyse the text. We can understand the connectedness between seemingly disconnected concepts.

The best of all, we can achieve **Graph Retrieval Augmented Generation (GRAG)** and chat with our text in a much more profound way using Graph as a retriever. This is a new and improved version of **Retrieval Augmented Generation (RAG)** where we use a vectory db as a retriever to chat with our documents.

---

## This project

This is a python library that can create knowledge graphs out of any text using a given ontology. The library creates the graph fairly consistently with a good resilience to the faulty response generated by the LLM.

Here are the steps to create the knowledge graph.

To set up this project, [you will need Poetry](https://python-poetry.org/docs/configuration/)

```zsh
# Create a local environment
$ poetry config --local virtualenvs.in-project true
# Install dependencies.
$ poetry install
```

### 1. Define the Ontology of your Graph

The library understands the following schema for the Ontology. Behind the scene, ontology is a pydantic model.

```python
ontology = Ontology(
    # labels of the entities to be extracted. Can be a string or an object, like the following.
    labels=[
        {"Person": "Person name without any adjectives, Remember a person may be references by their name or using a pronoun"},
        {"Object": "Do not add the definite article 'the' in the object name"},
        {"Event": "Event event involving multiple people. Do not include qualifiers or verbs like gives, leaves, works etc."},
        "Place",
        "Document",
        "Organisation",
        "Action",
        {"Miscellanous": "Any important concept can not be categorised with any other given label"},
    ],
    # Relationships that are important for your application.
    # These are more like instructions for the LLM to nudge it to focus on specific relationships.
    # There is no guarentee that only these relationships will be extracted, but some models do a good job overall at sticking to these relations.
    relationships=[
        "Relation between any pair of Entities",
        ],
)
```

I have tuned the prompts to yield results that are consistent with the given ontology.
I think it does a pretty good job at it. However, it is still not 100% accurate. The accuracy depends on the model we choose to generate the graph, the application, ontology, and the quality of the data.

### 2. Split the text into chunks.

We can use as large a corpus of text as we want to create large knowledge graphs. However, LLMs have a finite context window right now. So we need to chunk the text appropriately and create the graph one chunk at a time. The chunk size that we should use depends on the model context window. The prompts that are used in this project eat up around 500 tokens. The rest of the context can be divided into input text and output graph. In my experience, 800 to 1200 token chunks are well suited.

### 3. Convert these chunks into Documents.

Documents is a pydantic model with the following schema

```python
## Pydantic document model
class Document(BaseModel):
    text: str
    metadata: dict
```

The metadata we add to the document here is tagged to every relation that is extracted out of the document.
We can add the context of the relation, for example the page number, chapter, the name of the article, etc. into the metadata. More often than not, Each node pairs have multiple relation with each other across multiple documents. The metadata helps contextualise these relationships.

### 4. Run the Graph Maker.

The Graph Maker directly takes a list of documents and iterates over each of them to create one subgraph per document. The final output is the complete graph of all the documents.

Here is the simple example code

```python
from graph_maker import GraphMaker, Ontology, GroqClient
from graph_maker import Document

## Select a groq supported model
## model = "mixtral-8x7b-32768"
model ="llama3-8b-8192"
## model = "llama3-70b-8192"
## model="gemma-7b-it" ## This is probably the fastest of all models, though a tad inaccurate.

llm = GroqClient(model=model, temperature=0.1, top_p=0.5)
graph_maker = GraphMaker(ontology=ontology, llm_client=llm, verbose=False)

## create a graph out of a list of Documents.
graph = graph_maker.from_documents(
    list(docs),
    delay_s_between=10 ## delay_s_between because otherwise groq api maxes out pretty fast.
    )
## result -> a list of Edges.
print("Total number of Edges", len(graph))
## 1503
```

The output is the final graph as a list of edges, where every edge is a pydantic model like the following.

```python
class Node(BaseModel):
    label: str
    name: str

class Edge(BaseModel):
    node_1: Node
    node_2: Node
    relationship: str
    metadata: dict = {}
    order: Union[int, None] = None
```

The Graph Makers runs each document through the model and parses the response into graph edges.
I have tuned the prompts to a point where it is quite fault tolerant by now. Most JSON failures are automatically corrected. In case the JSON response fails to parse, it also tries to manually split the JSON string into multiple strings of edges and then tries to parse each one of them separately.

### 5. Save to Neo4j (optional step)

We can save the model to Neo4j either to create an RAG application, run Network algorithms, or maybe just visualise the graph using the Bloom

```python
from graph_maker import Neo4jGraphModel

create_indices = False
neo4j_graph = Neo4jGraphModel(edges=graph, create_indices=create_indices)

neo4j_graph.save()

```

Each edge of the graph is saved to database in a transaction. If you are running this code for the first time, then set the `create_indices` to true. This prepares the database by setting up the uniqueness constraints on the nodes.
