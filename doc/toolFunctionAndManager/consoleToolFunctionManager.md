# ConsoleToolFunctionManager

## Overview
A `ToolFunctionManager` must implement the `ArnaudDelgerie\AiToolAgent\Interface\ConsoleToolFunctionManagerInterface` to define a `ToolFunction` and its associated logic. At least one manager of this type is required when creating a `ConsoleToolAgent`.

### It must have methods : 
- `getName()`: Returns the function's name.
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\Util\AgentIO;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\Util\ToolValidation;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ConsoleToolFunctionManagerInterface;

class HelloToolFunctionManager implements ConsoleToolFunctionManagerInterface
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

use ArnaudDelgerie\AiToolAgent\Util\AgentIO;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\Util\ToolValidation;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ConsoleToolFunctionManagerInterface;

class HelloToolFunctionManager implements ConsoleToolFunctionManagerInterface
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

- `validate(array $args, array $context, array $responseContent, AgentIO $agentIO)`: Implements validation logic to determine whether the agent should call the `execute` method, preventing unnecessary execution if conditions are not met.
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\Util\AgentIO;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\Util\ToolValidation;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ConsoleToolFunctionManagerInterface;

class HelloToolFunctionManager implements ConsoleToolFunctionManagerInterface
{   
	// ...
    
    public function validate(array $args, array $context, array $responseContent, AgentIO $agentIO): ToolValidation
    {
        if(!$agentIO->confirm(\sprintf('Would you like the agent to say hello to %s ?', $args['to'])) {
            $reason = $agentIO->ask('Why ?');
            return new ToolValidation(
                args: $args,
                responseContent: $responseContent,
                isExecutable: false,
                message: $reason,
                stopSequence: true,
                stopRun: false,
            );
        }


        return new ToolValidation(
        	args: $args,
            responseContent: $responseContent,
            isExecutable: true
        );
    }
    
    // ...
}
```

- `execute(array $args, array $context, array $responseContent, AgentIO $agentIO)`: Implements the logic executed when the agent calls this `ConsoleToolFunctionManager`, and must return an instance of `ArnaudDelgerie\AiToolAgent\Util\ToolResponse`.
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\Util\AgentIO;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\Util\ToolValidation;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ConsoleToolFunctionManagerInterface;

class HelloToolFunctionManager implements ConsoleToolFunctionManagerInterface
{   
    // ...

    public function execute(array $args, array $context, array $responseContent, AgentIO $agentIO): ToolResponse
    {   
        /*
        The $args variable contains the values for each property of the ToolFunction.
        For example: $args = ['to' => 'World'];

		You can implement your logic directly within the execute() method.
		For example: $agentIO->text(\sprintf("Hello %s", $args['to']))

		Alternatively, you can add results to the $responseContent variable, which the agent includes in its final response.
		For example: $responseContent[self::getName()] = $args;

		When your agent calls multiple ConsoleToolFunctionManager, the $responseContent variable is passed from one 
		ConsoleToolFunctionManager to the next. Ensure you avoid overwriting results from previous function calls.
  
  		*/

        return new ToolResponse(
            responseContent: $responseContent,
            message: \sprintf("%s also says hello to you", $args['to']),
            stopRun: false
        );
    }
}
```

## `ToolValidation` parameters :
- **args :**
  The `$args` variable may be modified before reaching the `execute` method; therefore, it must be passed to the `ToolValidation`.

- **responseContent :**
  An array passed between `ConsoleToolFunctionManager` calls, allowing you to store and retrieve results in the agent's final response. 

- **isExecutable :**
  Determines whether the `execute` method can be called.

- **message :** 
  This message is returned to the agent only when `isExecutable` is **false**, allowing you to inform the agent why the function cannot be executed.

- **stopSequence :**
  Since an LLM API may call multiple functions within a single message, it can be necessary to skip remaining functions if the current function isn't executable. This parameter is ignored when `isExecutable` is **true**.

- **stopRun :**
  Determines whether the agent should stop after this validation step.  This parameter is ignored when `isExecutable` is **true** or `stopSequence` is **false**;

## `ToolResponse` parameters :
- **responseContent :**
  An array passed between `ConsoleToolFunctionManager` calls, allowing you to store and retrieve results in the agent's final response. If the logic is executed directly in the `execute()` method, this array can remain empty.

- **message :** 
  This message is returned to the agent.
  - If the `ConsoleToolFunctionManager` performs an action, the message typically confirms its completion.
  - If the `ConsoleToolFunctionManager` is used for **RAG** (Retrieval-Augmented Generation), the message usually provides contextual content for the agent.

- **stopRun :**
  Determines whether the agent should stop after executing this function. At least one `ConsoleToolFunctionManager` provided to the agent must have this parameter set to **true** to prevent infinite loops. If none of the existing `ConsoleToolFunctionManager` instances halt the agent's execution, you may include the `ArnaudDelgerie\AiToolAgent\ToolFunctionManager\ConsoleTasksCompletedToolFunctionManager` or `ArnaudDelgerie\AiToolAgent\ToolFunctionManager\ConsoleChatWithUserToolFunctionManager` in the list of functions supplied to your agent.