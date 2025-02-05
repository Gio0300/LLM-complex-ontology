# Challenges of using LLMs for reasoning over complex ontologies

Seeing the ever-growing list of successful LLM applications I posited there would be a great use-case for LLMs to reason over complex semantic ontologies. I assumed LLMs could answer natural language queries an other stereotypical use-cases. SPARQL query generation and knowledge graph question answering immediately came to mind. Turns out that getting a large language model to reason over a complex semantic ontology is not the easiest thing in the world. I naively attempted the aforementioned exercise and my results were ... less than favorable. The following is a compendium of challenges I faced. Some of the challenges are common to LLM-based solutions, others are specific to ontologies and graphs. Where possible I will venture to offer ideas on how each challenge may be solved. Take my suggestions with a grain of salt as I was unable to bring this solution to life; may you triumph where I faltered.

For reference here is a short glossary of the most salient terms, just so we are all on the same page.
* Ontology - [Web Ontology Language](https://en.wikipedia.org/wiki/Web_Ontology_Language)
* RDF - [Resource Description Framework](https://www.w3.org/2001/sw/wiki/RDF)
* Turtle Notation - [RDF 1.1 Turtle](https://www.w3.org/TR/turtle/)

Common examples on the web use small palatable ontologies to illustrate their point. Real world ontologies are large and complex. In this article we will define a complex semantic ontology as an ontology the meets all the criteria below.
* 10 or more classes
* 20 or more object properties
* 20 or more data properties
* Well-defined class restrictions, e.g., domain, range, disjoins, cardinality, lists
* Class hierarchies
* Supporting annotations and comments to make it comprehensible. 

Ontologies represent real-world entities and our exploration of the subject would not be complete unless we pair the ontology with a graph of individuals represented by different classes linked together by object properties and decorated by data properties. This may all feel a bit overly complicated but that's the point. I want you, the reader, to understand that a real-world solution will be complex and demanding.

To illustrate the complexity of working with ontologies I've chosen to build an ontology defining a milk processing plant. I based my fictional milk processing plant based on the video titled [The basics of milk production](https://www.youtube.com/watch?v=7TMtA8Eh9uE). For reference I have included a partial ontology titled [milk-plant-ontology](/milk-plant-ontology.ttl) and a graph for a fictional milk processing plant named [MooCho Milk Dairy](/moocho-milk-dairy.ttl). Note that I only developed the ontology for the first few stages of the plant. Early experiments integrating with the LLM failed using a partial ontology so I felt it best to explore solutions before progressing. I would encourage you to review and play around with the ontology and graph before progressing into the list of challenges.

> In this article you will find extensive use of qualitative terms like complex, long, and difficult. Quantitative terms are conspicuously absent. The reason is two fold. With the seemingly unstoppable pace of AI innovation I expect any quantitative values provided would be outdated quickly and I wish the reader to focus on the intrinsic challenges and not some short lived technical limitation.

## Challenges and potential workarounds

### 1. Ontologies are difficult to create and maintain
Ontologies are difficult to create and maintain. Developing ontologies requires a deep understanding of the domain, the relationships, the restriction of the domain, and the ability to represent such complex information formally and comprehensibly. The difficulty of building an effective ontology cannot be understated. To work effectively with LLMs the ontology will need to be defined using formal logic to accurately represent the domain, what [Smith & Ceusters, 2010](https://pubmed.ncbi.nlm.nih.gov/21637730/) called 'ontology realism', but not so verbose as to make it unmanageably large, more on that later.

Anything but the most trivial illustrative example will quickly balloon into a complex web of relationships. Unfortunately there is no shortcut around this. The real world is complex and messy and describing it with any semblance of accuracy and formality is difficult and requires a methodical and meticulous approach. While there may not be any shortcuts there are ways to manage the complexity.

One way is to break the ontology in to multiple sub-ontologies that may be assembled into a complete body of work. This approach is well suited for domains that are naturally hierarchical. Another strategy is to partition the ontology into manageable chunks. This strategy is better suited for domains that have natural breaks or transition points, for example the different stages associated with processing milk.

### 2. Token limits 
Token limits are a reality of working with large language models and although token limits increase regularly they still present a very real challenge when working with ontologies. Complex ontologies are verbose even when using relatively compact notation like [Turtle Notation](https://www.w3.org/TR/turtle/).

If you are not an AI expert like me your first instinct may be to attempt to use embedding but I have bad news for you, embedding of ontologies has been tried and the results haven't been great ([Ontology Embedding: A Survey of Methods, Applications and Resources](https://arxiv.org/html/2406.10964v1#:~:text=Such%20ontology%20embedding%20methods%20have,consuming%20ontologies%20in%20other%20systems.)). I suspect the reason embedding has been difficult to realize is because embeddings are a generalization while an ontology is a formal definition. Generalizing (embedding) a formal definition (ontology) leads to such a departure from the concept as to make the embedding ineffective because when working with ontologies specificity is key.

Before I present some techniques for dealing with token limits I would like to mention another challenge, namely that providing the ontology to the LLM is only part of the prompt. A typical use case will also include a system prompt, a natural language query, and perhaps some chat history. A slightly more sophisticated use case will also include some data in the form of a graph of individuals and their respective properties. I am sure you can image how quickly all this input can eat into our token limit.

There are a few of techniques that will aid in optimizing the ontology for use on an LLM. These techniques are complimentary to each other, you may choose to apply none, some, or all.

As mention above, you may partition the ontology into logical portions. Perhaps by organizing the ontology into a hierarchical structure if the domain allows, or finding natural partitions in the domain. Partition allows an orchestration layer to select the appropriate partition to submit in the prompt instead of submitting the entire ontology.

Although turtle notation is concise, there is still room for improvement. Minification is a way to cut down the number of tokens. If strict precision is not required namespaces and URIs may be shorten to serve only for differentiation or they may be removed altogether. Comments may be removed. As of this writing I am not aware of any tools to minify turtle notation. If you know of any please share it in the comments. 

In the same vain as minification you may want to leverage compaction. For our purposes I am defining compaction as the eviction of less useful sections of the ontology, redaction of things like multilingual descriptors or outright removal of entire portions that do not apply to your use-case.
> The distinction between minification and compaction is that minification does not change the logical content and only does away with formalities. Minification is conceptually lossless. Compaction on the other hand may remove entire sections of the ontology making it conceptually lossy.  

The last strategy I will introduce for dealing with token limits is implementing multi-resolution ontologies. If you haven't heard about multi-resolution ontologies before it is because I just made it up. The concept is simple, first we generate multiple version of the ontology with increasing level of detail and formality. The version of the ontology with the lowest resolution may be used initially to ask the LLM to identify the major area(s) of interest. We then select a higher resolution ontology of the area leveraging the partitioning strategies previously discussed. The idea is that the LLM does not have to reason about a large and highly detail ontology. We can use the LLM to direct the flow and scope of the analysis. The intent behind multi-resolution ontologies may be achieved with embedding of the ontology but consider that embedding the entire ontology may not yield the expected results and you that may want to apply partitioning, minification, and compaction before hand.

### 3. Complex prompt chaining
Prompt chaining in real world solutions is typically more complicated than what the typical RAG tutorial would have you believe but such is the nature of tutorials. Prompt chaining for ontologies is a bit more complex still, luckily, not much more but enough that it bares closer examination.

In a typical RAG scenario the orchestration layer must synthesize a SQL query from a natural language query and a schema, query the data store, and finally submit the data and the NLQ to the LLM. The LLM doesn't require the schema to be resubmitted because the LLM can infer the schema from the incidental metadata like table names, column names, and the context organically present in the prompt.

It gets a bit tricker for ontologies, its more difficult to evaluate domain rules than to infer a table schema. Domain rules can be expressed in a multitude of ways using OWL semantics. There are hierarchical relationships, inheritance, cardinality and quantifier restrictions, there are transitive and asymmetric object property characteristics, disjoints and more. The point is that inferring the relationship between entities based solely on the incidental metadata is not enough. To generate an dependable response the LLM needs not only reason about the data but also reason about the domain rules and evaluate those domain rules.

This type reasoning can be cognitively challenging for domain experts that possess an intuitive understanding of the domain rules and the fact is that LLMs can not reason that deeply over such a complex context without a lot of help. So what do we do, we break up the task into manageable pieces that results in a chatty and complicated prompt chain that will make anyone's head spin.

There are ways to manage prompt chaining to avoid exceedingly long chains. One could employ highly optimized intent detection paired with optimized function and data source selections. A well optimized solution employing the strategies listed above should keep the prompt chains to a manageable length or at the very least comprehensible and testable. 

### 4. Custom tooling & complex architectures
There are no readily available tools to perform minification, compaction, and partitioning; that means we would need to roll our own. These tools would need to be implemented as automatable components to support testing and maintenance for a production grade solution.

If one were to adopt the strategies listed in this article the total number of components for the final solution would be notable. For reference the [baseline architecture for an Azure OpenIA chat](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-openai-e2e-chat#architecture) may look like the diagram below. Now imagine the architecture necessary for the aforementioned custom tools.
![Baseline OpenAI end-to-end chat reference architecture](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/_images/openai-end-to-end-aml-deployment.svg#lightbox)

        
### Conclusion
So far we have discussed the challenges of getting data in and out of the LLM but how good are LLM are reasoning over ontologies? Assuming we did everything right, what can we expect from the LLM? Unfortunately not much. According to [Evaluating the effectiveness of prompt engineering for knowledge graph question answering](https://www.frontiersin.org/journals/artificial-intelligence/articles/10.3389/frai.2024.1454258/full) the best Spider4SPARQL score observed was 51%. The takeaway from this article and formal research papers ([here](https://arxiv.org/pdf/2410.01401), and [here](https://arxiv.org/pdf/2210.08008)) is this ... LLMs are not great at factual knowledge graph question answering. At the incredible pace of AI innovation it is not hard to imagine that LLMs will be able to reason over ontologies in the not so distant future.

### P.S.
I found the [Protégé](https://protege.stanford.edu/) extremely helpful for authoring and visualizing ontologies.
