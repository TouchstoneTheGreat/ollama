# API

## Endpoints

- [Generate a completion](#generate-a-completion)
- [Create a Model](#create-a-model)
- [List Local Models](#list-local-models)
- [Show Model Information](#show-model-information)
- [Copy a Model](#copy-a-model)
- [Delete a Model](#delete-a-model)
- [Pull a Model](#pull-a-model)
- [Push a Model](#push-a-model)
- [Generate Embeddings](#generate-embeddings)

## Conventions

### Model names

Model names follow a `model:tag` format. Some examples are `orca-mini:3b-q4_1` and `llama2:70b`. The tag is optional and, if not provided, will default to `latest`. The tag is used to identify a specific version.

### Durations

All durations are returned in nanoseconds.

### Streaming responses

Certain endpoints stream responses as JSON objects delineated with the newline (`\n`) character.

## Generate a completion

```shell
POST /api/generate
```

Generate a response for a given prompt with a provided model. This is a streaming endpoint, so will be a series of responses. The final response object will include statistics and additional data from the request.

### Parameters

- `model`: (required) the [model name](#model-names)
- `prompt`: the prompt to generate a response for

Advanced parameters (optional):

- `options`: additional model parameters listed in the documentation for the [Modelfile](./modelfile.md#valid-parameters-and-values) such as `temperature`
- `system`: system prompt to (overrides what is defined in the `Modelfile`)
- `template`: the full prompt or prompt template (overrides what is defined in the `Modelfile`)
- `context`: the context parameter returned from a previous request to `/generate`, this can be used to keep a short conversational memory
- `stream`: if `false` the response will be returned as a single response object, rather than a stream of objects

### Examples

#### Request

```shell
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2:7b",
  "prompt": "Why is the sky blue?"
}'
```

#### Response

A stream of JSON objects is returned:

```json
{
  "model": "llama2:7b",
  "created_at": "2023-08-04T08:52:19.385406455-07:00",
  "response": "The",
  "done": false
}
```

The final response in the stream also includes additional data about the generation:

- `total_duration`: time spent generating the response
- `load_duration`: time spent in nanoseconds loading the model
- `sample_count`: number of samples generated
- `sample_duration`: time spent generating samples
- `prompt_eval_count`: number of tokens in the prompt
- `prompt_eval_duration`: time spent in nanoseconds evaluating the prompt
- `eval_count`: number of tokens the response
- `eval_duration`: time in nanoseconds spent generating the response
- `context`: an encoding of the conversation used in this response, this can be sent in the next request to keep a conversational memory
- `response`: empty if the response was streamed, if not streamed, this will contain the full response

To calculate how fast the response is generated in tokens per second (token/s), divide `eval_count` / `eval_duration`.

```json
{
  "model": "llama2:7b",
  "created_at": "2023-08-04T19:22:45.499127Z",
  "response": "",
  "context": [1, 2, 3],
  "done": true,
  "total_duration": 5589157167,
  "load_duration": 3013701500,
  "sample_count": 114,
  "sample_duration": 81442000,
  "prompt_eval_count": 46,
  "prompt_eval_duration": 1160282000,
  "eval_count": 113,
  "eval_duration": 1325948000
}
```

#### Request

```shell
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2:7b",
  "prompt": "Why is the sky blue?",
  "stream": false
}'
```

#### Response

If `stream` is set to `false`, the response will be a single JSON object:

```json
{
  "model": "llama2:7b",
  "created_at": "2023-08-04T19:22:45.499127Z",
  "response": "The sky is blue because it is the color of the sky.",
  "context": [1, 2, 3],
  "done": true,
  "total_duration": 5589157167,
  "load_duration": 3013701500,
  "sample_count": 114,
  "sample_duration": 81442000,
  "prompt_eval_count": 46,
  "prompt_eval_duration": 1160282000,
  "eval_count": 13,
  "eval_duration": 1325948000
}
```

## Create a Model

```shell
POST /api/create
```

Create a model from a [`Modelfile`](./modelfile.md)

### Parameters

- `name`: name of the model to create
- `path`: path to the Modelfile
- `stream`: (optional) if `false` the response will be returned as a single response object, rather than a stream of objects

### Examples

#### Request

```shell
curl -X POST http://localhost:11434/api/create -d '{
  "name": "mario",
  "path": "~/Modelfile"
}'
```

#### Response

A stream of JSON objects. When finished, `status` is `success`.

```json
{
  "status": "parsing modelfile"
}
```

## List Local Models

```shell
GET /api/tags
```

List models that are available locally.

### Examples

#### Request

```shell
curl http://localhost:11434/api/tags
```

#### Response

A single JSON object will be returned.

```json
{
  "models": [
    {
      "name": "llama2:7b",
      "modified_at": "2023-08-02T17:02:23.713454393-07:00",
      "size": 3791730596
    },
    {
      "name": "llama2:13b",
      "modified_at": "2023-08-08T12:08:38.093596297-07:00",
      "size": 7323310500
    }
  ]
}
```

## Show Model Information

```shell
POST /api/show
```

Show details about a model including modelfile, template, parameters, license, and system prompt.

### Parameters

- `name`: name of the model to show

### Examples

#### Request

```shell
curl http://localhost:11434/api/show -d '{
  "name": "llama2:7b"
}'
```

#### Response

```json
{
  "license": "<contents of license block>",
  "modelfile": "# Modelfile generated by \"ollama show\"\n# To build a new Modelfile based on this one, replace the FROM line with:\n# FROM llama2:latest\n\nFROM /Users/username/.ollama/models/blobs/sha256:8daa9615cce30c259a9555b1cc250d461d1bc69980a274b44d7eda0be78076d8\nTEMPLATE \"\"\"[INST] {{ if and .First .System }}<<SYS>>{{ .System }}<</SYS>>\n\n{{ end }}{{ .Prompt }} [/INST] \"\"\"\nSYSTEM \"\"\"\"\"\"\nPARAMETER stop [INST]\nPARAMETER stop [/INST]\nPARAMETER stop <<SYS>>\nPARAMETER stop <</SYS>>\n",
  "parameters": "stop                           [INST]\nstop                           [/INST]\nstop                           <<SYS>>\nstop                           <</SYS>>",
  "template": "[INST] {{ if and .First .System }}<<SYS>>{{ .System }}<</SYS>>\n\n{{ end }}{{ .Prompt }} [/INST] "
}
```

## Copy a Model

```shell
POST /api/copy
```

Copy a model. Creates a model with another name from an existing model.

### Examples

#### Request

```shell
curl http://localhost:11434/api/copy -d '{
  "source": "llama2:7b",
  "destination": "llama2-backup"
}'
```

#### Response

The only response is a 200 OK if successful.

## Delete a Model

```shell
DELETE /api/delete
```

Delete a model and its data.

### Parameters

- `name`: model name to delete

### Examples

#### Request

```shell
curl -X DELETE http://localhost:11434/api/delete -d '{
  "name": "llama2:13b"
}'
```

#### Response

If successful, the only response is a 200 OK.

## Pull a Model

```shell
POST /api/pull
```

Download a model from the ollama library. Cancelled pulls are resumed from where they left off, and multiple calls will share the same download progress.

### Parameters

- `name`: name of the model to pull
- `insecure`: (optional) allow insecure connections to the library. Only use this if you are pulling from your own library during development.
- `stream`: (optional) if `false` the response will be returned as a single response object, rather than a stream of objects

### Examples

#### Request

```shell
curl -X POST http://localhost:11434/api/pull -d '{
  "name": "llama2:7b"
}'
```

#### Response

If `stream` is not specified, or set to `true`, a stream of JSON objects is returned:

The first object is the manifest:

```json
{
  "status": "pulling manifest"
}
```

Then there is a series of downloading responses. Until any of the download is completed, the `completed` key may not be included. The number of files to be downloaded depends on the number of layers specified in the manifest.

```json
{
  "status": "downloading digestname",
  "digest": "digestname",
  "total": 2142590208,
  "completed": 241970
}
```

After all the files are downloaded, the final responses are:

```json
{
    "status": "verifying sha256 digest"
}
{
    "status": "writing manifest"
}
{
    "status": "removing any unused layers"
}
{
    "status": "success"
}
```

if `stream` is set to false, then the response is a single JSON object:

```json
{
  "status": "success"
}
```

## Push a Model

```shell
POST /api/push
```

Upload a model to a model library. Requires registering for ollama.ai and adding a public key first.

### Parameters

- `name`: name of the model to push in the form of `<namespace>/<model>:<tag>`
- `insecure`: (optional) allow insecure connections to the library. Only use this if you are pushing to your library during development.
- `stream`: (optional) if `false` the response will be returned as a single response object, rather than a stream of objects

### Examples

#### Request

```shell
curl -X POST http://localhost:11434/api/push -d '{
  "name": "mattw/pygmalion:latest"
}'
```

#### Response

If `stream` is not specified, or set to `true`, a stream of JSON objects is returned:

```json
{ "status": "retrieving manifest" }
```

and then:

```json
{
  "status": "starting upload",
  "digest": "sha256:bc07c81de745696fdf5afca05e065818a8149fb0c77266fb584d9b2cba3711ab",
  "total": 1928429856
}
```

Then there is a series of uploading responses:

```json
{
  "status": "starting upload",
  "digest": "sha256:bc07c81de745696fdf5afca05e065818a8149fb0c77266fb584d9b2cba3711ab",
  "total": 1928429856
}
```

Finally, when the upload is complete:

```json
{"status":"pushing manifest"}
{"status":"success"}
```

If `stream` is set to `false`, then the response is a single JSON object:

```json
{ "status": "success" }
```

## Generate Embeddings

```shell
POST /api/embeddings
```

Generate embeddings from a model

### Parameters

- `model`: name of model to generate embeddings from
- `prompt`: text to generate embeddings for

Advanced parameters:

- `options`: additional model parameters listed in the documentation for the [Modelfile](./modelfile.md#valid-parameters-and-values) such as `temperature`

### Examples

#### Request

```shell
curl -X POST http://localhost:11434/api/embeddings -d '{
  "model": "llama2:7b",
  "prompt": "Here is an article about llamas..."
}'
```

#### Response

```json
{
  "embedding": [
    0.5670403838157654, 0.009260174818336964, 0.23178744316101074, -0.2916173040866852, -0.8924556970596313,
    0.8785552978515625, -0.34576427936553955, 0.5742510557174683, -0.04222835972905159, -0.137906014919281
  ]
}
```
