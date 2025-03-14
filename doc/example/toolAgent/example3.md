# Example 3 (txt) : 1 advanced function / 1 function call

## Purpose :
Our goal is to assign one or multiple topics to the user's content.

## Implementation :
We will create the `TopicClassificationToolFunctionManager` class with a `topics` property, which will be an array of objects.

**Step 1 :** create `TopicClassificationToolFunctionManager` 
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
        /*
        Define the function using a nested structure:
          - The 'topics' property is an array containing objects.
          - Each object has two properties: 'topic' (string) and 'prediction_confidence' (number).
          - Instead of setting a description for the object itself, we describe each of its properties.
        */
        return (new ToolFunction())
            ->setName(self::getName())
            ->setDescription('Categorizes content into predefined topics')
            ->addProperty(
                'topics',
                (new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::Array)
                    ->setDescription('A list of topics relevant to the content')
                    ->setArrayItemProperty(
                        (new ToolFunctionProperty())
                            ->setType(ToolFunctionPropertyTypeEnum::Object)
                            ->addObjectProperty(
                                'topic',
                                (new ToolFunctionProperty())
                                    ->setType(ToolFunctionPropertyTypeEnum::String)
                                    ->setDescription('The topic name')
                                    ->setEnum($context['availableTopics'])
                            )
                            ->addObjectProperty(
                                'prediction_confidence',
                                (new ToolFunctionProperty())
                                    ->setType(ToolFunctionPropertyTypeEnum::Number)
                                    ->setDescription('Confidence score (0-10) indicating classification certainty')
                            )
                    )
            );
    }

    public function execute(array $args, array $context, array $responseContent): ToolResponse
    {        
        // Store classified topics in the response
        $responseContent[self::getName()] = $args['topics'];

        // Return response with stopRun set to true
        return new ToolResponse($responseContent, "", true);
    }
}
```

**Step 2 :** write  `/prompts/topic_classification.txt` system prompt
```
You are a topic classification agent, for each content to analyze, you can add one or more topics.
You call the 'topic_classification' function only once to add all the topics
```

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
        // API key for the client (replace with a secure method in production)
        $apiKey = 'Your client API key (OpenAI, Mistral, or Anthropic)';
        
        // Configure the AI client with model selection and parameters
        $clientConfig = new ClientConfig(ClientEnum::Mistral, $apiKey, 'mistral-small-latest', 0.1);
        
        // Load the system prompt for topic classification
        $systemPrompt = file_get_contents($this->params->get('kernel.project_dir') . '/prompts/topic_classification.txt');
        
        // Define the function to be used by the tool agent
        $functionNames = [TopicClassificationToolFunctionManager::getName()];
        
        // Provide available topics as context
        $context = ['availableTopics' => $this->topicRepo->getTopicSlugs()];
        
        // Initialize the tool agent with configuration
        $agentConfig = new AgentConfig($systemPrompt, $functionNames, $context);
        $toolAgent = $this->toolAgentProvider->createToolAgent($clientConfig, $agentConfig);
        
        // Execute topic classification
        $response = $toolAgent->addUserMessage($content)->run();
        
        /*
        Example response content:
        [
          "topic_classification" => [ 
            ["topic" => "artificial-intelligence", "prediction_confidence" => 10],
            ["topic" => "large-language-model", "prediction_confidence" => 10]
          ]
        ]
        */

        /*
        Example API usage report:
        [
          "nbRequest" => 1, 
          "promptTokens" => 338, 
          "completionTokens" => 57, 
          "totalTokens" => 395
        ]
        */

        return $response->content[TopicClassificationToolFunctionManager::getName()] ?? [];
    }

    public function setPostTopics(Post $post): void
    {
        // Retrieve classified topics
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