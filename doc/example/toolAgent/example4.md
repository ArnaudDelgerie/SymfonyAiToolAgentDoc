# Example 4 (txt) : 2 functions / 2 function calls

## Purpose :
Our goal is to assign one or multiple topics and extract relevant keywords from the user's content.

## Implementation :
We will modify `TopicClassificationToolFunctionManager` from Example 3 and introduce `ExtractKeywordToolFunctionManager`.

**Step 1 :** update `TopicClassificationToolFunctionManager` 
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use App\ToolFunctionManager\ExtractKeywordToolFunctionManager;
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
        // Function definition remains the same as in Example 2.
    }

    public function execute(array $args, array $context, array $responseContent): ToolResponse
    {        
        // Store classified topics in the response
        $responseContent[self::getName()] = $args['topics'];

        // Determine if the "run" should stop based on keyword extraction
        $stop = isset($responseContent[ExtractKeywordToolFunctionManager::getName()]);

        return new ToolResponse($responseContent, "Topics have been added", $stop);
    }
}
```

**Step 2 :** create `ExtractKeywordToolFunctionManager` 
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use App\ToolFunctionManager\TopicClassificationToolFunctionManager;
use ArnaudDelgerie\AiToolAgent\Interface\ToolFunctionManagerInterface;

class ExtractKeywordToolFunctionManager implements ToolFunctionManagerInterface
{   
    public static function getName(): string
    {
        return 'extract_keyword';
    }

    public function getToolFunction(array $context): ToolFunction
    {
        return (new ToolFunction())
            ->setName(self::getName())
            ->setDescription('Extracts keywords from content')
            ->addProperty(
                'keywords',
                (new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::Array)
                    ->setDescription('List of extracted keywords')
                    ->setArrayItemProperty(
                        (new ToolFunctionProperty())->setType(ToolFunctionPropertyTypeEnum::String)
                    )
            );
    }

    public function execute(array $args, array $context, array $responseContent): ToolResponse
    {        
        // Store extracted keywords in the response
        $responseContent[self::getName()] = $args['keywords'];

        // Determine if execution should stop based on topic classification completion
        $stop = isset($responseContent[TopicClassificationToolFunctionManager::getName()]);

        return new ToolResponse($responseContent, "Keywords extracted", $stop);
    }
}
```

**Step 3 :** write  `/prompts/natural_language.txt` system prompt
```
You are a natural language processing agent. For each content analysis, first, assign one or more topics by calling the "topic_classification" function once. Then, extract key terms by calling the "extract_keyword" function once.

Focus on the most essential words or phrases that define the core meaning of the content. Select only unique, subject-specific terms, eliminating generic or supporting words. The extracted keywords should instantly convey the main topic.
```

**Step 4:**  create `NaturalLanguage`
```php
<?php

namespace App\Service;

use App\Repository\TopicRepository;
use ArnaudDelgerie\AiToolAgent\Enum\ClientEnum;
use ArnaudDelgerie\AiToolAgent\Util\ToolAgentProvider;
use ArnaudDelgerie\AiToolAgent\Util\Config\AgentConfig;
use ArnaudDelgerie\AiToolAgent\Util\Config\ClientConfig;
use App\ToolFunctionManager\ExtractKeywordToolFunctionManager;
use App\ToolFunctionManager\TopicClassificationToolFunctionManager;
use Symfony\Component\DependencyInjection\ParameterBag\ParameterBagInterface;

class NaturalLanguage
{
    public function __construct(
        private TopicRepository        $topicRepo,
        private ParameterBagInterface  $params,
        private ToolAgentProvider      $toolAgentProvider
    ) {}

    private function analyze(string $content): array
    {
        // API key for the client (replace with a secure method in production)
        $apiKey = 'Your client API key (OpenAI, Mistral, or Anthropic)';
        
        // Configure the AI client with model selection and parameters
        $clientConfig = new ClientConfig(ClientEnum::Mistral, $apiKey, 'mistral-small-latest', 0.1);
        
        // Load the system prompt for natural language analysis
        $systemPrompt = file_get_contents($this->params->get('kernel.project_dir') . '/prompts/natural_language.txt');
        
        // Provide available topics as context
        $context = ['availableTopics' => $this->topicRepo->getTopicSlugs()];
        
        // Define the functions to be used by the tool agent
        $functionNames = [
            TopicClassificationToolFunctionManager::getName(),
            ExtractKeywordToolFunctionManager::getName(),
        ];
        
        // Initialize the tool agent with configuration
        $agentConfig = new AgentConfig($systemPrompt, $functionNames, $context);
        $toolAgent = $this->toolAgentProvider->createToolAgent($clientConfig, $agentConfig);
        
        // Execute content analysis
        $response = $toolAgent->addUserMessage($content)->run();
        
        /*
        Example response content:
        [
          "topic_classification" => [ 
            ["topic" => "artificial-intelligence", "prediction_confidence" => 10],
            ["topic" => "large-language-model", "prediction_confidence" => 10],
          ],
          "extract_keyword" => [
            "Function calling in LLMs",
            "AI models",
            "external functions",
            "structured outputs",
            "real-time data",
            "perform calculations",
            "automate tasks",
            "user queries"
          ]
        ]
        */

        /*
        Example API usage report:
        [
          "nbRequest" => 1, 
          "promptTokens" => 477, 
          "completionTokens" => 110, 
          "totalTokens" => 587
        ]
        */

        return $response->content;
    }
}
```