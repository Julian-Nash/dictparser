# dictparser

![Python package](https://github.com/Julian-Nash/dictparse/workflows/Python%20package/badge.svg?branch=master)

A simple, slim and useful, zero-dependency utility for parsing JSON strings, dictionaries or dictionary-like objects
 (think Flask's `request.form`, `request.json` or `request.get_json()`).

`dictparser` borrows a slimmed down design from Python's own `argparse` library, similar to the `ArgumentParser` class
, taking input as a dictionary like object or a dictionary string as JSON, enforcing rules, types, actions, default
 values and returning a `NameSpace`, with values mapped to attributes.
 
### Installation

```shell script
pip install dictparser
```

### Example

The following code is a Python program that takes takes some data in the form of a dictionary:

```pycon
>>> from dictparser import DictionaryParser
>>> parser = DictionaryParser()
>>> parser.add_param("name", str, required=True)
>>> params = parser.parse_params({"name": "FooBar"})
>>> params.name
'FooBar'
```

### Creating a parser

The first step is to create the DictionaryParser object

```pycon
>>> from dictparser import DictionaryParser
>>> parser = DictionaryParser(description="Create a new user")
```

### Adding parameters

Adding parameters to the parser is done by making calls to the `add_param` method. These calls tell the
 `DictionaryParser` how to handle the values passed in and turn them into the desired output, enforcing rules
 , changing types and transforming values based on the arguments passed to the `add_param` method.
 
```pycon
>>> parser = DictionaryParser()
>>> parser.add_param("name", str, required=True)
>>> parser.add_param("language", str, choices=["python", "javascript", "rust"])
>>> parser.add_param("tags", str, action=lambda x: x.split(","))
>>> params = parser.parse_params({"name": "FooBar", "language": "python", "tags": "foo,bar,baz"})
>>> params.name
'FooBar'
>>> params.language
'python'
>>> params.tags
['foo', 'bar', 'baz']
>>> params.to_dict()
{'name': 'FooBar', 'language': 'python', 'tags': ['foo', 'bar', 'baz']}
```

If the parser does not find a value matching the name, the default value is `None`

### Arguments available for `add_param`

```py3
DictionaryParser.add_param(
    self,
    name: str,
    type_: Optional[type] = None,
    dest: Optional[str] = None,
    required: Optional[bool] = False,
    choices: Optional[Union[list, set, tuple]] = None,
    action: Optional[Callable] = None,
    description: Optional[str] = None,
    default: Optional[Any] = None,
    regex: Optional[str] = None
) -> None
```

- `name`: The parameter name (required)
- `type_`: The common parameter type (The parser will attempt to convert the parameter value to the given type)
- `dest`: The destination name of the parameter
- `required`: If `True`, enforce a value for the parameter must exists
- `choices`: A list, set, or tuple of possible choices
- `action`: A callable to apply to the value (Applied after any type conversion)
- `description`: A description of the parameter
- `default`: A default value for the parameter if not found
- `regex`: A regular expression to match against

### Parsing the data

After creating the parser and adding parameters to it, data can be parsed by calling the `parse_params` method, passing
in the data to be parsed. This returns a `NameSpace` object.

```py3
DictionaryParser.parse_params(
    self, 
    data: Union[dict, str], 
    strict: Optional[bool] = False
) -> NameSpace:
```

- `data`: A dictionary, dictionary-like object or a valid JSON string that can be decoded to a dictionary
- `strict`: If `True`, raises an exception if any parameters not added to the parser are received

### The `NameSpace` object

A `NameSpace` object is returned when calling `parse_params` and contains the parsed data after applying your rules
defined when adding arguments.

Parameters can be accessed as attributes of the `NameSpace`, simply using dot notation:

```pycon
>>> parser = DictionaryParser()
>>> parser.add_param("age", int, required=True)
>>> params = parser.parse_params({"age": 30})
>>> params.age
30
```

A standard `AttributeError` will be raised if an attribute is accessed without being added to the parser:

```pycon
>>> params.foo
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'NameSpace' object has no attribute 'foo'
```

if the `dest` parameter is supplied when adding a parameter in `add_param`, the value can only be accessed by using the
`dest` value:

```pycon
>>> parser = DictionaryParser()
>>> parser.add_param("bar", str, dest="foo")
>>> params = parser.parse_params({"bar": "bar"})
>>> params.foo
'bar'
```

The `NameSpace` object has the following available methods:

#### get

```py3
NameSpace.get(
    self, 
    name: str, 
    default: Optional[Any] = None
) -> Union[None, Any]:
```

Calling the `get` method on the `NameSpace` works in the same way as calling `get` on a dictionary, returning either
 the value for the `name` parameter or `None` if the `NameSpace` does not have that attribute.
 
```pycon
>>> parser = DictionaryParser()
>>> parser.add_param("age", int, required=True)
>>> parser.add_param("weight", int)
>>> params = parser.parse_params({"age": 30, "height": 1.9})
>>> params.weight
None
```

#### to_dict

```py3
NameSpace.to_dict(self) -> dict
```

Returns a dictionary of the parsed parameters.

```pycon
>>> parser = DictionaryParser()
>>> parser.add_param("one", str)
>>> parser.add_param("two", int)
>>> parser.add_param("three", list)
>>> params = parser.parse_params({"one": "one", "two": 2, "three": [1, 2, 3]})
>>> params.to_dict()
{'one': 'one', 'two': 2, 'three': [1, 2, 3]}
```

### Flask example

An example of parsing JSON data sent in a POST request to a Flask route:

```py3
from app.users import create_user

from flask import Flask, request
from respond import JSONResponse
from dictparser import DictionaryParser


def create_app():

    app = Flask(__name__)

    @app.route("/", methods=["POST"])
    def post():

        parser = DictionaryParser(description="Create a new user")

        parser.add_param("name", str, required=True)
        parser.add_param("age", int)
        parser.add_param("password", str, required=True, action=lambda x: x.encode("utf-8"))
        parser.add_param("interests", list, action=lambda x: [i.strip() for i in x])
        parser.add_param("level", float, default=1.5)
        parser.add_param("stage", str, choices=["alpha", "beta"])

        try:
            params = parser.parse_params(request.get_json())
        except Exception as e:
            return JSONResponse.bad_request(str(e))

        user = create_user(
            name=params.name,
            age=params.age,
            password=params.password,
            interests=params.interests,
            level=params.level,
            stage=params.stage
        )

        return JSONResponse.created(user.to_dict())

    return app


if __name__ == "__main__":
    app = create_app()
    app.run()

```