# ToolFunction

## Overview
The `ArnaudDelgerie\AiToolAgent\DTO\ToolFunction` class allows you to define a function that can be used by a `ToolAgent` or a `ConsoleToolAgent`.
```php
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunction;
use ArnaudDelgerie\AiToolAgent\DTO\ToolFunctionProperty;
use ArnaudDelgerie\AiToolAgent\Enum\ToolFunctionPropertyTypeEnum;

(new ToolFunction())

	// Ensure the function name matches the one returned by ToolFunctionManager::getName() or ConsoleToolFunctionManager::getName() parent.
    ->setName('function_name')
        
    // Provide a description to help the tool agent determine when to use it
    ->setDescription('function description')
        
    // add string property
    ->addProperty(
        'string_property_name',
        (new ToolFunctionProperty())
            ->setType(ToolFunctionPropertyTypeEnum::String)
        	->setDescription('property description')
        	// add a list of possible values (optional)
        	->setEnum(['value 1', 'value 2'])
    )
    
    // add number property (you can specify whether it should be of type 'int' or 'float' in the property description)
    ->addProperty(
        'number_property_name',
        (new ToolFunctionProperty())
            ->setType(ToolFunctionPropertyTypeEnum::Number)
            ->setDescription('property description')
        	// add a list of possible values (optional)
        	->setEnum([1, 3, 5])
    )
        
    // add object property
    ->addProperty(
        'object_property_name',
        (new ToolFunctionProperty())
            ->setType(ToolFunctionPropertyTypeEnum::Object)
        	->addObjectProperty(
                'first_object_property_name',
                (new ToolFunctionProperty())
                	->setType(ToolFunctionPropertyTypeEnum::String)
        			->setDescription('first object property description')
            )
        	->addObjectProperty(
                'second_object_property_name',
                (new ToolFunctionProperty())
                	->setType(ToolFunctionPropertyTypeEnum::Number)
        			->setDescription('second object property description')
            )
        	// ...
    )
        
    // add array of string
    ->addProperty(
        'array_of_string_property',
        (new ToolFunctionProperty())
            ->setType(ToolFunctionPropertyTypeEnum::Array)
        	->setDescription('property description')
        	->setArrayItemProperty(
            	(new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::String)
                	// add a list of possible values (optional)
        			->setEnum(['value 1', 'value 2'])
            )
    )
        
    // add array of number
    ->addProperty(
        'array_of_number_property',
        (new ToolFunctionProperty())
            ->setType(ToolFunctionPropertyTypeEnum::Array)
        	->setDescription('property description')
        	->setArrayItemProperty(
            	(new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::Number)
                	// add a list of possible values (optional)
        			->setEnum([1, 2])
            )
    )
        
    // add array of object
    ->addProperty(
        'array_of_object_property',
        (new ToolFunctionProperty())
            ->setType(ToolFunctionPropertyTypeEnum::Array)
        	->setDescription('property description')
        	->setArrayItemProperty(
            	(new ToolFunctionProperty())
                    ->setType(ToolFunctionPropertyTypeEnum::Object)
                	->addObjectProperty(
                        'first_object_property_name',
                        (new ToolFunctionProperty())
                            ->setType(ToolFunctionPropertyTypeEnum::String)
                            ->setDescription('first object property description')
                    )
                    ->addObjectProperty(
                        'second_object_property_name',
                        (new ToolFunctionProperty())
                            ->setType(ToolFunctionPropertyTypeEnum::Number)
                            ->setDescription('second object property description')
                    )
            )
    );
```