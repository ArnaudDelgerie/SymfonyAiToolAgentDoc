# ClientConfig

## Overview
To create a `ToolAgent` or a `ConsoleToolAgent`, a class implementing `ArnaudDelgerie\AiToolAgent\Interface\ClientConfigInterface` is required.

You can use the `ArnaudDelgerie\AiToolAgent\Util\Config\ClientConfig` class provided by the bundle.
```php
use ArnaudDelgerie\AiToolAgent\Enum\ClientEnum;
use ArnaudDelgerie\AiToolAgent\Util\Config\ClientConfig;

$clientConfig = new ClientConfig(
	clientEnum: ClientEnum::Mistral,
    apiKey: 'openai, mistral or anthropic api key',
    model: 'openai, mistral or anthropic model name',
    temperature: 1.0, // optional
    requestLimit: null, // optional
    timeout: 60, // optional
    maxOutputToken: 8192 // optional
);
```

Or create your own class implementing `ArnaudDelgerie\AiToolAgent\Interface\ClientConfigInterface`.
```php
<?php

namespace App\Util\ClientConfig;

use ArnaudDelgerie\AiToolAgent\Enum\ClientEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ClientConfigInterface;

class MyCustomClientConfig implements ClientConfigInterface
{
    public function getClientEnum(): ClientEnum
    {
        return ClientEnum::Mistral;
    }

    public function getApiKey(): string
    {
        return 'openai, mistral or anthropic api key';
    }

    public function getModel(): string
    {
        return 'openai, mistral or anthropic model';
    }
    
    public function getTemperature(): float
    {
        return 1.0;
    }

    public function getRequestLimit(): ?int
    {
        return null;
    }

    public function getTimeout(): int
    {
        return 60;
    }

    public function getMaxOutputToken(): int
    {
        return 8192;
    }
}
```
```php
use App\Util\ClientConfig\MyCustomClientConfig;

$clientConfig = new MyCustomClientConfig();
```

## Parameters :
- **clientEnum / getClientEnum :**
  `ArnaudDelgerie\AiToolAgent\Enum\ClientEnum` that defines the client used by the agent (available cases: `Openai`, `Mistral` or `Anthropic`).

- **apiKey / getApiKey :** 
  The API key of the client used by the agent.

- **model / getModel :**
  The AI model used by the agent.

- **temperature / getTemperature :**
  What sampling temperature to use, between 0  and 2. Higher values like 0.8 will make the output more random, while  lower values like 0.2 will make it more focused and deterministic.

- **requestLimit / getRequestLimit :**
  The maximum number of API calls the agent can make between each "run", set to null to have no limit.

- **timeout / getTimeout :**
  Timeout for each API call.

- **maxOutputToken / getMaxOutputToken :**
  The maximum number of tokens that the API can return on each call, must not be greater than the model's capacity.