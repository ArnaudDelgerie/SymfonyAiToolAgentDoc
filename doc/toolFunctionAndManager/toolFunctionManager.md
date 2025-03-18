# ToolFunctionManager

## Overview
A `ToolFunctionManager` must implement the `ArnaudDelgerie\AiToolAgent\Interface\ToolFunctionManagerInterface` to define a `ToolFunction` and its associated logic. At least one manager of this type is required when creating a `ToolAgent`.

### It must have methods : 

- `getName()`: Returns the function's name.
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ToolFunctionManagerInterface;

class HelloToolFunctionManager implements ToolFunctionManagerInterface
{   
    public static function getName(): string
    {
        return 'say_hello';
    }
    
    // ...
}
```

- `getToolFunction(array $context)`: Defines and returns an instance of `ArnaudDelgerie\AiToolAgent\DTO\ToolFunction`.
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ToolFunctionManagerInterface;

class HelloToolFunctionManager implements ToolFunctionManagerInterface
{   
	// ...
    
    public function getToolFunction(array $context): ToolFunction
    {
        return (new ToolFunction())
            // Ensure the function name matches self::getName()
            ->setName(self::getName())
            // Provide a description to help the tool agent determine when to use it
            ->setDescription('Use this feature to say hello to someone')
            // Define required properties for the function
            // the agent himself determines the value of each property, based on the user's prompt
            ->addProperty(
                'to',
                (new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::String)
                    ->setDescription('Name of the person to say hello to')
            );
    }
    
    // ...
}
```

- `execute(array $args, array $context, array $responseContent)`: Implements the logic executed when the agent calls this `ToolFunctionManager`, and must return an instance of `ArnaudDelgerie\AiToolAgent\Util\ToolResponse`.
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ToolFunctionManagerInterface;

class HelloToolFunctionManager implements ToolFunctionManagerInterface
{   
    // ...

    public function execute(array $args, array $context, array $responseContent): ToolResponse
    {   
        /*
        The $args variable contains the values for each property of the ToolFunction.
        For example: $args = ['to' => 'World'];

		You can implement your logic directly within the execute() method, such as storing data in a database.

		Alternatively, you can add results to the $responseContent variable, which the agent includes in its final response.
		For example: $responseContent[self::getName()] = $args;

		When your agent calls multiple ToolFunctionManager, the $responseContent variable is passed from one ToolFunctionManager 
		to the next. Ensure you avoid overwriting results from previous function calls.
  
  		*/

        return new ToolResponse(
            responseContent: $responseContent,
            message: \sprintf("%s also says hello to you", $args['to']),
            stopRun: false
        );
    }
}
```

## `ToolResponse` parameters :
- **responseContent :**
  An array passed between `ToolFunctionManager` calls, allowing you to store and retrieve results in the agent's final response. If the logic is executed directly in the `execute()` method, this array can remain empty.

- **message :** 
  This message is returned to the agent.
  - If the `ToolFunctionManager` performs an action, the message typically confirms its completion.
  - If the `ToolFunctionManager` is used for **RAG** (Retrieval-Augmented Generation), the message usually provides contextual content for the agent.

- **stopRun :**
  Determines whether the agent should stop after executing this function. At least one `ToolFunctionManager` provided to the agent must have this parameter set to **true** to prevent infinite loops. If none of the existing `ToolFunctionManager` instances halt the agent's execution, you may include the `ArnaudDelgerie\AiToolAgent\ToolFunctionManager\TasksCompletedToolFunctionManager` in the list of functions supplied to your agent.