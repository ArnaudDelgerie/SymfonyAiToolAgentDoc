# Example 2 (txt) : 1 basic function / multiple function calls

## Purpose :
Our goal is to assign one or multiple topics to the user's content.

## Implementation :
We will use a slightly modified version of `TopicClassificationToolFunctionManager` from Example 1, instructing the tool agent to call it separately for each topic it assigns to the content.
Multiple calls don’t necessarily mean multiple requests, as most models support **parallel tool calls**, allowing them to invoke multiple functions in a single response. However, it’s uncertain whether one or multiple requests will be required in advance. Therefore, we must account for scenarios where multiple requests are needed.

**Step 1 :** create `TopicClassificationToolFunctionManager` (almost identical to that of example 1 except some modifications made in the execute method)
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ToolFunctionManagerInterface;

class TopicClassificationToolFunctionManager implements ToolFunctionManagerInterface
{   
    public static function getName(): string
    {
        return 'topic_classification';
    }

    public function getToolFunction(array $context): ToolFunction
    {
        // No changes needed; reusing logic from example 1.
    }

    public function execute(array $args, array $context, array $responseContent): ToolResponse
    {
        // Ensure the responseContent array has an entry for this function
        if (!isset($responseContent[self::getName()])) {
            $responseContent[self::getName()] = [];
        }
        
        // Append the result to responseContent
        $responseContent[self::getName()][] = $args;
        
        // Set the message to confirm the task was completed (useful if multiple requests are required)
        // Keep stopRun as false to allow the tool agent to call the function again if needed
        return new ToolResponse($responseContent, \sprintf("Topic '%s' has been added to the content.", $args['topic']), false);
    }
}
```

**Step 2 :** write the `/prompts/topic_classification.txt` system prompt
We modify the prompt from Example 1 to explicitly inform the tool agent that it can invoke the `topic_classification` function multiple times.
```
You are a topic classification agent. For each content analysis, you can assign one or more topics. Call the "topic_classification" function once for each topic you add (e.g., for three topics, call the function three times).

After adding all topics, call the "tasks_completed" function to signal completion.

Ensure all function calls ("topic_classification" and "tasks_completed") are grouped within a single message.
```
In this example, the "run" does not stop at the end of the `execute` method in the `TopicClassificationToolFunctionManager` class. Instead, we instruct the tool agent to call the `tasks_completed` function after adding topics to prevent infinite call loops. The `TasksCompletedToolFunctionManager` class, provided by the bundle, is designed to stop the "run" when the tool agent performs multiple function calls without knowing the exact number in advance. While you can create your own stop function, a tool agent **MUST** have a mechanism to stop execution.
Additionally, we instruct the tool agent to group function calls into a single message to minimize the number of requests (and reduce costs). Without this, models tend to separate calls by function—for example, sending one message with all `topic_classification` calls and another with the `tasks_completed` call. However, even with this instruction, some models may still fail to group function calls.

**Step 3:**  create `TopicClassifier`
```php
<?php

namespace App\Service;

use App\Entity\Post;
use App\Repository\TopicRepository;
use Doctrine\ORM\EntityManagerInterface;
use ArnaudDelgerie\AiToolAgent\Enum\ClientEnum;
use ArnaudDelgerie\AiToolAgent\Util\ToolAgentProvider;
use ArnaudDelgerie\AiToolAgent\Util\Config\AgentConfig;
use ArnaudDelgerie\AiToolAgent\Util\Config\ClientConfig;
use App\ToolFunctionManager\TopicClassificationToolFunctionManager;
use Symfony\Component\DependencyInjection\ParameterBag\ParameterBagInterface;
use ArnaudDelgerie\AiToolAgent\ToolFunctionManager\TasksCompletedToolFunctionManager;

class TopicClassifier
{
    public function __construct(
        private EntityManagerInterface $em,
        private TopicRepository        $topicRepo,
        private ParameterBagInterface  $params,
        private ToolAgentProvider      $toolAgentProvider
    ) {}

    private function getContentTopics(string $content): array
    {
        $apiKey = 'Your client API key (OpenAI, Mistral, or Anthropic)';
        
        // Limit the number of requests to prevent hallucinations
        // In this case, the task is simple, and most models should handle it reliably
        $clientConfig = new ClientConfig(ClientEnum::Mistral, $apiKey, 'mistral-small-latest', 0.1, requestLimit: 5);
        
        // Load the system prompt for topic classification
        $systemPrompt = file_get_contents($this->params->get('kernel.project_dir') . '/prompts/topic_classification.txt');
        
        // Include 'tasks_completed' function to allow the tool agent to determine when to stop
        $functionNames = [
            TopicClassificationToolFunctionManager::getName(),
            TasksCompletedToolFunctionManager::getName(),
        ];
        
        // Define available topics from the database
        $context = ['availableTopics' => $this->topicRepo->getTopicSlugs()];
        
        // Configure and create the tool agent
        $agentConfig = new AgentConfig($systemPrompt, $functionNames, $context);
        $toolAgent = $this->toolAgentProvider->createToolAgent($clientConfig, $agentConfig);
        
        // Execute classification
        $response = $toolAgent->addUserMessage($content)->run();
        
        /*
        Example response content:
        [
          "topic_classification" => [ 
            ["topic" => "artificial-intelligence", "prediction_confidence" => 9],
            ["topic" => "large-language-model", "prediction_confidence" => 9],
          ],
          "tasks_completed" => "The content was classified into 2 topics: artificial-intelligence and large-language-model"
        ]
        */

        /*
        Example API usage report:
        [
          "nbRequest" => 1, 
          "promptTokens" => 430, 
          "completionTokens" => 93, 
          "totalTokens" => 523
        ]
        - Task was completed in one request.
        - More prompt tokens were used due to two functions being included.
        - More completion tokens were used since there were three function calls (2 for topic classification, 1 for tasks_completed).
        */
        
		/*
        Analyze stop report for debugging:
        If stopped by 'tasks_completed' function (expected behavior):
          - $response->stopReport->stopReason === StopReasonEnum::Function
          - $response->stopReport->value === 'tasks_completed'
        If stopped due to request limit (potential issue):
          - $response->stopReport->stopReason === StopReasonEnum::RequestLimit
          - $response->stopReport->value === 5 (matches requestLimit in ClientConfig)
		*/
        return $response->content[TopicClassificationToolFunctionManager::getName()] ?? [];
    }

    public function setPostTopics(Post $post): void
    {
        // Retrieve classified topics from content
        $results = $this->getContentTopics($post->getContent());

        // Associate each classified topic with the post
        foreach ($results as $result) {
            $topic = $this->topicRepo->findOneBy(['slug' => $result['topic']]);
            $relatedTopic = new RelatedTopic();
            $relatedTopic->setTopic($topic)->setTopicConfidence((int)$result['prediction_confidence']);
            $post->addRelatedTopic($relatedTopic);
        }
        
        // Persist changes
        $this->em->flush();
    }
}
```