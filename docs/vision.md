# Learning Design Approach

An absence of knowledge engineering to manage data science causes models to become biased and inflexible. This is known as [concept drift](https://en.wikipedia.org/wiki/Concept_drift). Products are theories of reality, and science already knows how to evolve theories. When support for new business domains need to be added to a product that includes learning, there needs to be an organized label structure to query the annotated training data through, otherwise models will calcify and become unable to adapt. Just like regular code debt caused by tightly coupled concerns.

There are many kinds of models that fit different problems and not everything needs to be encoded into any one particular type of classifier. Feature engineering is important of course, but shoehorning every blob of dimensionality into a single uniform structure for all types of predictions is asking for trouble. As with business logic, classification modules should be orthogonal, pluggable adapters that implement interfaces. We ask chefs about cooking and bakers about baking. They are both specialized at their respective (sub)domains. Even though they share common ingredients and tools, they operate under different rules using different jargon.

To manage the challenges of model bias and over fitting that stem from unstructured target labels and neglect of semantics, one general solution is to train multiple models. That is, one model for each domain or class of facts that is possible to maintain a high level of accuracy about. Then reasoning can be externalized by deciding which model has more authority in a given context, making classifiers more programmable.

Reyearn uses a forest of taxonomic trees, a directed graph. Each branch in a tree represents a nested set of labelled observations. These observations are leaves on a tree (at the very edge of the graph). This label tree idea is related to [Hierarchical Multilabel Classification](https://en.wikipedia.org/wiki/Multiclass_classification#Hierarchical_classification). It's programmed at the application level rather than being used to structure word embeddings inside statistical models (although that is an interesting technique on its own). The ensemble of models approach has been demonstrated by [MILABOT](https://arxiv.org/abs/1709.02349) for the Alexa Prize, [Microsoft LUIS](https://docs.microsoft.com/en-us/azure/cognitive-services/luis/luis-concept-prebuilt-model#prebuilt-domains) and others.

Making the ensemble of models configurable by data at runtime allows users to shape the behavior of the system, and the operational overhead of training and deploying very large models is reduced. It also allows us to mix vastly different types of models more easily, e.g. some can be CNNs, RNNs, some Bayesian, some expression parsers, rule matchers etc. Any function or third party endpoint can act as a classifier in principle.

## Other projects

There is so much activity in the area of open source ML that it is hard to keep up. This is a running list of similar projects doing things in different but complimentary ways.

- [Creme](https://github.com/creme-ml/creme): Solving the concept drift problem at a lower level. A library focussed on stream-based online learning, with great ML algorithm support. It might be a candidate to supply more classifiers out of the box in Reyearn if its code can be properly serialized to run on Dask.
- [Weights and Biases](https://docs.wandb.com/): A commercial product with full UI for experiment management. Stores model metadata such as accuracy reports and hyperparameters to enable multivariate testing to iterate model performance in a rational way. It also seems really nice for reporting and communications.
- [ConceptNet Numberbatch](https://github.com/commonsense/conceptnet-numberbatch): training general purpose language models by querying for training data through a semantic network. The semantic qualities are great for combatting model bias because training data can be balanced by its conceptual meanings ("priors" are inherently normalized). This is a problem faced by all training pipeline designers but the implementation isn't usually so elegant.
- [Grakn KGLIB](https://github.com/graknlabs/kglib): training models by querying for training data through arbitrary knowledge (property) graphs with deductive logic. Inference is backward chaining (finding facts given conclusions a.k.a. modus tolens). This is best for applications like drug discovery where you can start with a symptom (conclusion) and find the networks of affecting systems like enzymes or hormones, in order to better match drug actions to those systems. Reyearn needs to use forward chaining (finding conclusions given facts a.k.a. modus ponens) for finding solutions in relatively knowable spaces like black letter law. For example, each new event or piece of evidence in a fact pattern adds to the context of the scenario, allowing possible case outcomes to be predicted.

## Why?

We need to embrace the modern statistical methods (because they are uniquely capable of dealing with unseen observations compared to straight code and logic), but this is just the foundation layer of the overall system. It is the base layer on which the rest of the stack is built, so that these statistical (a.k.a. connectionist) methods can be leveraged by traditional symbolic methods. Why? It can unlock domains where previously only humans could be trusted to handle the mundane computation, in order to help them be more creative by freeing them up to focus on harder problems.

The rest of this document is about leveraging the incremental learning capabilities described above by integrating them with traditional expert systems. This is where the concept of "non-expert learning system" comes from. It is an inversion of the approaches which try to solve bias and drift inside the models themselves. Instead, Reyearn applies similar techniques at the application, orchestration and preprocessing levels rather than at the training and vector embedding levels. This permits systems to be more easily explainable, configurable, and programmable than their purely statistical cousins. It's not to compete with lower level optimizations but to complement them by broadening the spectrum of abstraction available to applications that might have stricter constraints on training data, expertise, and time.

# High-Level Application Vision

One of the inspirations for the motivation behind Reyearn is the [British Columbia Civil Resolution Tribunal](https://civilresolutionbc.ca) (BCCRT), which is an online dispute resolution service with real world jurisdiction for small claims. In particular, the service provides a "Solution Explorer" which allows people to interactively learn about possible solutions to their legal problem and how to proceed. Despite being somewhat conservative on UX, as a government project [developed by PwC](https://www.pwc.com/ca/en/industries/government-and-public-services/citizen-experience.html), the almost-conversational experience is a good example of an expert system UI metaphor. It validates that such systems are possible and successful even in one of the most conservative and challenging domains. The legal profession is notoriously slow to innovate (not always a bad thing), but combine that with the challenges faced by large government projects, and it must be appreciated for not failing miserably. For a deep dive [this is a very good talk](https://www.youtube.com/watch?v=1YWMgpueDIM) by Shannon Salter, the chair of the tribunal.

A deeper motivation is to get software into domains that are too mission critical to be left up to machine learning alone. The law is a great example of this, and it stands to benefit regular people, so it is the primary target use case. Any policy-oriented domain would be a good fit, and the system is designed in an orthogonal way so that any domain can be supported by plugging in different classifiers while still benefiting from the opinionated architecture and operations automation.

Some awareness of the pre-connectionist [history](https://lib.dr.iastate.edu/cgi/viewcontent.cgi?article=1087&context=rtd) [of AI](https://www.wikiwand.com/en/History_of_artificial_intelligence) from a multidisciplinary perpective would be useful, because the current popular understanding is often limited to machine learning and people seem to have forgotten about the old symbolic methods in the rush towards shiny new things.

## User Experience Rationale

This project is partly inspired by learnings from shipping conversational assistants in production, and first-hand experience supporting data science teams by developing tooling for their operational needs. The user personas being empathized with are data scientists, engineers, administrators and end users.

General purpose "bot" platforms generally do not take into account expert systems or knowledge graphs because they are targeting less specialized users (nothing wrong with that). The consequence of this simplification is that developers will write a lot of brittle if-then code to map label predictions to action handlers. If they are building a real world project that ends up trying to bring model training in-house, data scientists will inherit an amorphous blob of categories that cannot be untangled easily, let alone extending it to support new user-facing features. Even if they are able to re-organize the taxonomies of labels and update all the annotations, virtually all of the application code will need to be rewritten. Many product organizations have a poor engineering culture that is unable to throw away prototypes, so they attempt to build around this black hole of technical debt. This can be mitigated by organizing the labels, training an ensemble of models and using rule matching instead of if-thens.

Tracking state by slot filling without a dialog model is insufficient for conversational user experience. When users fall outside of a menu-like decision tree, they have to start again from scratch unless edge cases are handled by hardcoding to send them back down to a lower branch. As there are many edge cases in most domains, scaling a conversational experience requires keeping track of what has been said and what it means in context. This context must be richer than what a finite state machine can handle alone, because it must take into account all possible paths to support backtracking based on new input of previously unseen observations, opening up those latent paths to possible solutions. Administrator users (domain experts) want to be able to configure the behaviour, which is impossible to do without rules and facts being dynamic.

### Engineering Solutions

To handle dialog management, a topic stack can be used. The topic stack is a stack of frames where each frame represents a turn of dialog (one volley in an agent-to-agent exchange). There is no hard limit to the number of agents in principle, but for learning about a fact pattern from an end user, we only need two, the user for input and the application to find explanations that apply to the situation and trigger the next action. Human agents are not required, and running multiple communicating virtual agents would be an interesting area to explore. This state tracking is loosely inspired by [COLLAGEN](https://www.merl.com/publications/docs/TR97-21a.pdf) and [RavenClaw](https://www.microsoft.com/en-us/research/publication/ravenclaw-dialog-management-using-hierarchical-task-decomposition-expectation-agenda/). Although it implements a dialectical approach, it doesn't have to rely on natural language or a conversational interface. In theory it could talk to itself.

To walk through the knowledge graph as new facts come in from the user, Reyearn needs an inference engine. Traditionally this has meant first-order logic, but there's no reason why inference needs to be so narrowly defined. The dialog system can be implemented by calling multiple classifier backends, where the manager process asks one or many classifiers what they predict a new fact should mean. Then, based on configuration, weights or other rules, recommends a solution by pushing a new frame onto the topic stack that describes the decision and triggers an action. One of the classifier backend modules is inspired by [CLIPS](http://www.clipsrules.net/AboutCLIPS.html), and is used for finding explanations that might apply to the user's situation by comparing their facts to the rules. The other module is inspired by the cases module of [SHYSTER](http://users.cecs.anu.edu.au/~James.Popple/shyster/) and is used for finding explanations by analogy represented as a ranking of structural distances to canonical explanations (via k-NN). This component is not strictly required to navigate basic use cases, since common solutions might not rely on references to a large variety of past solutions, and are more likely to be decidable by rules alone.

An interactive dashboard can be built for users to explore, with the ability to iterate on it by editing or adding facts and developing the details of the situation. They might want to iterate on the rules, weights and authoritative explanations (e.g. past decisions) at the same time. The dashboard should be a separate project.

### Science Background

Decision making must be explainable if we are to trust it. Without some kind of auditable rationale, which machine learning cannot provide on its own, we will have a hard time adopting software systems for critical domains like the law. Domains like medical diagnosis and [space shuttle telemetry](http://www.aaai.org/Papers/IAAI/1989/IAAI89-007.pdf), for example, have been partially automated by expert systems. These are not limited to statistical methods like today's deep learning marvels, they also use rules and logic to recognize patterns. However, without being able to analogize and approximate the fuzzier aspects of reality like the new approaches, these expert systems could never reach general intelligence, the original pipe dream. They were before their time due to limits in computing power and they were mistrusted by a deep cultural skepticism of their potential to rival human decision making. Both of these hurdles have been significantly reduced today.

What we learned from this most recent, [second wave of investment in AI](https://www.darpa.mil/about-us/darpa-perspective-on-ai) is that current computing power is enough to make statistical methods practical. What the previous, first wave needed most was machine learning to handle unknowns from outside their monolithic rule-based expert systems. The problem with the new second wave (deep learning all the things), is that we have thrown the baby out with the bathwater. Expert systems continue to be used in critical domains, and should not have been forgotten in favor of heaps of if-then spaghetti sitting on top of neural nets. The next, third wave, is to build hybrid systems that combine rules with statistical inference, or, deduction with induction. To synthesize a new approach out of the symbolic and connectionist approaches.

This ratcheting-up of deductive and inductive inference is a pragmatic idea called [abductive reasoning](https://plato.stanford.edu/entries/abduction/#DedIndAbd). It's a model of dialectic, which makes up one half of discourse (the other being rhetoric). Both parts of discourse make use of the same underlying symbols, but rhetoric is the art of making persuasive arguments based on examples and evidence.

We might say inductive things like "my cat Fluffball will meow at me loudly for food tomorrow morning" without any logical proof, simply based on having observed it every day in the past. However, [induction can't stand up to all probabilities](https://stanford.library.sydney.edu.au/archives/sum2016/entries/induction-problem/), even though it may seem to work often enough. Even if the sample sizes are large and randomized, statistical generalization (high quality induction) doesn't actually actually point to any conclusions or prove causal links, it just helps generate new queries by providing more precise terms.

We might also say [deductive](https://www.wikiwand.com/en/Modus_ponens#/Explanation) things like "all cats are cute and Fluffball is a cat, therefore Fluffball is cute", which works great for well-formalized domains and closed systems. Thankfully, proving facts is tedious and "beyond a reasonable doubt" is [all that is required](https://en.wikipedia.org/wiki/Rhetoric_of_science) in practice. Also, [logical positivism still can't capture reality](https://en.wikipedia.org/wiki/Logical_positivism#Logicism), even though there's no alternative that computers can understand yet.

Neither deduction alone or induction alone would seem to be reliable for explaining real situations. What happens when we learn there are multiple subtypes of cats and cuteness, or need to support birds that are only sometimes cute? What if all cats are not cute? Is loud meowing _ever_ indicative of cuteness? Let's say "sometimes", based on a large set of training data. Am I even _present_ to feed Fluffball in the morning? Not if I'm out of town, which might be deduced from my calendar. This repeated application of inference by modus ponens (forward chaining deductive logic) or by statistical, inductive generalization, can lead to the best hypothetical explanations given the facts. If the domain is about solutions to problems, then an explanation might be a solution to the user's problem.

We can use a flavor of the scientific method that [has worked forever to make discoveries](http://philsci-archive.pitt.edu/15321/1/Abduction.pdf) and also to mete out various [forms of justice](https://plato.stanford.edu/entries/legal-reas-prec/). In legal reasoning, there doesn't seem to be a one-size-fits-all logical framework (although positivism is popular). Reyearn tries to be agnostic about jurisprudence. It [might not be important](http://users.cecs.anu.edu.au/~James.Popple/shyster/book/) (chapter 1.2) to legal expert systems.

Statistics and logic are both used in a rhetorical way. Assuming we know about the implementation of a given classifier, the reason why we trust its answers can be seen as an appeal to its authority on account of its having seen a lot of training data in the past and our assent to its map of annotations. It has experience, like an expert. We could even call it [sophistry](https://en.wikipedia.org/wiki/Rhetoric#Sophists) that these things only appeal to probability. Whether a justification is an analogy to a canonical example, or a correspondence to a majority of past examples, proof isn't necessary to be useful, only agreement that it's good enough in the circumstances, which may include rules that take precedence over the probabilities.

Dialectical inferences are made about legislation to compute the ideal state of a situation, and rhetorical analogies are made about case law where previous decisions have explained similar situations. Together they form a discourse. Both logic and probability are held in balance. Judges apparently make decisions on a "preponderance" of the evidence according to rules, not on their subjective intuition or by formal maxims. Although it is not always fair, the brute fact that the law continues to exist in society suggests that it's a living, intelligent system that humans have historically placed some amount of trust in, presumably due to having no better alternative. The law attempts to [explain its predictions](https://www.wikiwand.com/en/Judicial_opinion) (i.e. decisions), permitting us to accept or reject its explanations (but maybe not their consequences), and to make improvements by correcting biases and gathering new evidence. Thus it would seem that a legal system would be a decent metaphor for any intelligent system that deals with social facts. What we want from software is what we have already (mostly) gotten from institutions that already transact the messy business of social interaction. This legal metaphor for intelligence is also convenient from a research perspective, as so much has been written about the theoretical applications of logic and rhetoric to the law that it's a virtually infinite wellspring of ideas from which to draw inspiration for features and conceptual modeling. That and the fact that the law underlies virtually all social domains that stand to benefit the most from software-assisted research and decision making.

### Architecture Notes

There are a few major moving parts.

- **Knowledge graph:** Postgres LTREE data type for organizing annotation labels into an "ontology" (not formally grounded, but an unbounded forest to query the training data by)
- **Triple store:** straight ahead encoding of subject-object-predicate triples in Postgres, where the entities/facts are observations
- **Experiment management:**
  - Model test results stored together in experiment tracking table
  - Integration with CI/CD for automating go-no-go checklists to drive deployment
  - Long term: Postgres schema-based branching and merging based on optimistic locking and versioning à la Elasticsearch and ActiveRecord (has been prototyped in a separate project but never integrated)
- **Inference engine:** querying the main decision trees at runtime can become a performance bottleneck, so the Rete algorithm is the best fit. For this, something descended from CLIPS like the Experta Python library can be useful.
- **Classifiers:** for case law, K Nearest Neighbour (k-NN) is the traditional technique from the literature. However, custom neural nets may be more appropriate. Not planned for the initial phase of development, as it is only needed for harder cases that rely on past court decisions.
- **Annotation:** For UI, SpaCy Prodigy or other online learning thing to avoid low value repetition and increase coverage of unique examples. Annotation event hooks call ingestion service to insert the label selections into Postgres range types which can have relational constraints, and enable overlapping annotations (for multiple entity types that intersect the same character ranges).

## Update to leverage LLMs in 2024 and beyond

Since the design of the Reyearn architecture as an "abductive reasoning machine" for building an "Explainable AI", LLMs have largely replaced custom classifiers for many tasks in industry.

At this point, most practitioners have built prototypes that wrap one of the largre commercail LLM vendors, experimented with [Retrieval Augmented Generation (RAG)](https://en.wikipedia.org/wiki/Prompt_engineering#Retrieval-augmented_generation), and had an opportunity to observe the limitations of these models and wrapper approaches.

For example, I've done some experimental work to extend the [Supabase Doc Search Starter](https://github.com/supabase-community/nextjs-openai-doc-search) project for the [legal domain](https://github.com/chartpath/reso-legal).

There is a great amount of coverage of the limitations of LLMs in the literature, but what stood out to me as a practitioner is that the reliability of their output depends heavily on the quality and completeness of the prompting. Just like tradtitional intent classification will not be reliable if the labels being predicted are not well-organized, LLMs are not reliable if the prompts are not well-organized.

What I discovered is that all the same limitations of traditional statistical models and custom neural nets with respect to reasoning and planning are also true of LLMs.

There is a recent groundbreaking paper which demonstrates a direct throughline from the design of the Reyearn architecture into a Generative AI world.

### LLMs Can't Plan, But Can Help Planning in LLM-Modulo Frameworks

In this [paper by Subbarao Kambhampati et al.](https://arxiv.org/abs/2402.01817), it is clear that LLMs are just as incapable of reasoning or planning as mast machine learning models.

This is not a reason to avoid using ML, just as the inability of logical rules to describe the whole world is not a reason to avoid using them.

Here are the main takeaways from the paper that will help to articulate he throughline from what a neuro-symbolic system looked like in 2020 to what it looks like in 2024. The most significant observation is that it's not different at all.

The primary difference is that LLMs can be swapped with custom classifiers to save a lot of development time, but otherwise the Reyearn architecture remains the same.

#### Takeways

- LLMs can't, by themselves, do planning or self-verification (which is a form of reasoning).
- These models are just n-gram approximators, just like traditional search engines.
- Fine-tuning does not affect the planning performance of the model.
- Performance is even worse when the model is fine-tuned on a specific task and the names of obejcts or actions are obfuscated.
- Because these models can't verify their plans, they also cannot self-critique.
- Even with multi-shot prompting (chain of thought), the model cannot verify its own plans.
- The original hope that models could do plan verification came from computational complexity theory, in which verification is less complex than generating a plan.
- However, it turns out that the initial "generation" step is not equivalent to complex plan generation because it is only approximate retrieval of existing knowledge, which means that verification through the same retrieval mechanism has equal complexity.
- Direct LLM self-critique _just doesn't work_.
- That said, approximate retrieval and extraction of knowledge from LLMs _can_ be used to help with planning in a neuro-symbolic system.
- By incorporating human soundness critics, plans generated with their feedback can be useful for sythetic generation of fine-tuning data which would extend self-critique retrieval coverage linearly.
- The soundness of the framework is _derived_ from the soundness of the critics. This can be seen as a limitation, but it is also a strength because it is a direct throughline from the human-in-the-loop design of the Reyearn architecture.
- Fine-tuning _without_ domain-grounded prompting _won't work_.
- Humans are only required once per domain and once per problem. After that, the critic role can be simulated programmatically.

### Implications

Here is a rough sketch of how the Reyearn architecture is be applied to leverage LLMs.

- Use [GraphRAG](https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/) plus rules for planning and reasoning.
  - Using a representation of the ontological (taxonomic and logical) context around the initial raw examples from vector search or other forms of retrieval.
    - For example, entities in one category obey rules with respect to things in another cartegory, so their relationship can be "discovered" at retrieval time.
  - Converting that representation into prompt-friendly format.
- Ideal cases of plans are just templates of a set of entities with corresponding ontological grounding which have been approved by a human critic.
- Such ideal cases can be used to generate fine-tuning data for the LLM as well as test cases for the continuous integration pipeline to detect potential drift by the underlying models.
- New cases that may be ideal candidates are added to a queue for human soundness critics to verify before they are used in recommendations for other users or in fine-tuning.
- Since models cannot be trusted to know when they have achieved the user's goals (which required verification and self-critique), we much continue to rely on traditional slot filling.
  - This is another example of wishful thinking that appears in such cases as the otherwise promising [current implementation of OpenInterpreter](https://github.com/OpenInterpreter/open-interpreter/blob/main/interpreter/core/core.py#L39).
  - However, models can extract the slot values from the collected outputs, and rules can determine when ther are all filled.
- If answers are opposites of ideal cases, they can be recycled to further ground the prompts with negative examples.

It seems that there is enough discourse in the literature to suggest that the Reyearn architecture remains a valid approach to building Explainable AI, even moreso in the age of LLMs.
