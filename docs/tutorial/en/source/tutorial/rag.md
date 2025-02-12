# RAG

We want to introduce three concepts related to RAG in AgentScope: Knowledge, KnowledgeBank and RAG agent.

The Knowledge modules (now only `LlamaIndexKnowledge`; support for LangChain will come soon) are responsible for handling all RAG-related operations.

## Creating Knowledge
A Knowledge object can be created with a JSON configuration to specify 1) data path, 2) data loader, 3) data preprocessing methods, and 4) embedding model (model config name).
A detailed example can refer to the following:
<details>
<summary> A detailed example of Knowledge object configuration </summary>

```json
[
{
"knowledge_id": "{your_knowledge_id}",
"emb_model_config_name": "{your_embed_model_config_name}",
"data_processing": [
  {
    "load_data": {
      "loader": {
        "create_object": true,
        "module": "llama_index.core",
        "class": "SimpleDirectoryReader",
        "init_args": {
          "input_dir": "{path_to_your_data_dir_1}",
          "required_exts": [".md"]
        }
      }
    }
  },
  {
    "load_data": {
      "loader": {
        "create_object": true,
        "module": "llama_index.core",
        "class": "SimpleDirectoryReader",
        "init_args": {
          "input_dir": "{path_to_your_python_code_data_dir}",
          "recursive": true,
          "required_exts": [".py"]
        }
      }
    },
    "store_and_index": {
      "transformations": [
        {
          "create_object": true,
          "module": "llama_index.core.node_parser",
          "class": "CodeSplitter",
          "init_args": {
            "language": "python",
            "chunk_lines": 100
          }
        }
      ]
    }
  }
]
}
]
```

</details>

### Configuring Knowledge

The aforementioned configuration is usually saved as a JSON file, it musts
contain the following key attributes,
* `knowledge_id`: a unique identifier of the knowledge;
* `emb_model_config_name`: the name of the embedding model;
* `chunk_size`: default chunk size for the document transformation (node parser);
* `chunk_overlap`: default chunk overlap for each chunk (node);
* `data_processing`: a list of data processing methods.


#### Using LlamaIndexKnowledge

Regarding the last attribute `data_processing`, each entry of the list (which is a dict) configures a data
loader object that loads the needed data (i.e. `load_data`),
and a transformation object that the process the loaded data (`store_and_index`).
Accordingly, one may load data from multiple sources (with different data loaders),
process with individually defined manners (i.e. transformation or node parser),
and merge the processed data into a single index for later retrieval.
For more information about the components, please refer to
[LlamaIndex-Loading Data](https://docs.llamaindex.ai/en/stable/module_guides/loading/).
In common, we need to set the following attributes
* `create_object`: indicates whether to create a new object, must be true in this case;
* `module`: where the class is located;
* `class`: the name of the class.

More specifically, for setting the `load_data`, you can use a wide collection of data loaders,
    such as `SimpleDirectoryReader` (in `class`), provided by Llama-index, to load a various collection of data types
    (e.g. txt, pdf, html, py, md, etc.). Regarding this data loader, you can set the following attributes
* `input_dir`: the path to the data directory;
* `required_exts`: the file extensions that the data loader will load.

For more information about the data loaders, please refer to [here](https://docs.llamaindex.ai/en/stable/module_guides/loading/simpledirectoryreader/)

For `store_and_index`, it is optional and if it is not specified, the default transformation (a.k.a. node parser) is `SentenceSplitter`. For some specific node parser such as `CodeSplitter`, users can set the following attributes:
* `language`: the language of the code;
* `chunk_lines`: the number of lines for each of the code chunk.

For more information about the node parsers, please refer to [here](https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/).


If users want to avoid the detailed configuration, we also provide a quick way in `KnowledgeBank` (see the following).

### How to use a Knowledge object
After a knowledge object is created successfully, users can retrieve information related to their queries by calling `.retrieve(...)` function.
The `.retrieve` function accepts at least three basic parameters:
* `query`: input that will be matched in the knowledge;
* `similarity_top_k`: how many most similar "data blocks" will be returned;
* `to_list_strs`: whether return the retrieved information as strings.

*Advanaced:* In `LlamaIndexKnowledge`, it also supports users passing their own retriever to retrieve from knowledge.

### More details inside `LlamaIndexKnowledge`
Here, we will use `LlamaIndexKnowledge` as an example to illustrate the operation within the `Knowledge` module.
When a `LlamaIndexKnowledge` object is initialized, the `LlamaIndexKnowledge.__init__` will go through the following steps:
  *  It processes data and prepare for retrieval in `LlamaIndexKnowledge._data_to_index(...)`, which includes
      * loading the data `LlamaIndexKnowledge._data_to_docs(...)`;
      * preprocessing the data with preprocessing methods (e.g., splitting) and embedding model `LlamaIndexKnowledge._docs_to_nodes(...)`;
      * get ready for being query, i.e. generate indexing for the processed data.
  * If the indexing already exists, then `LlamaIndexKnowledge._load_index(...)` will be invoked to load the index and avoid repeating embedding calls.
</br>

## Knowledge Bank
The knowledge bank maintains a collection of Knowledge objects (e.g., on different datasets) as a set of *knowledge*. Thus,
different agents can reuse the Knowledge object without unnecessary "re-initialization".
Considering that configuring the Knowledge object may be too complicated for most users, the knowledge bank also provides an easy function call to create Knowledge objects.
* `KnowledgeBank.add_data_as_knowledge`: create Knowledge object. An easy way only requires to provide `knowledge_id`, `emb_model_name` and `data_dirs_and_types`.
As knowledge bank process files as `LlamaIndexKnowledge` by default, all text file types are supported, such as `.txt`, `.html`, `.md`, `.csv`, `.pdf` and all code file like `.py`.  File types other than the text can refer to [LlamaIndex document](https://docs.llamaindex.ai/en/stable/module_guides/loading/simpledirectoryreader/).
```python
knowledge_bank.add_data_as_knowledge(
    knowledge_id="agentscope_tutorial_rag",
    emb_model_name="qwen_emb_config",
    data_dirs_and_types={
        "../../docs/sphinx_doc/en/source/tutorial": [".md"],
    },
)
```
More advance initialization, users can still pass a knowledge config as a parameter `knowledge_config`:
```python
# load knowledge_config as dict
knowledge_bank.add_data_as_knowledge(
  knowledge_id=knowledge_config["knowledge_id"],
  emb_model_name=knowledge_config["emb_model_config_name"],
  knowledge_config=knowledge_config,
)
```
* `KnowledgeBank.get_knowledge`: It accepts two parameters, `knowledge_id` and `duplicate`.
  It will return a knowledge object with the provided `knowledge_id`; if `duplicate` is true, the return will be deep copied.
* `KnowledgeBank.equip`: It accepts three parameters, `agent`, `knowledge_id_list` and `duplicate`.
 The function will provide knowledge objects according to the `knowledge_id_list` and put them into `agent.knowledge_list`. If `duplicate` is true, the assigned knowledge object will be deep copied first.

## RAG agent
RAG agent is an agent that can generate answers based on the retrieved knowledge.
  * Agent using RAG: a RAG agent has a list of knowledge objects (`knowledge_list`).
    * RAG agent can be initialized with a `knowledge_list`
      ```python
        knowledge = knowledge_bank.get_knowledge(knowledge_id)
        agent = LlamaIndexAgent(
            name="rag_worker",
            sys_prompt="{your_prompt}",
            model_config_name="{your_model}",
            knowledge_list=[knowledge], # provide knowledge object directly
            similarity_top_k=3,
            log_retrieval=False,
            recent_n_mem_for_retrieve=1,
        )
      ```
    * If RAG agent is build with a configurations with `knowledge_id_list` specified, agent can load specific knowledge from a `KnowledgeBank` by passing it and a list ids into the `KnowledgeBank.equip` function.
       ```python
          # >>> agent.knowledge_list
          # >>> []
          knowledge_bank.equip(agent, agent.knowledge_id_list)
          # >>> agent.knowledge_list
          # [<LlamaIndexKnowledge object at 0x16e516fb0>]
      ```
  * Agent can use the retrieved knowledge in the `reply` function and compose their prompt to LLMs.



**Building RAG agent yourself.** As long as you provide a list of knowledge id, you can pass it with your agent to the `KnowledgeBank.equip`.
Your agent will be equipped with a list of knowledge according to the `knowledge_id_list`.
You can decide how to use the retrieved content and even update and refresh the index in your agent's `reply` function.

## (Optional) Setting up a local embedding model service

For those who are interested in setting up a local embedding service, we provide the following example based on the
`sentence_transformers` package, which is a popular specialized package for embedding models (based on the `transformer` package and compatible with both HuggingFace and ModelScope models).
In this example, we will use one of the SOTA embedding models, `gte-Qwen2-7B-instruct`.

* Step 1: Follow the instruction on [HuggingFace](https://huggingface.co/Alibaba-NLP/gte-Qwen2-7B-instruct) or [ModelScope](https://www.modelscope.cn/models/iic/gte_Qwen2-7B-instruct ) to download the embedding model.
  (For those who cannot access HuggingFace directly, you may want to use a HuggingFace mirror by running a bash command
    `export HF_ENDPOINT=https://hf-mirror.com` or add a line of code `os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"` in your Python code.)
* Step 2: Set up the server. The following code is for reference.

```python
import datetime
import argparse

from flask import Flask
from flask import request
from sentence_transformers import SentenceTransformer

def create_timestamp(format_: str = "%Y-%m-%d %H:%M:%S") -> str:
    """Get current timestamp."""
    return datetime.datetime.now().strftime(format_)

app = Flask(__name__)

@app.route("/embedding/", methods=["POST"])
def get_embedding() -> dict:
    """Receive post request and return response"""
    json = request.get_json()

    inputs = json.pop("inputs")

    global model

    if isinstance(inputs, str):
        inputs = [inputs]

    embeddings = model.encode(inputs)

    return {
        "data": {
            "completion_tokens": 0,
            "messages": {},
            "prompt_tokens": 0,
            "response": {
                "data": [
                    {
                        "embedding": emb.astype(float).tolist(),
                    }
                    for emb in embeddings
                ],
                "created": "",
                "id": create_timestamp(),
                "model": "flask_model",
                "object": "text_completion",
                "usage": {
                    "completion_tokens": 0,
                    "prompt_tokens": 0,
                    "total_tokens": 0,
                },
            },
            "total_tokens": 0,
            "username": "",
        },
    }

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--model_name_or_path", type=str, required=True)
    parser.add_argument("--device", type=str, default="auto")
    parser.add_argument("--port", type=int, default=8000)
    args = parser.parse_args()

    global model

    print("setting up for embedding model....")
    model = SentenceTransformer(
        args.model_name_or_path
    )

    app.run(port=args.port)
```

* Step 3: start server.
```bash
python setup_ms_service.py --model_name_or_path {$PATH_TO_gte_Qwen2_7B_instruct}
```


Testing whether the model is running successfully.
```python
from agentscope.models.post_model import PostAPIEmbeddingWrapper


model = PostAPIEmbeddingWrapper(
    config_name="test_config",
    api_url="http://127.0.0.1:8000/embedding/",
    json_args={
        "max_length": 4096,
        "temperature": 0.5
    }
)

print(model("testing"))
```
