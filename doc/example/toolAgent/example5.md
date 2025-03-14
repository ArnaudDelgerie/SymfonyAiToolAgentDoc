# Example 5 (txt) : 2 functions / multiple function calls

## Purpose :
Our goal is to assign one or multiple topics and extract relevant keywords from the user's content.

## Implementation :
We will use `TopicClassificationToolFunctionManager` from Example 2 and modify `ExtractKeywordToolFunctionManager`.

**Step 1 :** update `ExtractKeywordToolFunctionManager` 
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
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
            ->setDescription('Extracts a keyword from the given content.')
            ->addProperty(
                'keyword',
                (new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::String)
                    ->setDescription('The extracted keyword.')
            );
    }

    public function execute(array $args, array $context, array $responseContent): ToolResponse
    {        
        // Ensure the response array contains an entry for extracted keywords
        if (!isset($responseContent[self::getName()])) {
            $responseContent[self::getName()] = [];
        }
        
        // Append the extracted keyword to the response
        $responseContent[self::getName()][] = $args['keyword'];

        // Return the response with a confirmation message, allowing further calls if needed
        return new ToolResponse($responseContent, \sprintf("The keyword '%s' has been extracted.", $args['keyword']), false);
    }
}
```

**Step 2 :** write  `/prompts/natural_language.txt` system prompt
```
You are a natural language processing agent. For each content analysis, assign one or more topics and extract relevant keywords.
- Call the "topic_classification" function once for each topic you add.
- Call the "extract_keyword" function once for each keyword you extract.
- Identify only the most essential words or phrases that define the core meaning of the content. Focus on unique, subject-specific terms while eliminating generic or supporting words. The selected keywords should instantly convey the main topic.
- Once all topics are assigned and keywords extracted, call the "tasks_completed" function.
- Group all function calls ("topic_classification", "extract_keyword", and "tasks_completed") into a single message.
```

**Step 3:**  update `NaturalLanguage`
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
use ArnaudDelgerie\AiToolAgent\ToolFunctionManager\TasksCompletedToolFunctionManager;

class NaturalLanguage
{
    public function __construct(
        private TopicRepository        $topicRepo,
        private ParameterBagInterface  $params,
        private ToolAgentProvider      $toolAgentProvider
    ) {}

    private function analyze(string $content): array
    {
        // API key for the client (ensure secure handling in production)
        $apiKey = 'Your client API key (OpenAI, Mistral, or Anthropic)';
        
        // Configure the AI client with model selection and parameters
        $clientConfig = new ClientConfig(ClientEnum::Mistral, $apiKey, 'mistral-small-latest', 0.1);
        
        // Load the system prompt for natural language processing
        $systemPrompt = file_get_contents($this->params->get('kernel.project_dir') . '/prompts/natural_language.txt');
        
        // Define functions to be used by the tool agent
        $functionNames = [
            TopicClassificationToolFunctionManager::getName(),
            ExtractKeywordToolFunctionManager::getName(),
            TasksCompletedToolFunctionManager::getName(),
        ];
        
        // Provide available topics as context
        $context = ['availableTopics' => $this->topicRepo->getTopicSlugs()];
        
        // Initialize the tool agent with configuration
        $agentConfig = new AgentConfig($systemPrompt, $functionNames, $context);
        $toolAgent = $this->toolAgentProvider->createToolAgent($clientConfig, $agentConfig);
        
        // Execute content analysis
        $response = $toolAgent->addUserMessage($content)->run();
        
        /*
        Example response content:
        [
          "topic_classification" => [ 
            ["topic" => "artificial-intelligence", "prediction_confidence" => 9],
            ["topic" => "large-language-model", "prediction_confidence" => 9],
          ],
          "extract_keyword" => [
            "Function calling",
            "AI models",
            "LLMs",
            "external functions",
            "real-time data"
          ]
        ]
        */

        /*
        Example API usage report:
        [
          "nbRequest" => 1, 
          "promptTokens" => 545, 
          "completionTokens" => 202, 
          "totalTokens" => 747
        ]
        */

        return $response->content;
    }
}
```