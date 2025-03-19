# ToolAgent

## Overview
A `ToolAgent` automates tasks like moderation and semantic analysis, processing both text and image content
```php
<?php

namespace App\Service;

use App\Entity\Post;
use Doctrine\ORM\EntityManagerInterface;
use ArnaudDelgerie\AiToolAgent\Enum\ClientEnum;
use ArnaudDelgerie\AiToolAgent\Util\ToolAgentProvider;
use ArnaudDelgerie\AiToolAgent\Util\Config\AgentConfig;
use ArnaudDelgerie\AiToolAgent\Util\Config\ClientConfig;
use App\ToolFunctionManager\ModerationToolFunctionManager;
use App\ToolFunctionManager\TopicClassificationToolFunctionManager;
use Symfony\Component\DependencyInjection\ParameterBag\ParameterBagInterface;
use ArnaudDelgerie\AiToolAgent\ToolFunctionManager\TasksCompletedToolFunctionManager;

class ToolAgentService
{
    public function __construct(
        private EntityManagerInterface $em,
        private ParameterBagInterface  $params,
        private ToolAgentProvider      $toolAgentProvider
    ) {}

    private function execute(Post $post): void
    {
        // API key for the client (replace with a secure method in production)
        $apiKey = 'Your client API key (OpenAI, Mistral, or Anthropic)';
        
        // Configure the AI client with model selection and parameters
        $clientConfig = new ClientConfig(ClientEnum::Mistral, $apiKey, 'mistral-small-latest');
        
        // Load the system prompt
        $systemPrompt = file_get_contents($this->params->get('kernel.project_dir') . '/prompts/my_system_promp.txt');
        
        // Define the function to be used by the tool agent (instance of ArnaudDelgerie\AiToolAgent\Interface\ToolFunctionManagerInterface)
        // To prevent infinite loops, at least one of your `ToolFunctionManager` instances must return a `ToolResponse` with `stopRun` set to `true`
        // If none do, you can use `ArnaudDelgerie\AiToolAgent\ToolFunctionManager\TasksCompletedToolFunctionManager`
        $functionNames = [
            ModerationToolFunctionManager::getName(),
            TopicClassificationToolFunctionManager::getName(),
            TasksCompletedToolFunctionManager::getName(),
        ];
        
        // $context array is optional, it is useful if you want to pass data to your `ToolFunctionManager`(s)
        $context = ['entityClass' => Post::class, 'entityId' => $post->getId()];
        
        // Create agent config
        $agentConfig = new AgentConfig($systemPrompt, $functionNames, $context);
        
        // Create new agent
        $toolAgent = $this->toolAgentProvider->createToolAgent($clientConfig, $agentConfig);
        
        // Execute
        $response = $toolAgent->addUserMessage($content)->run();
        
        /*
        $response->content is an array containing results added to the $responseContent variable across different ToolFunctionManager instances.
        */

        /*
        $response->usageReport returns an ArnaudDelgerie\AiToolAgent\Util\AgentUsageReport instance, providing information  about the number of tokens used by the ToolAgent.
        
        for example : $response->usageReport->toArray()
        [
          "nbRequest" => 1, 
          "promptTokens" => 338, 
          "completionTokens" => 57, 
          "totalTokens" => 395
        ]
        */
        
        /*
        $response->stopReport returns an ArnaudDelgerie\AiToolAgent\Util\AgentStopReport instance, indicating why the ToolAgent stopped.
        
        for example:
        $reponse->stopReport->stopReason = ArnaudDelgerie\AiToolAgent\Enum\StopReasonEnum::Function
        $reponse->stopReport->step = ArnaudDelgerie\AiToolAgent\Enum\StopStepEnum::Execute
        $reponse->stopReport->value = 'tasks_completed' (TasksCompletedToolFunctionManager::getName())
        */
        
    }
}
```