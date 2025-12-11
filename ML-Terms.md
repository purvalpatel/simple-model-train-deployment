RAG
-----
RAG (Retrival Augmented Generation) <br>
R (Retrival)  - Fetch the information from your data source. (PDF, Websites, docs etc. ) <br>
A (Augmentation) - Insert (Augment) the retrived data into LLM's Prompt. <br>
G (Generation) - LLM uses the retrived context to generate accurate answer. <br>

employee-handbook.pdf <br>
You ask the LLM: <br>
```
“How many casual leaves per year do employees get?”
```
**Without RAG** → LLM hallucinates. <br>
**With RAG** → LLM pulls the answer from YOUR PDF. <br>
Provides answers from your data.
