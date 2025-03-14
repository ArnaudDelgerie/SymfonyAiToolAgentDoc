# Example 2 : code writer

## Purpose :
Our goal is to develop a `ConsoleToolAgent` code writer capable of creating and modifying files.

## Implementation :

**Step 1 :** create `GetFileContentToolFunctionManagager`
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
use Symfony\Component\DependencyInjection\ParameterBag\ParameterBagInterface;

class GetFileContentToolFunctionManager implements ConsoleToolFunctionManagerInterface
{
    public function __construct(private ParameterBagInterface $params) {}

    public static function getName(): string
    {
        return 'get_file_content';
    }

    public function getToolFunction(array $context): ToolFunction
    {
        return (new ToolFunction())
            ->setName(self::getName())
            ->setDescription('Retrieves the content of a specified file.')
            ->addProperty(
                'file_path',
                (new ToolFunctionProperty())->setType(ToolFunctionPropertyTypeEnum::String)
            );
    }

    public function validate(array $args, array $context, array $responseContent, AgentIO $agentIO): ToolValidation
    {
        // Ensure the file path starts with "/"
        if (!str_starts_with($args['file_path'], '/')) {
            $args['file_path'] = '/' . $args['file_path'];
        }

        // Check if the file exists
        $fileFullPath = $this->params->get('kernel.project_dir') . $args['file_path'];
        if (!file_exists($fileFullPath)) {
            $message = \sprintf('The file %s does not exist. Ask the user for the correct path.', $args['file_path']);

            // Mark the task as not executable, preventing the "execute" method from running
            // Stop the sequence to prevent execution of subsequent tool calls
            return new ToolValidation($args, $responseContent, isExecutable: false, message: $message, stopSequence: true);
        }

        // Mark the task as executable
        return new ToolValidation($args, $responseContent, isExecutable: true);
    }

    public function execute(array $args, array $context, array $responseContent, AgentIO $agentIO): ToolResponse
    {   
        // Notify the user that the file is being retrieved
        $agentIO->text(\sprintf('%s file retrieval', $args['file_path']));

        // Read the file content
        $fileContent = file_get_contents($this->params->get('kernel.project_dir') . $args['file_path']);

        // Return the file content to the agent and continue execution
        return new ToolResponse($responseContent, message: $fileContent, stopRun: false);
    }
}
```

**Step 2 :** create `SaveFileToolFunctionManager`
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
use Symfony\Component\DependencyInjection\ParameterBag\ParameterBagInterface;

class SaveFileToolFunctionManager implements ConsoleToolFunctionManagerInterface
{
    public function __construct(private ParameterBagInterface $params) {}

    public static function getName(): string
    {
        return 'save_file';
    }

    public function getToolFunction(array $context): ToolFunction
    {
        return (new ToolFunction())
            ->setName(self::getName())
            ->setDescription('Saves or updates a file with provided content.')
            ->addProperty(
                'file_path',
                (new ToolFunctionProperty())->setType(ToolFunctionPropertyTypeEnum::String)
            )
            ->addProperty(
                'content',
                (new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::String)
                    ->setDescription('Content to be saved or updated.')
            );
    }

    public function validate(array $args, array $context, array $responseContent, AgentIO $agentIO): ToolValidation
    {
        // Display the generated or updated content
        $agentIO->text('Generated content:' . PHP_EOL . PHP_EOL . trim($args['content']));
        
        // Ask the user for confirmation before saving
        if (!$agentIO->confirm('Do you want to save it?')) {
            // If declined, ask the user for a reason
            $reason = $agentIO->ask('Why?');

            // Mark the task as not executable, passing the user's reason to the agent
            return new ToolValidation($args, $responseContent, isExecutable: false, message: $reason, stopSequence: true);
        }

        // Mark the task as executable
        return new ToolValidation($args, $responseContent, isExecutable: true);
    }

    public function execute(array $args, array $context, array $responseContent, AgentIO $agentIO): ToolResponse
    {  
        // Ensure the file path starts with "/"
        if (!str_starts_with($args['file_path'], '/')) {
            $args['file_path'] = '/' . $args['file_path'];
        }

        // Notify the user that the file is being saved
        $agentIO->text(\sprintf('Saving the %s file', $args['file_path']));

        // Create the directory if it doesn't exist
        $dir = dirname($args['file_path']);
        if (!is_dir($dir)) {
            mkdir($dir, 0777, true);
        }

        // Save the file content
        file_put_contents($args['file_path'], trim($args['content']));

        return new ToolResponse($responseContent, message: 'File has been saved.', stopRun: false);
    }
}
```

**Step 3 :** write the `/prompts/code_writter.txt` system prompt
```
You are an agent responsible for creating or updating user-requested files.  

- **For file updates:**  
  1. Use the **"get_file_content"** function to retrieve the initial content.  
  2. Use the **"save_file"** function to save the updated file.  

- **For new file creation:**  
  - Directly use the **"save_file"** function to save the new file.  

Once all files are created or updated, or if additional information is needed, use the **"chat_with_user"** function.
```

**Step 4 :** create `CodeWritterCommand`
```php
<?php

namespace App\Command;

use ArnaudDelgerie\AiToolAgent\Util\AgentIO;
use Symfony\Component\Console\Command\Command;
use ArnaudDelgerie\AiToolAgent\Enum\ClientEnum;
use Symfony\Component\Console\Style\SymfonyStyle;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use ArnaudDelgerie\AiToolAgent\Util\ToolAgentProvider;
use ArnaudDelgerie\AiToolAgent\Util\Config\AgentConfig;
use App\ToolFunctionManager\SaveFileToolFunctionManager;
use ArnaudDelgerie\AiToolAgent\Util\Config\ClientConfig;
use App\ToolFunctionManager\GetFileContentToolFunctionManagager;
use Symfony\Component\DependencyInjection\ParameterBag\ParameterBagInterface;
use ArnaudDelgerie\AiToolAgent\ToolFunctionManager\ConsoleChatWithUserToolFunctionManager;

#[AsCommand(name: 'code-writer')]
class CodeWriterCommand extends Command
{
    public function __construct(private ParameterBagInterface $params, private ToolAgentProvider $toolAgentProvider)
    {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);
        $io->title('CODE WRITER');

        // API key for authentication (ensure secure handling in production)
        $apiKey = 'Your client API key (OpenAI, Mistral, or Anthropic)';

        // Configure the AI client using the selected model
        $clientConfig = new ClientConfig(ClientEnum::Mistral, $apiKey, 'codestral-latest');

        // Load the system prompt for the code writer agent
        $systemPrompt = file_get_contents($this->params->get('kernel.project_dir') . '/prompts/code_writter.txt');

        // Define functions that the agent can use
        $functionNames = [
            GetFileContentToolFunctionManagager::getName(),
            SaveFileToolFunctionManager::getName(),
            ConsoleChatWithUserToolFunctionManager::getName(),
        ];

        // Initialize the tool agent with the system prompt and function configuration
        $agentConfig = new AgentConfig($systemPrompt, $functionNames);
        $consoleToolAgent = $this->toolAgentProvider->createConsoleToolAgent($clientConfig, $agentConfig);

        // Initialize agent I/O for user interaction
        $agentIo = new AgentIO($io, 'Code writer agent');

        // Prompt the user for input
        $userPrompt = $agentIo->ask('What do you want to do?', 'stop');

        // Exit if the user types 'stop'
        if ('stop' === $userPrompt) {
            return Command::SUCCESS;
        }

        // Process user input and execute the agent
        $response = $consoleToolAgent->addUserMessage($userPrompt)->run($agentIo);

        // Log API usage details
        $agentIo->logUsage($response->usageReport);

        return Command::SUCCESS;
    }
}
```

## Demo :
We'll ask `ConsoleToolAgent` to add `docBlocks` to this file
```php
<?php

namespace App\Util;

class CaseConverter
{
    public static function camelCaseToSnakeCase(string $input): string
    {
        return strtolower(preg_replace('/([a-z])([A-Z])/', '$1_$2', $input));
    }
}
```

In this demo, we have deliberately provided an incomplete file path so that the agent cannot find the file.
![Demo 1](../../../img/demo_console_example_2(1).png)

We specify in which folder the file is located.
![Demo 2](../../../img/demo_console_example_2(2).png)

The agent displays the content of the modified file but we reject this modification and ask it to add the doc blocks to the class.
![Demo 3](../../../img/demo_console_example_2(3).png)

The agent displays the new content of the modified file, we can now validate and save the changes.
![Demo 4](../../../img/demo_console_example_2(4).png)