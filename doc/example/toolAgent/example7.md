# Example 2 (txt/img) : 1 function / 1 function call

## Purpose :
Our goal is to identify sensitive data in user content, including text and images.

## Implementation :
**Step 1 :** create `SensitiveDataToolFunctionManager` 
```php
<?php

namespace App\ToolFunctionManager;

use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\Util\ToolResponse;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;
use ArnaudDelgerie\AiToolAgent\Interface\ToolFunctionManagerInterface;

class SensitiveDataToolFunctionManager implements ToolFunctionManagerInterface
{

    public static function getName(): string
    {
        return 'detect_sensitive_data';
    }

    public function getToolFunction(array $context): ToolFunction
    {
        return (new ToolFunction())
            ->setName(self::getName())
            ->setDescription('Use this function to detect sensitive data in user content')
            ->addProperty(
                'email_adress',
                (new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::Boolean)
                    ->setDescription('True if an email address is detected')
            )
            ->addProperty(
                'phone_number',
                (new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::Boolean)
                    ->setDescription('True if a phone number is detected')
            )
            ->addProperty(
                'postal_address',
                (new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::Boolean)
                    ->setDescription('True if a postal address is detected')
            );
    }

    public function execute(array $args, array $context, array $responseContent): ToolResponse
    {   
        $responseContent[self::getName()] = $args;

        return new ToolResponse($responseContent, "", true);
    }
}
```

**Step 2 :** write  `/prompts/sensitive_data.txt` system prompt
```
You are an agent tasked with detecting sensitive data in content, including email addresses, phone numbers, and postal addresses. The content may consist of text, images, or both.
```

**Step 3:**  Create `ModerationService`
```php
<?php

namespace App\Service;

use ArnaudDelgerie\AiToolAgent\Enum\ClientEnum;
use ArnaudDelgerie\AiToolAgent\DTO\MessageImage;
use ArnaudDelgerie\AiToolAgent\Enum\ImageTypeEnum;
use ArnaudDelgerie\AiToolAgent\Util\ToolAgentProvider;
use ArnaudDelgerie\AiToolAgent\Util\Config\AgentConfig;
use ArnaudDelgerie\AiToolAgent\Util\Config\ClientConfig;
use App\ToolFunctionManager\SensitiveDataToolFunctionManager;
use Symfony\Component\DependencyInjection\ParameterBag\ParameterBagInterface;

class ImageService
{
    public function __construct(
        private ParameterBagInterface  $params,
        private ToolAgentProvider      $toolAgentProvider
    ) {}

    private function generateAlternativeText(string $content, string $imgPath): array
    {
        // Example inputs:
        // $content: "Contact me for more information"
        // $imgPath: Path to an image (e.g., a business card with email, phone number, and postal address)
        
        // API key for authentication (ensure secure handling in production)
        $apiKey = 'Your client API key (OpenAI, Mistral, or Anthropic)';

        // Configure the AI client using a vision-enabled model
        $clientConfig = new ClientConfig(ClientEnum::Openai, $apiKey, 'gpt-4o');

        // Load the system prompt for sensitive data detection
        $systemPrompt = file_get_contents($this->params->get('kernel.project_dir') . '/prompts/sensitive_data.txt');

        // Define the function used for detecting sensitive data
        $functionNames = [SensitiveDataToolFunctionManager::getName()];

        // Create agent config with the system prompt and function configuration
        $agentConfig = new AgentConfig($systemPrompt, $functionNames);

        // Convert the image to base64 for processing
        $messageImgs = [new MessageImage(ImageTypeEnum::JPEG, base64_encode(file_get_contents($imgPath)))];

        // Create and execute the tool agent
        $toolAgent = $this->toolAgentProvider->createToolAgent($clientConfig, $agentConfig);
        $response = $toolAgent->addUserMessage($content, $messageImgs)->run();

        /*
        Example response content:
        [
          "detect_sensitive_data" => [ 
            "email_address" => true,
            "phone_number" => true,
            "postal_address" => true
          ]
        ]
        */

        /*
        Example API usage report:
        [
          "nbRequest" => 1, 
          "promptTokens" => 891, 
          "completionTokens" => 71, 
          "totalTokens" => 962
        ]
        */

        return $response->content;
    }
}
```