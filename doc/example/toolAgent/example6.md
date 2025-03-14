# Example 6 (img) : 1 function / multiple function calls

## Purpose :
Our goal is to generate alternative text for one or multiple images.

## Implementation :
**Step 1 :** create `ImageAltToolFunctionManager` 
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ToolFunctionManagerInterface;

class ImageAltToolFunctionManager implements ToolFunctionManagerInterface
{   
    public static function getName(): string
    {
        return 'add_alternative_text_to_img';
    }

    public function getToolFunction(array $context): ToolFunction
    {
        return (new ToolFunction())
            ->setName(self::getName())
            ->setDescription('Use this function to add an alternative text to an image')
            ->addProperty(
                'alternate_text',
                (new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::String)
                    ->setDescription('Short alternative text')
            )
            ->addProperty(
                'image_index',
                (new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::Number)
                    ->setDescription('Index of image (integer start from 0)')
            );
    }

    public function execute(array $args, array $context, array $responseContent): ToolResponse
    {        
        if(!isset($responseContent[self::getName()])) {
            $responseContent[self::getName()] = [];
        }
        
        $responseContent[self::getName()][] = $args;

        return new ToolResponse($responseContent, \sprintf("the alternative text %s has been extracted for image %s", $args['alternate_text'], (string)$args['image_index']), false);
    }
}
```

**Step 2 :** write  `/prompts/alternative_text.txt` system prompt
```
You're an agent who has to add an alternate text for one or more images.
You call the 'add_alternative_text_to_img' function for each image supplied by the user.
Once all the alternates text are added, call the 'tasks_completed' function.
Group all function calls (add_alternative_text_to_img tasks_completed) in the same message
```

**Step 3:**  Create `ImageService`
```php
<?php

namespace App\Service;

use ArnaudDelgerie\AiToolAgent\Enum\ClientEnum;
use ArnaudDelgerie\AiToolAgent\DTO\MessageImage;
use ArnaudDelgerie\AiToolAgent\Enum\ImageTypeEnum;
use ArnaudDelgerie\AiToolAgent\Util\ToolAgentProvider;
use ArnaudDelgerie\AiToolAgent\Util\Config\AgentConfig;
use ArnaudDelgerie\AiToolAgent\Util\Config\ClientConfig;
use App\ToolFunctionManager\ImageAltToolFunctionManager;
use Symfony\Component\DependencyInjection\ParameterBag\ParameterBagInterface;
use ArnaudDelgerie\AiToolAgent\ToolFunctionManager\TasksCompletedToolFunctionManager;

class ImageService
{
    public function __construct(
        private ParameterBagInterface  $params,
        private ToolAgentProvider      $toolAgentProvider
    ) {}

    private function generateAlternativeText(array $imgPaths): array
    {
        // API key for the client (ensure secure handling in production)
        $apiKey = 'Your client API key (OpenAI, Mistral, or Anthropic)';

        // Configure AI client with vision-enabled model
        $clientConfig = new ClientConfig(ClientEnum::Openai, $apiKey, 'gpt-4o');

        // Load the system prompt for generating alternative text
        $systemPrompt = file_get_contents($this->params->get('kernel.project_dir') . '/prompts/alternative_text.txt');

        // Define functions to be used by the tool agent
        $functionNames = [
            ImageAltToolFunctionManager::getName(),
            TasksCompletedToolFunctionManager::getName(),
        ];

        // Create agent config with system prompt and function configuration
        $agentConfig = new AgentConfig($systemPrompt, $functionNames);

        // Convert images to base64 format for processing
        $messageImgs = [];
        foreach ($imgPaths as $imgPath) {
            $base64 = base64_encode(file_get_contents($imgPath));
            
            // Specify the correct image format
            $messageImgs[] = new MessageImage(ImageTypeEnum::JPEG, $base64);
        }

        // Create and execute the tool agent
        $toolAgent = $this->toolAgentProvider->createToolAgent($clientConfig, $agentConfig);
        $response = $toolAgent->addUserMessage(null, $messageImgs)->run();

        /*
        Example response content:
        [
          "add_alternative_text_to_img" => [ 
            [
              "alternate_text" => "Person wearing a hat taking a photo with a camera outdoors.",
              "image_index" => 0
            ],
            [
              "alternate_text" => "Business card with contact information for a photographer named John Doe.",
              "image_index" => 1
            ]
          ],
          "tasks_completed" => "Added alternate text to two images: one of a person taking a photo and another of a business card for a photographer."
        ]
        */

        /*
        Example API usage report:
        [
          "nbRequest" => 1, 
          "promptTokens" => 1202, 
          "completionTokens" => 122, 
          "totalTokens" => 1324
        ]
        */

        return $response->content;
    }
}
```