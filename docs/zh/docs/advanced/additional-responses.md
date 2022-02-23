# OpenAPI的附加响应

你可以声明附加的响应，包括附加的状态码、媒体类型、描述等。

这些附加的响应将包含在 OpenAPI 架构中，因它们也将出现在 API 文档中。

但是在附加的响应中,你必须确保直接返回一个类似`JSONResponse`的`Response`，包含状态码和内容。

## 附加响应模型

你可以将参数响应传递给路径操作装饰器。

它接收一个字典，键是每个响应的状态代码，例如 200，值是其他字典，值中包含每个响应的信息。

每个响应字典都有一个'model'键，对应的值是一个 Pydantic 模型，就像 response_model 一样。

**FastAPI** 将使用该模型，生成其 JSON Schema 并将其包含在 OpenAPI 中的正确位置。

例如，要使用状态码 404 和 Pydantic 模型消息声明另一个响应，你可以编写：

```Python hl_lines="18  23"
{!../../../docs_src/additional_responses/tutorial001.py!}
```

!!! 注意
    切记必须直接返回 JSONResponse对象。

!!! info
    `model`不是 OpenAPI 中的键

    **FastAPI** 将从那里获取 Pydantic 模型，生成 JSON Schema，并将其放在正确的位置。

    正确的位置是指：
    
    * 在对应的值中，它具有另一个 JSON 对象 (dict) 作为值，其中包含：
        * 具有媒体类型的key值，例如 application/json，它包含另一个 JSON 对象作为值，其中包含：
            * 一个`schema`键，其值为模型中的 JSON 模式，这是正确的位置。
                * **FastAPI** 在此处添加对 OpenAPI 中另一个位置的全局 JSON 模式的引用，而不是直接包含它。 这样，其他应用程序和客户端可以直接使用这些 JSON Schema，提供更好的代码生成工具等。

此路径操作在 OpenAPI 中生成的响应将如下：

```JSON hl_lines="3-12"
{
    "responses": {
        "404": {
            "description": "Additional Response",
            "content": {
                "application/json": {
                    "schema": {
                        "$ref": "#/components/schemas/Message"
                    }
                }
            }
        },
        "200": {
            "description": "Successful Response",
            "content": {
                "application/json": {
                    "schema": {
                        "$ref": "#/components/schemas/Item"
                    }
                }
            }
        },
        "422": {
            "description": "Validation Error",
            "content": {
                "application/json": {
                    "schema": {
                        "$ref": "#/components/schemas/HTTPValidationError"
                    }
                }
            }
        }
    }
}
```

这些模式被引用到 OpenAPI 模式中的另一个地方：

```JSON hl_lines="4-16"
{
    "components": {
        "schemas": {
            "Message": {
                "title": "Message",
                "required": [
                    "message"
                ],
                "type": "object",
                "properties": {
                    "message": {
                        "title": "Message",
                        "type": "string"
                    }
                }
            },
            "Item": {
                "title": "Item",
                "required": [
                    "id",
                    "value"
                ],
                "type": "object",
                "properties": {
                    "id": {
                        "title": "Id",
                        "type": "string"
                    },
                    "value": {
                        "title": "Value",
                        "type": "string"
                    }
                }
            },
            "ValidationError": {
                "title": "ValidationError",
                "required": [
                    "loc",
                    "msg",
                    "type"
                ],
                "type": "object",
                "properties": {
                    "loc": {
                        "title": "Location",
                        "type": "array",
                        "items": {
                            "type": "string"
                        }
                    },
                    "msg": {
                        "title": "Message",
                        "type": "string"
                    },
                    "type": {
                        "title": "Error Type",
                        "type": "string"
                    }
                }
            },
            "HTTPValidationError": {
                "title": "HTTPValidationError",
                "type": "object",
                "properties": {
                    "detail": {
                        "title": "Detail",
                        "type": "array",
                        "items": {
                            "$ref": "#/components/schemas/ValidationError"
                        }
                    }
                }
            }
        }
    }
}
```

## 主响应的其他媒体类型

你同样可以使用 `responses` 参数为主响应添加不同的媒体类型。

例如，你可以添加额外的媒体类型 `image/png`，声明您的路径操作可以返回 JSON 对象（媒体类型为 `application/json`）或 PNG 图像：

```Python hl_lines="19-24  28"
{!../../../docs_src/additional_responses/tutorial002.py!}
```

!!! note
    Notice that you have to return the image using a `FileResponse` directly.

!!! info
    Unless you specify a different media type explicitly in your `responses` parameter, FastAPI will assume the response has the same media type as the main response class (default `application/json`).

    But if you have specified a custom response class with `None` as its media type, FastAPI will use `application/json` for any additional response that has an associated model.

## Combining information

You can also combine response information from multiple places, including the `response_model`, `status_code`, and `responses` parameters.

You can declare a `response_model`, using the default status code `200` (or a custom one if you need), and then declare additional information for that same response in `responses`, directly in the OpenAPI schema.

**FastAPI** will keep the additional information from `responses`, and combine it with the JSON Schema from your model.

For example, you can declare a response with a status code `404` that uses a Pydantic model and has a custom `description`.

And a response with a status code `200` that uses your `response_model`, but includes a custom `example`:

```Python hl_lines="20-31"
{!../../../docs_src/additional_responses/tutorial003.py!}
```

It will all be combined and included in your OpenAPI, and shown in the API docs:

<img src="/img/tutorial/additional-responses/image01.png">

## Combine predefined responses and custom ones

You might want to have some predefined responses that apply to many *path operations*, but you want to combine them with custom responses needed by each *path operation*.

For those cases, you can use the Python technique of "unpacking" a `dict` with `**dict_to_unpack`:

```Python
old_dict = {
    "old key": "old value",
    "second old key": "second old value",
}
new_dict = {**old_dict, "new key": "new value"}
```

Here, `new_dict` will contain all the key-value pairs from `old_dict` plus the new key-value pair:

```Python
{
    "old key": "old value",
    "second old key": "second old value",
    "new key": "new value",
}
```

You can use that technique to re-use some predefined responses in your *path operations* and combine them with additional custom ones.

For example:

```Python hl_lines="13-17  26"
{!../../../docs_src/additional_responses/tutorial004.py!}
```

## More information about OpenAPI responses

To see what exactly you can include in the responses, you can check these sections in the OpenAPI specification:

* <a href="https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md#responsesObject" class="external-link" target="_blank">OpenAPI Responses Object</a>, it includes the `Response Object`.
* <a href="https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md#responseObject" class="external-link" target="_blank">OpenAPI Response Object</a>, you can include anything from this directly in each response inside your `responses` parameter. Including `description`, `headers`, `content` (inside of this is that you declare different media types and JSON Schemas), and `links`.
