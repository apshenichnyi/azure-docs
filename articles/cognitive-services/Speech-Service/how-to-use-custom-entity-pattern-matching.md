---
title: How to use custom entity pattern matching with the C++ Speech SDK
titleSuffix: Azure Cognitive Services
description: In this guide, you learn how to recognize intents and custom entities from simple patterns.
services: cognitive-services
author: chschrae
manager: travisw
ms.service: cognitive-services
ms.subservice: speech-service
ms.topic: conceptual
ms.date: 11/15/2021
ms.author: chschrae
ms.devlang: cpp
ms.custom: devx-track-cpp
---

# How to use custom entity pattern matching with the C++ Speech SDK

The Cognitive Services [Speech SDK](speech-sdk.md) has a built-in feature to provide **intent recognition** with **simple language pattern matching**. An intent is something the user wants to do: close a window, mark a checkbox, insert some text, etc.

In this guide, you use the Speech SDK to develop a C++ console application that derives intents from speech utterances spoken through your device's microphone. You'll learn how to:

> [!div class="checklist"]
>
> - Create a Visual Studio project referencing the Speech SDK NuGet package
> - Create a speech configuration and get an intent recognizer
> - Add intents and patterns via the Speech SDK API
> - Add custom entities via the Speech SDK API
> - Use asynchronous, event-driven continuous recognition

## When should you use this?

Use this sample code if: 
* You are only interested in matching very strictly what the user said. These patterns match more aggressively than LUIS.
* You do not have access to a [LUIS](../LUIS/index.yml) app, but still want intents. This can be helpful since it is embedded within the SDK.
* You cannot or do not want to create a LUIS app but you still want some voice-commanding capability.

If you do not have access to a [LUIS](../LUIS/index.yml) app, but still want intents, this can be helpful since it is embedded within the SDK.


## Prerequisites

Be sure you have the following items before you begin this guide:

- A [Cognitive Services Azure resource](https://ms.portal.azure.com/#create/Microsoft.CognitiveServicesSpeechServices) or a [Unified Speech resource](https://ms.portal.azure.com/#create/Microsoft.CognitiveServicesSpeechServices)
- [Visual Studio 2019](https://visualstudio.microsoft.com/downloads/) (any edition).

## Pattern Matching Model overview

[!INCLUDE [Pattern Matching Overview](includes/pattern-matching-overview.md)]

## Create a speech project in Visual Studio

[!INCLUDE [Create project](../../../includes/cognitive-services-speech-service-quickstart-cpp-create-proj.md)]

## Open your project in Visual Studio

Next, open your project in Visual Studio.

1. Launch Visual Studio 2019.
2. Load your project and open `helloworld.cpp`.

## Start with some boilerplate code

Let's add some code that works as a skeleton for our project.

```cpp
    #include <iostream>
    #include <speechapi_cxx.h>

    using namespace Microsoft::CognitiveServices::Speech;
    using namespace Microsoft::CognitiveServices::Speech::Intent;

    int main()
    {
        std::cout << "Hello World!\n";

        auto config = SpeechConfig::FromSubscription("YOUR_SUBSCRIPTION_KEY", "YOUR_SUBSCRIPTION_REGION");
    }
```

## Create a Speech configuration

Before you can initialize an `IntentRecognizer` object, you need to create a configuration that uses the key and Azure region for your Cognitive Services prediction resource.

* Replace `"YOUR_SUBSCRIPTION_KEY"` with your Cognitive Services prediction key.
* Replace `"YOUR_SUBSCRIPTION_REGION"` with your Cognitive Services resource region.

This sample uses the `FromSubscription()` method to build the `SpeechConfig`. For a full list of available methods, see [SpeechConfig Class](/cpp/cognitive-services/speech/speechconfig).

## Initialize an IntentRecognizer

Now create an `IntentRecognizer`. Insert this code right below your Speech configuration.

```cpp
    auto intentRecognizer = IntentRecognizer::FromConfig(config);
```

## Add some intents

You need to associate some patterns with a `PatternMatchingModel` and apply it to the `IntentRecognizer`.
We will start by creating a `PatternMatchingModel` and adding a few intents to it. A PatternMatchingIntent is a struct so we will just use the in-line syntax.

> [!Note]
> We can add multiple patterns to an `Intent`.

```cpp
    auto model = PatternMatchingModel::FromId("myNewModel");
    
    model->Intents.push_back({"Take me to floor {floorName}.", "Go to floor {floorName}."} , "ChangeFloors");
    model->Intents.push_back({"{action} the door."}, "OpenCloseDoor");
```

## Add some custom entities

To take full advantage of the pattern matcher you can customize your entities. We will make "floorName" a list of the available floors.

```cpp
    model->Entities.push_back({ "floorName" , Intent::EntityType::List, Intent::EntityMatchMode::Strict, {"one","two", "lobby", "ground floor"} });
```

## Apply our model to the Recognizer

Now it is necessary to apply the model to the `IntentRecognizer`. It is possible to use multiple models at once so the API takes a collection of models.

```cpp
    std::vector<std::shared_ptr<LanguageUnderstandingModel>> collecton;
    
    collection.push_back(model);
    intentRecognizer->ApplyLanguageModels(collection);
```

## Recognize an intent

From the `IntentRecognizer` object, you're going to call the `RecognizeOnceAsync()` method. This method asks the Speech service to recognize speech in a single phrase, and stop recognizing speech once the phrase is identified. For simplicity we'll wait on the future returned to complete.

Insert this code below your intents:

```cpp
    std::cout << "Say something ..." << std::endl;
    auto result = intentRecognizer->RecognizeOnceAsync().get();
```

## Display the recognition results (or errors)

When the recognition result is returned by the Speech service, let's just print the result.

Insert this code below `auto result = intentRecognizer->RecognizeOnceAsync().get();`:

```cpp
switch (result->Reason)
{
case ResultReason::RecognizedSpeech:
        std::cout << "RECOGNIZED: Text = " << result->Text.c_str() << std::endl;
        std::cout << "NO INTENT RECOGNIZED!" << std::endl;
        break;
case ResultReason::RecognizedIntent:
    std::cout << "RECOGNIZED: Text = " << result->Text.c_str() << std::endl;
    std::cout << "  Intent Id = " << result->IntentId.c_str() << std::endl;
    auto entities = result->GetEntities();
    if (entities.find("floorName") != entities.end())
    {
        std::cout << "  Floor name: = " << entities["floorName"].c_str() << std::endl;
    }

    if (entities.find("action") != entities.end())
    {
        std::cout << "  Action: = " << entities["action"].c_str() << std::endl;
    }

    break;
case ResultReason::NoMatch:
{
    auto noMatch = NoMatchDetails::FromResult(result);
    switch (noMatch->Reason)
    {
    case NoMatchReason::NotRecognized:
        std::cout << "NOMATCH: Speech was detected, but not recognized." << std::endl;
        break;
    case NoMatchReason::InitialSilenceTimeout:
        std::cout << "NOMATCH: The start of the audio stream contains only silence, and the service timed out waiting for speech." << std::endl;
        break;
    case NoMatchReason::InitialBabbleTimeout:
        std::cout << "NOMATCH: The start of the audio stream contains only noise, and the service timed out waiting for speech." << std::endl;
        break;
    case NoMatchReason::KeywordNotRecognized:
        std::cout << "NOMATCH: Keyword not recognized" << std::endl;
        break;
    }
    break;
}
case ResultReason::Canceled:
{
    auto cancellation = CancellationDetails::FromResult(result);

    if (!cancellation->ErrorDetails.empty())
    {
        std::cout << "CANCELED: ErrorDetails=" << cancellation->ErrorDetails.c_str() << std::endl;
        std::cout << "CANCELED: Did you update the subscription info?" << std::endl;
    }
}
default:
    break;
}
```

## Check your code

At this point, your code should look like this:

```cpp
#include <iostream>
#include <speechapi_cxx.h>

using namespace Microsoft::CognitiveServices::Speech;
using namespace Microsoft::CognitiveServices::Speech::Intent;

int main()
{
    auto config = SpeechConfig::FromSubscription("YOUR_SUBSCRIPTION_KEY", "YOUR_SUBSCRIPTION_REGION");
    auto intentRecognizer = IntentRecognizer::FromConfig(config);

    auto model = PatternMatchingModel::FromId("myNewModel");
    
    model->Intents.push_back({"Take me to floor {floorName}.", "Go to floor {floorName}."} , "ChangeFloors");
    model->Intents.push_back({"{action} the door."}, "OpenCloseDoor");
    
    model->Entities.push_back({ "floorName" , Intent::EntityType::List, Intent::EntityMatchMode::Strict, {"one","two", "lobby", "ground floor"} });

    std::vector<std::shared_ptr<LanguageUnderstandingModel>> collecton;
    
    collection.push_back(model);
    intentRecognizer->ApplyLanguageModels(collection);

    std::cout << "Say something ..." << std::endl;

    auto result = intentRecognizer->RecognizeOnceAsync().get();

    switch (result->Reason)
    {
    case ResultReason::RecognizedSpeech:
        std::cout << "RECOGNIZED: Text = " << result->Text.c_str() << std::endl;
        std::cout << "NO INTENT RECOGNIZED!" << std::endl;
        break;
    case ResultReason::RecognizedIntent:
        std::cout << "RECOGNIZED: Text = " << result->Text.c_str() << std::endl;
        std::cout << "  Intent Id = " << result->IntentId.c_str() << std::endl;
        auto entities = result->GetEntities();
        if (entities.find("floorName") != entities.end())
        {
            std::cout << "  Floor name: = " << entities["floorName"].c_str() << std::endl;
        }

        if (entities.find("action") != entities.end())
        {
            std::cout << "  Action: = " << entities["action"].c_str() << std::endl;
        }

        break;
    case ResultReason::NoMatch:
    {
        auto noMatch = NoMatchDetails::FromResult(result);
        switch (noMatch->Reason)
        {
        case NoMatchReason::NotRecognized:
            std::cout << "NOMATCH: Speech was detected, but not recognized." << std::endl;
            break;
        case NoMatchReason::InitialSilenceTimeout:
            std::cout << "NOMATCH: The start of the audio stream contains only silence, and the service timed out waiting for speech." << std::endl;
            break;
        case NoMatchReason::InitialBabbleTimeout:
            std::cout << "NOMATCH: The start of the audio stream contains only noise, and the service timed out waiting for speech." << std::endl;
            break;
        case NoMatchReason::KeywordNotRecognized:
            std::cout << "NOMATCH: Keyword not recognized." << std::endl;
            break;
        }
        break;
    }
    case ResultReason::Canceled:
    {
        auto cancellation = CancellationDetails::FromResult(result);

        if (!cancellation->ErrorDetails.empty())
        {
            std::cout << "CANCELED: ErrorDetails=" << cancellation->ErrorDetails.c_str() << std::endl;
            std::cout << "CANCELED: Did you update the subscription info?" << std::endl;
        }
    }
    default:
        break;
    }
}
```
## Build and run your app

Now you're ready to build your app and test our speech recognition using the Speech service.

1. **Compile the code** - From the menu bar of Visual Studio, choose **Build** > **Build Solution**.
2. **Start your app** - From the menu bar, choose **Debug** > **Start Debugging** or press <kbd>F5</kbd>.
3. **Start recognition** - It will prompt you to say something. The default language is English. Your speech is sent to the Speech service, transcribed as text, and rendered in the console.

For example if you say "Take me to floor 2", this should be the output:

```
Say something ...
RECOGNIZED: Text = Take me to floor 2.
  Intent Id = ChangeFloors
  Floor name: = 2
```

Another example if you say "Take me to floor 7", this should be the output:

```
Say something ...
RECOGNIZED: Text = Take me to floor 7.
NO INTENT RECOGNIZED!
```

The Intent ID is empty because 7 was not in our list.
