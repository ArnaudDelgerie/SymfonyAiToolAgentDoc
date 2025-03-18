# AgentConfig

## Overview
To create a `ToolAgent` or a `ConsoleToolAgent`, a class implementing `ArnaudDelgerie\AiToolAgent\Interface\AgentConfigInterface` is required.

You can use the `ArnaudDelgerie\AiToolAgent\Util\Config\AgentConfig` class provided by the bundle.
```php
use ArnaudDelgerie\AiToolAgent\Util\Config\AgentConfig;

$agentConfig = new AgentConfig(
	systemPrompt: 'You are a helpful assistant',
	functionNames: [],
    context: [], //optional
);
```

Or create your own class implementing `ArnaudDelgerie\AiToolAgent\Interface\AgentConfigInterface`.
```php
<?php

namespace App\Util\AgentConfig;

use ArnaudDelgerie\AiToolAgent\Interface\AgentConfigInterface;

class MyCustomAgentConfig implements AgentConfigInterface
{
    public function getSystemPrompt(): string
    {
        return 'You are a helpful assistant';
    }

    public function getFunctionNames(): array
    {
        return [];
    }

    public function getContext(): array
    {
        return [];
    }
}
```
```php
use App\Util\AgentConfig\MyCustomAgentConfig;

$agentConfig = new MyCustomAgentConfig();
```

## Parameters :
- **systemPrompt / getSystemPrompt :**
  The system prompt that defines the agent's role and behavior.

- **functionNames / getFunctionNames :**
  A list of `ToolFunctionManagerInterface` names (for `ToolAgent`) or `ConsoleToolFunctionManagerInterface` names (for `ConsoleToolAgent`) that the agent can use (at least one name is required).

- **context / getContext :**
  An array containing data that may be used by the `ToolFunctionManager` or `ConsoleToolFunctionManager`.