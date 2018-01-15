---
layout: post
title: Designing Player World Interaction in Unreal Engine 4
image: https://cdn2.unrealengine.com/Unreal+Engine%2FUE-Logo-988x988-1dee3bc7f6714edf3c21ee71826ebab54ae02077.png
---

[Unreal Engine 4](https://www.unrealengine.com) is an awesome game engine and the Editor is just as good. There are a lot of built in tools for a game (especially shooters) and some excellent tutorials out there for it. So, here is one more. Today the topic to discuss is different methods to program player world interaction in Unreal Engine 4 in C++. While the context is specific to UE4, it can also easily translate to any game with a similar architecture.

![Unreal Engien 4 Logo](https://cdn2.unrealengine.com/Unreal+Engine%2FUE-Logo-988x988-1dee3bc7f6714edf3c21ee71826ebab54ae02077.png "UE4 Logo")

# Interaction via Overlaps

By and far, the most common tutorials for player-world interaction is to use [Trigger Volumes or Trigger Actors](https://docs.unrealengine.com/latest/INT/Engine/Actors/Triggers/). This makes sense, it is a decoupled way to set up interaction and leverages most of the work using classes already provided by the engine. Here is a simple example where the overlap code is used to interact with the player:

## Header
```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "InteractiveActor.generated.h"

UCLASS()
class GAME_API InteractiveActor : public AActor
{
	GENERATED_BODY()

public:
	// Sets default values for this actor's properties
	InteractiveActor();

    virtual void BeginPlay() override;

protected:
	UFUNCTION()
	virtual void OnInteractionTriggerBeginOverlap(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

	UFUNCTION()
	virtual void OnInteractionTriggerEndOverlap(UPrimitiveComponent* OverlappedComp, class AActor* OtherActor, class UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);

    UFUNCTION()
    virtual void OnPlayerInputActionReceived();

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = Interaction)
	class UBoxComponent* InteractionTrigger;
}
```

This is a small header file for a simple base Actor class that can handle overlap events and a single input action. From here, one can start building up the various entities within a game that will respond to player input. For this to work, the player pawn or character will have to overlap with the `InteractionTrigger` component. This will then put the `InteractiveActor` into the input stack for that *specific player*. The player will then trigger the input action (via a keyboard key press for example), and then the code in `OnPlayerInputActionReceived` will execute. Here is a layout of the executing code.

## Source
```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#include "InteractiveActor.h"
#include "Components/BoxComponent.h"

// Sets default values
AInteractiveActor::AInteractiveActor()
{
	PrimaryActorTick.bCanEverTick = true;

	RootComponent = CreateDefaultSubobject<USceneComponent>(TEXT("Root"));
	RootComponent->SetMobility(EComponentMobility::Static);

	InteractionTrigger = CreateDefaultSubobject<UBoxComponent>(TEXT("Interaction Trigger"));
	InteractionTrigger->InitBoxExtent(FVector(128, 128, 128));
	InteractionTrigger->SetMobility(EComponentMobility::Static);
	InteractionTrigger->OnComponentBeginOverlap.AddUniqueDynamic(this, &ABTPEquipment::OnInteractionProxyBeginOverlap);
	InteractionTrigger->OnComponentEndOverlap.AddUniqueDynamic(this, &ABTPEquipment::OnInteractionProxyEndOverlap);

	InteractionTrigger->SetupAttachment(RootComponent);
}

void AInteractiveActor::BeginPlay()
{
    if(InputComponent == nullptr)
    {
        InputComponent = ConstructObject<UInputComponent>(UInputComponent::StaticClass(), this, "Input Component");
        InputComponent->bBlockInput = bBlockInput;
    }

    InputComponent->BindAction("Interact", EInputEvent::IE_Pressed, this, &AInteractiveActor::OnPlayerInputActionReceived);
}

void AInteractiveActor::OnPlayerInputActionReceived()
{
    //this is where logic for the actor when it receives input will be execute. You could add something as simple as a log message to test it out.
}

void AInteractiveActor::OnInteractionProxyBeginOverlap(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
	if (OtherActor)
	{
		AController* Controller = OtherActor->GetController();
        if(Controller)
        {
            APlayerController* PC = Cast<APlayerController>(Controller);
            if(PC)
            {
                EnableInput(PC);
            }
        }
	}
}

void AInteractiveActor::OnInteractionProxyEndOverlap(UPrimitiveComponent* OverlappedComp, class AActor* OtherActor, class UPrimitiveComponent* OtherComp, int32 OtherBodyIndex)
{
	if (OtherActor)
	{
		AController* Controller = OtherActor->GetController();
        if(Controller)
        {
            APlayerController* PC = Cast<APlayerController>(Controller);
            if(PC)
            {
                DisableInput(PC);
            }
        }
	}
}
```

## Pros and Cons

The positives of the collision volume approach is the ease at which the code is implemented and the strong decoupling from the rest of the game logic. The negatives to this approach is that interaction becomes broad when considering the game space as well as the introduction to a new interactive volume for each interactive within the scene.

# Interaction via Raytrace

Another popular method is to use the look at viewpoint of the player to [ray trace](https://en.wikipedia.org/wiki/Ray_tracing_(graphics)) for any interactive world items for the player to interact with. This method usually relies on inheritance for handling player interaction within the interactive object class. This method eliminates the need for another collision volume for item usage and allows for more precise interaction targeting.

## Source

### AInteractiveActor.h

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "InteractiveActor.generated.h"

UCLASS()
class GAME_API AInteractiveActor : public AActor
{
	GENERATED_BODY()

public:
    virtual OnReceiveInteraction(class APlayerController* PC);
}
```

### AMyPlayerController.h

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerController.h"
#include "AMyPlayerController.generated.h"

UCLASS()
class GAME_API AMyPlayerController : public APlayerController
{
	GENERATED_BODY()

    AMyPlayerController();

public:
    virtual void SetupInputComponent() override;

    float MaxRayTraceDistance;

private:
    AInteractiveActor* GetInteractiveByCast();

    void OnCastInput();
}
```

These header files define the functions minimally needed to setup raycast interaction. Also note that there are two files here as two classes would need modification to support input. This is more work that the first method shown that uses trigger volumes. However, all input binding is now constrained to the single `ACharacter` class or - if you designed it differently - the `APlayerController` class. Here, the latter was used.

The logic flow is straight forward. A player can point the center of the screen towards an object (Ideally a HUD crosshair aids in the coordination) and press the desired input button bound to `Interact`. From here, the function `OnCastInput()` is executed. It will invoke `GetInteractiveByCast()` returning either the first camera ray cast collision or `nullptr` if there are no collisions. Finally, the `AInteractiveActor::OnReceiveInteraction(APlayerController*)` function is invoked. That final function is where inherited classes will implement interaction specific code.

The simple execution of the code is as follows in the class definitions.

### AInteractiveActor.cpp

```cpp
void AInteractiveActor::OnReceiveInteraction(APlayerController* PC)
{
    //nothing in the base class (unless there is logic ALL interactive actors will execute, such as cosmetics (i.e. sounds, particle effects, etc.))
}
```

### AMyPlayerController.cpp

```cpp
AMyPlayerController::AMyPlayerController()
{
    MaxRayTraceDistance = 1000.0f;
}

AMyPlayerController::SetupInputComponent()
{
    Super::SetupInputComponent();
    InputComponent->BindAction("Interact", EInputEvent::IE_Pressed, this, &AInteractiveActor::OnCastInput);
}

void AMyPlayerController::OnCastInput()
{
    AInteractiveActor* Interactive = GetInteractiveByCast();
    if(Interactive != nullptr)
    {
        Interactive->OnReceiveInteraction(this);
    }
    else
    {
        return;
    }
}

AInteractiveActor* AMyPlayerController::GetInteractiveByCast()
{
    FVector CameraLocation;
	FRotator CameraRotation;

	GetPlayerViewPoint(CameraLocation, CameraRotation);
	FVector TraceEnd = CameraLocation + (CameraRotation.Vector() * MaxRayTraceDistance);

	FCollisionQueryParams TraceParams(TEXT("RayTrace"), true, GetPawn());
	TraceParams.bTraceAsyncScene = true;

	FHitResult Hit(ForceInit);
	GetWorld()->LineTraceSingleByChannel(Hit, CameraLocation, TraceEnd, ECC_Visibility, TraceParams);

    AActor* HitActor = Hit.GetActor();
    if(HitActor != nullptr)
    {
        return Cast<AInteractiveActor>(HitActor);
    }
	else
    {
        return nullptr;
    }
}
```

## Pros and Cons

One pro for this method is the control of input stays in the player controller and implementation of input actions is still owned by the `Actor` that receives the input. Some cons are that the interaction can be fired as many times as a player clicks and does not repeatedly detect interactive state without a refactor using a `Tick` function override.

# Conclusion

There are many methods to player-world interaction within a game world. In regards to creating `Actors` within Unreal Engine 4 that allow for player interaction, two of these potential methods are collision volume overlaps and ray tracing from the player controller. There are several other methods discussed out there that could also be used. Hopefully, the two implementations presented help you decide on how to go about player-world interaction within your game. Cheers!