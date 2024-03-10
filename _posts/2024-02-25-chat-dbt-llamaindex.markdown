---
layout: post
title:  "Chat to your dbt data project using LlamaIndex"
date:   2024-02-25 13:00:00 +0700
categories: data-engineering
tag: [llm,llama-index,dbt]
---

LLM applications are widely recognized for enhancing productivity across various domains, including data management and analysis. Utilizing LLM tools in your data platform can streamline your workflow and boost efficiency. It's anticipated that major cloud providers such as AWS, Azure, and GCP will soon introduce their products to facilitate seamless operations for engineers.

Why not get started with your own tool? It's not as complicated as you might think. Let me show you how to create a simple chat application that can assist with a dbt project.


![Image description](/images/chat-dbt/llama-index-dbt.png)


> TLDR;
> You can just pull the code and run it locally from this github repository: https://github.com/leehuwuj/chatdbt-llamaindex



## DBT

dbt (data build tool) is a crucial product for companies aiming to manage their data models seamlessly. While developing dbt models is straightforward, understanding an entire dbt project to design the model can be challenging. Additionally, many companies use dbt-core, which lacks an official management UI, making monitoring more difficult. In this post, we'll create a chat application that understands your dbt project, helping you to resolve issues more easily.


## LlamaIndex

If you're new to this complex world of AI, your first question might be "How...": How can I build that? Luckily, I have good news: using LlamaIndex, an LLM framework, it's easier than you think. You could simply pull my code from the GitHub repository below and start with it, but let's walk through it step by step.

LlamaIndex is an excellent choice for building an LLM application. You don't need to have extensive knowledge of the framework or start reading their documentation from scratch. Instead, get started with these tools:

- [create-llama](https://blog.llamaindex.ai/create-llama-a-command-line-tool-to-generate-llamaindex-apps-8f7683021191): A command-line interface that quickly creates a full-stack chat application.
- [LlamaHub](https://llamahub.ai): A centralized hub for all integrations with your app.
- [LlamaPack](https://blog.llamaindex.ai/introducing-llama-packs-e14f453b913a): This tool pulls the best practices from the community.

In this post, I won't discuss all the tools. Instead, I'll focus on starting the app and implementing a custom tool for the DBT project.


### How can we integrate LlamaIndex with dbt?

When initiating a chat with the LLM model, it only understands the knowledge with which it was trained. Each dbt project may have different configurations and data models, and we want the model to comprehend this information to aid in development and maintenance. By using `Agent` and `Tools`, we allow the model to reason your questions and make decisions to fetch information from your dbt project by executing certain functions.

LlamaIndex already has hundreds of integration packages in LlamaHub, implemented by the community. Unfortunately, there is no existing integration with dbt. However, don’t worry, it's not difficult to implement your own tools. I have already done so, and you can review it below.


So let get started.


### Generate code for the app by using create-llama.

You should install `npx` first, then just run the below command to create a full-stack app with FastAPI and NextJS (front-end).

```bash
❯ npx create-llama@latest

✔ What is your project named? … chatdbt
? Which template would you like to use? › - Use arrow-keys. Return to submit.
   Chat without streaming
❯  Chat with streaming
   Community template from https://github.com/run-llama/create_llama_projects
   Example using a LlamaPack
```

Let create a streaming app by choosing: `Chat with streaming` template. Then follows the options below:

```bash
✔ What is your project named? … chatdbt
✔ Which template would you like to use? › Chat with streaming
✔ Which framework would you like to use? › FastAPI (Python)
✔ Would you like to generate a NextJS frontend for your FastAPI (Python) backend? … No / Yes
✔ Which UI would you like to use? › Shadcn
✔ Which model would you like to use? › gpt-3.5-turbo
✔ Which data source would you like to use? › Use an example PDF
✔ Would you like to use a vector database? › No, just store the data in the file system
✔ Would you like to build an agent using tools? If so, select the tools here, otherwise just press enter ›
✔ Please provide your OpenAI API key (leave blank to skip): …
✔ How would you like to proceed? › Generate code and install dependencies (~2 min)
Creating a new LlamaIndex app in /tmp/demo/chatdbt.

Initializing Python project with template: .....
```

At the end, it’ll generate the code and install all python and npm packages to you.  

- To start the backend manually, you can go to the `backend` folder and start the app by running:

```python
poetry run python main.py
```

- To start the front-end app manually, you can go the `frontend` folder and start the app by running:

```python
npm run dev
```

### Create a custom tools to interact with your dbt project.

dbt manages your project using configuration files such as `project.yaml`, `profile.yaml`, `manifest.json`, `run_result.json`, and so on. By enabling the model to fetch these files, it can obtain all the information about your project. Therefore, let's implement a custom tool for this idea.

To create a custom tool, we just need to define the tool functions, along with a comment to guide the LLM model on what it is used for and how to call it. The code below is an example:

```python
class DbtRunResultToolSpec(BaseToolSpec):
    spec_functions = ["get_run_result"]

    def __init__(self, project_dir: str):
        self.project_dir = project_dir

    def get_run_result(self):
        """
        Get the run result of the dbt project.
        """
        run_result_path = path.join(self.project_dir, "target/run_results.json")
        with open(run_result_path) as f:
            run_result = json.loads(f.read())
            return Document(text=json.dumps(run_result))
```

We declared a tool call  `DbtRunResultToolSpec` which include a functions named `get_run_result` . This function have a guideline: `Get the run result of the dbt project.` to let the LLM know what is the function does. At the result, we already have a tool to allow LLM get the run result of the project by fetching the `run_results.json` file.

We can add more functions to fetch the project and model information by using the code below:

```python
class DbtManifestToolSpec(BaseToolSpec):
    spec_functions = ["get_project_sources", "get_models_info", "get_model_sql"]

    def __init__(self, project_dir: str):
        self.project_dir = project_dir

    def _fetch_manifest(self):
        manifest_path = path.join(self.project_dir, "target/manifest.json")
        with open(manifest_path) as f:
            raw = json.loads(f.read())
            manifest = parse_manifest(raw)
            return manifest

    @staticmethod
    def get_sub_attributes(data, sub_attributes: list[str]):
        """
        Get sub attributes of a dictionary.
        """
        for sub_attribute in sub_attributes:
            dict_data = str(data.__getattribute__(sub_attribute))
        return dict_data

    def get_project_sources(self):
        """
        Get the source database information of dbt project.
        """
        manifest = self._fetch_manifest()
        sources_info = {
            source_id: source_value.json()
            for source_id, source_value in manifest.sources.items()
        }

        return Document(text=json.dumps(sources_info))

    def get_models_info(self, model_id: Optional[str] = None):
        """
        Get model information of the dbt project.
        pass model_id to get specific model information, dont pass to get all models information.
        """
        manifest = self._fetch_manifest()
        models_info = {
            model_id: self.get_sub_attributes(
                model_value,
                [
                    "database",
                    "schema",
                    "name",
                    "relation_name",
                    "path",
                    "unique_id",
                    "columns",
                    "depends_on",
                ],
            )
            for model_id, model_value in manifest.nodes.items()
            if model_id.startswith("model")
        }

        return Document(text=json.dumps(models_info))

    def get_model_sql(self, model_id: str):
        """
        Get the sql of the model. Use can reasoning the relation, columns of a model from sql result of this function
        Args: model_id: the id of the model, it should start with "model.schema.model_name"
        """
        manifest = self._fetch_manifest()
        model = manifest.nodes.get(model_id)
        return Document(text=model.compiled_code)
```

So, now we already defined our own custom functions, so let add it to the created application.

- Create a new folder in `app/` project, named as `dbt` . And put the code in a [tool.py](http://tool.py) file
- Update the tools for the app in the `__init__.py` file at `app/engine` folder.

```python
def dbt_tools():
    """
    Returns a DBT tool.
    """
    from app.dbt import DbtManifestToolSpec, DbtRunResultToolSpec
    from app.constants import DBT_PROJECT_DIR

    if DBT_PROJECT_DIR is None:
        raise ValueError("DBT_PROJECT_DIR is not set")

    manifest_tools = DbtManifestToolSpec(project_dir=DBT_PROJECT_DIR).to_tool_list()

    run_result_tools = DbtRunResultToolSpec(project_dir=DBT_PROJECT_DIR).to_tool_list()

    return basic_tools + manifest_tools + run_result_tools

def get_chat_engine():
    """
    Constructs an AgentRunner with the default LLM, a query engine tool, and additional tools from the environment.
    """
    tools = []

    # Add the query engine tool
    tools.append(get_query_engine_tool())

    tools += dbt_tools()

    return ReActAgent.from_tools(
        tools=tools, 
				verbose=True
    )
```

You’ll need update these environment variables :

```python
# Your OPENAI api key
OPENAI_API_KEY=

# The directory of your dbt project
DPT_PROJECT_DIR=
```

That’s all, so let restart the back-end app, then go to the UI (at localhost:3000) and ask some questions about the dbt project:

![Image description](/images/chat-dbt/chat-dbt-first.png)

If you are new to dbt, you can play with it as a co-worker to become familiar with the dbt project.

![Image description](/images/chat-dbt/cht-dbt-second.png)


If you look at the log of the application app, it shown the reasoning progress of the model and which tools it used to get the information.

```python
INFO:     127.0.0.1:51986 - "POST /api/chat HTTP/1.1" 200 OK
Thought: I can use the `get_models_info` tool to get information about the models in the project.
Action: get_models_info
Action Input: {}
Observation: Doc ID: dd60a24b-1c1d-4d8f-8b6e-ac611682b033
Text: {"model.hackernews.stg_job_items": "macros=[]
nodes=['source.hackernews.hackernews.items']",
"model.hackernews.stg_comment_items": "macros=[]
nodes=['source.hackernews.hackernews.items']",
"model.hackernews.stg_story_items": "macros=[]
nodes=['source.hackernews.hackernews.items']",
"model.hackernews.items_fact": "macros=[] nodes=['model.hackerne...
INFO:     127.0.0.1:51989 - "POST /api/chat HTTP/1.1" 200 OK
Thought: I can use the `get_run_result` tool to check the latest status of the models.
Action: get_run_result
Action Input: {}
Observation: Doc ID: c0a1e261-386a-4ca9-9b9a-cb7f39680092
Text: {"metadata": {"dbt_schema_version":
"https://schemas.getdbt.com/dbt/run-results/v4.json", "dbt_version":
"1.4.5", "generated_at": "2023-04-01T01:43:43.036598Z",
"invocation_id": "2a02e6f3-99f3-48c6-a352-9af0dc105990", "env": {}},
"results": [{"status": "success", "timing": [{"name": "compile",
"started_at": "2023-04-01T01:43:40.649756Z", "comple...
INFO:     127.0.0.1:51993 - "POST /api/chat HTTP/1.1" 200 OK
INFO:     127.0.0.1:52004 - "POST /api/chat HTTP/1.1" 200 OK
INFO:     127.0.0.1:52007 - "POST /api/chat HTTP/1.1" 200 OK

```


You can just pull the code above and start the app from this repository:
https://github.com/leehuwuj/chatdbt-llamaindex



