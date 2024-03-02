# Iudex Python Client SDK

**Easily build 🛠️ natural language interfaces for ✨ your own ✨ applications 💻**

[Iudex](https://iudex.ai) is an API layer that allows you to quickly build applications that can use LLM reasoning in a way that is more
accurate, secure, and scalable than the toy examples from other projects. Iudex does this by providing an out-of-the-box
LLM orchestration architecture capable of working with millions of functions and documents.

**❗ Leverage the power 🦾 of LLMs 🤖 with Iudex ❗**

## Installation

```bash
pip install iudex
```

## How It Works

Iudex works by first indexing your functions. Afterwards, when making a query, Iudex will figure out the best way to accomplish that query
by intelligently sequencing the functions you have connected and with the prebuilt functions we have created. This way, Iudex ensures
that the functions that get called do not suffer from hallucinations and can properly use the results from previously run functions.

## Usage Example

See more examples in the `examples` directory.

Before running examples, make sure to set the `IUDEX_API_KEY` environment variable or pass your API key directly to the `Iudex` constructor.

Visit [iudex.ai](https://iudex.ai) to Sign Up and receive an API key.

```python
import json
from iudex import Iudex
from openai.types.chat.completion_create_params import Function

client = Iudex()

get_current_weather_spec = Function(
    name='get_current_weather',
    description='Gets the current weather',
    parameters={
        'type': 'object',
        'properties': {
            'location': {
                'type': 'string',
                'description': 'The city and state, e.g., San Francisco, CA',
            },
            'unit': {
                'type': 'string',
                'enum': ['celsius', 'fahrenheit'],
                'description': 'The temperature unit',
            },
        },
        'required': ['location'],
    },
)

def get_current_weather(location, unit='fahrenheit'):
    temp = 70
    if unit == 'celsius':
        temp = temp // 2
    return {
        'temp': temp,
        'unit': unit,
        'humidity': 0.5,
        'wind_speed': 1,
        'wind_unit': 'mph',
        'wind_direction': 'NE',
        'precipitation': 0.1,
        'description': 'partially sunny',
    }

NAME_TO_FUNCTION = {
    'get_current_weather': get_current_weather,
}

def upload_functions():
    functions = [get_current_weather_spec]
    client.functions.upsert(functions=functions, module='weather_module')
    print('Successfully uploaded functions!\n')

def run_weather_chatbot():
    messages = [{'role': 'user', 'content': 'What is the weather in Philadelphia, PA?'}]

    while True:
        res = client.chat.completions.create(
            messages=messages,
            model='gpt-4-turbo-preview',
        )

        msg = res.choices[0].message
        messages.append(msg)

        tool_calls = msg.tool_calls
        if not tool_calls:
            break

        print('message:', messages[-1], '\n')

        for tool_call in tool_calls:
            fn_name = tool_call.function.name
            if fn_name not in NAME_TO_FUNCTION:
                raise ValueError(f'Unsupported function name: {fn_name}')
            fn = NAME_TO_FUNCTION[fn_name]

            fn_args = tool_call.function.arguments.replace('\'', '\"').replace('None', '\"null\"')
            print('fn_args:', fn_args, '\n')
            fn_return = fn(**json.loads(fn_args))
            print('fn_return:', json.dumps(fn_return), '\n')

            messages.append(
                {
                    'role': 'tool',
                    'content': json.dumps(fn_return),
                    'tool_call_id': tool_call.id,
                }
            )
    
    print('Final response:', messages[-1], '\n')

if __name__ == '__main__':
    upload_functions()
    run_weather_chatbot()
```

## Developer / Contributing

While Iudex is currently in its nascent stages, we welcome contributions from the developer community.

We use Poetry for dependency management and packaging.

To get started:

1. Fork the repository and clone it locally.
2. Install Poetry on your system if you haven't already.
3. Run `poetry install` to install dependencies and the project as editable using `pip install --editable .`.
4. Run scripts or tests with `poetry run` or `poetry run pytest -s`. Alternatively start a nested shell with `poetry shell`.
