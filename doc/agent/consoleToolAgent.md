# ConsoleToolAgent

## Overview
A `ConsoleToolAgent` is an interactive agent that performs tasks via Symfony console using custom commands.
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
use ArnaudDelgerie\AiToolAgent\Util\Config\ClientConfig;
use Symfony\Component\DependencyInjection\ParameterBag\ParameterBagInterface;
use ArnaudDelgerie\AiToolAgent\ToolFunctionManager\ConsoleChatWithUserToolFunctionManager;

#[AsCommand(name: 'chat:bot')]
class ChatBotCommand extends Command
{
    public function __construct(private ToolAgentProvider $toolAgentProvider, private ParameterBagInterface $params)
    {
        parent::__construct();
    }

    // Peux tu me generer un PostForm pour l'entité Post avec les champs title, teaser and content
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);

        $io->title('CHAT BOT');

        // API key for the client (replace with a secure method in production)
        $apiKey = 'Your client API key (OpenAI, Mistral, or Anthropic)';
        
        // Configure the AI client with model selection and parameters
        $clientConfig = new ClientConfig(ClientEnum::Mistral, $apiKey, 'mistral-small-latest');
        
        // Load the system prompt
        $systemPrompt = file_get_contents($this->params->get('kernel.project_dir') . '/prompts/my_system_promp.txt');
        
        // Define the function to be used by the tool agent (instance of ArnaudDelgerie\AiToolAgent\Interface\ConsoleToolFunctionManagerInterface)
        // To prevent infinite loops, at least one of your `ConsoleToolFunctionManager` instances must return a `ToolResponse` with `stopRun` set to `true`
        // If none do, you can use `ArnaudDelgerie\AiToolAgent\ToolFunctionManager\ConsoleTasksCompletedToolFunctionManager`
        // or `ArnaudDelgerie\AiToolAgent\ToolFunctionManager\ConsoleChatWithUserToolFunctionManager`
        $functionNames = [
            ConsoleChatWithUserToolFunctionManager::getName(),
        ];
        
        // Create agent config
        $agentConfig = new AgentConfig($systemPrompt, $functionNames, $context);

        // Create ConsoleToolAgent
        $consoleToolAgent = $this->toolAgentProvider->createConsoleToolAgent($clientConfig, $agentConfig);

        // Create agent I/O
        $agentIo = new AgentIO($io, 'Chat bot');
        
        // Retrieve user prompt
        $userPrompt = $agentIo->ask('What do you want to do ?', 'stop');

        if ('stop' === $userPrompt) {
            return Command::SUCCESS;
        }

        // Execute
        $response = $consoleToolAgent->addUserMessage($userPrompt)->run($agentIo);

        // Display usage report
        $agentIo->logUsage($response->usageReport);

        return Command::SUCCESS;
    }
}
```