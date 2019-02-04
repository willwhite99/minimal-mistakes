---
title: How to make custom UPROPERTY specifiers
date: 2018-01-16 03:33:00 Z
categories:
- ue4
tags:
- ue4
- uproperty
- custom
- engine
- c++
---

### Intro
Firstly this is an engine modification, if you are not comfortable doing engine modifications then this post might not be for you. If you are interested but don't know how <a href="https://docs.unrealengine.com/latest/INT/GettingStarted/DownloadingUnrealEngine/" target="_blank">look here</a> (and make sure to get access to the <a href="https://www.unrealengine.com/en-US/ue4-on-github" target="_blank">source</a>).

Have you ever wondered how to make your own UPROPERTY specifiers? I did recently when I came across a problem in my project, I wanted a class derived in blueprint to have properties that I could edit in editor, but I didn't want all instances to hold copies of this data. As my project holds hundreds of instances at any time, it is a huge waste of memory. But I needed access to these defaults (I did not need them to change ever), I first tried the Transient property but this didn't work and my instances were still copying over data, I then tried with the Instanced property which worked! However this only works if the property is a UObject (which is required by the Instanced property), which not all of my properties were.

{% highlight cpp %}
UPROPERTY(EditDefaultsOnly, Category="Test")
TArray<FMyStruct> SomeInformation;
{% endhighlight %}

I could just make all my instances delete the copied data when they are created, however I foresaw that I might be doing this a few times throughout my project. So I decided to implement my own UPROPERTY specifier that would store the defaults in the CDO but not any instances that were spawned from it.

### Implementation
Firstly the UnrealHeaderTool needs to be tweaked to include our new UPROPERTY, all of it's code is located in Source/Programs/UnrealHeaderTool/ and we will first need to change Private/Specifiers/VariableSpecifiers.def
This contains all of the UPROPERTY specifiers, you might find one that you haven't seen before (as I did)! For my UPROPERTY I called my specifier CDOOnly so I added a line like all the others, in alphabetical order:
```
VARIABLE_SPECIFIER(CDOOnly)
```
Next we need to edit the Engine side a little to hold our new Property specifier, under Source/Runtime/CoreUObject/Public/UObject/ObjectMacros.h

At line 364 (4.17) begins the list of CPFs aka the property specifiers. Right at the bottom after CPF_SkipSerialization we will add our own. It's important to point out that these specifiers are packed into a 64 bit mask, currently epic has used 56 bits of this mask, leaving 8 bits free, so don't go too crazy adding specifiers.

So add our new CPF specifier at the end like so:
{% highlight cpp %}
#define CPF_CDOOnly DECLARE_UINT64(0x0100000000000000) // Property will only export to the CDO object, instances will not be copied
{% endhighlight %}

Back to the UnrealHeaderTool, now we need to edit the Private/HeaderParserTool.cpp file, this is where your headers get parsed, the thing that check that you haven't forgotten your .generated.h files or that you have used the right UCLASS or USTRUCT specifiers, it all goes here. We want to go the FHeaderParser::GetVarType function under the huge switch statement, we want to put our new property specifier at the bottom (or top if you want), or just search for case EVariableSpecifier::SkipSerialization: and put the following below it:
{% highlight cpp %}
case EVariableSpecifier::CDOOnly:
{
	Flags |= CPF_CDOOnly;
}
break;
{% endhighlight %}

That's it, you now have made a custom UPROPERTY specifier, that does nothing. From here, you could check that the specifier is being used properly such as if you want the specifier only applied to uobjects, or you could move onto the engine side implementation.

From here it really depends what you want your UPROPERTY specifier to do, I will continue how I added my implementation but this is just for my implementation.

### CDOs and how they work
When the engine or game starts, an instance of every UCLASS has a default object created with the defaults you specify in the class' constructor. This happens for derived blueprints as well. The CDO loads the class' serialized data if there is any, this includes the properties that we have for that class. This is so that any spawned instances of this class just copy the CDOs data over instead of reloading all of the default data, which saves time.

So the way I wanted my specifier to work is that I need to prevent the duplication of the CDO properties, this happens in 2 places, ObjectGlobals.cpp and BlueprintGeneratedClass.cpp.

In ObjectGlobals.cpp under FObjectInitializer::InitProperties we want to change a single line from line 3026 (4.17)
{% highlight cpp %}
for (UProperty* P = bCanUsePostConstructLink ? Class->PostConstructLink : Class->PropertyLink; P; P = bCanUsePostConstructLink ? P->PostConstructLinkNext : P->PropertyLinkNext)
{
	if (bNeedInitialize)
	{		
		bNeedInitialize = InitNonNativeProperty(P, Obj);
	}

	bool IsTransient = P->HasAnyPropertyFlags(CPF_Transient | CPF_DuplicateTransient | CPF_NonPIEDuplicateTransient);
--->	if (!IsTransient || !P->ContainsInstancedObjectProperty() && !P->HasAnyPropertyFlags(CPF_CDOOnly))
	{
		if (bCopyTransientsFromClassDefaults && IsTransient)
		{
			// This is a duplicate. The value for all transient or non-duplicatable properties should be copied
			// from the source class's defaults.
			P->CopyCompleteValue_InContainer(Obj, ClassDefaults);
		}
		else if (P->IsInContainer(DefaultsClass))
		{
			P->CopyCompleteValue_InContainer(Obj, DefaultData);
		}
	}
}

// This step is only necessary if we're not iterating the full property chain.
if (bCanUsePostConstructLink)
{
	// Initialize remaining property values from defaults using an explicit custom post-construction property list returned by the class object.
	Class->InitPropertiesFromCustomList((uint8*)Obj, (uint8*)DefaultData);
}
{% endhighlight %}

This is actually only for when the game starts, when the CDO is not fully initialized so the engine does a manual copy of each property. Usually this will only occur on the class' subobjects such as the components. Otherwise the properties get skipped and are copied over in the InitPropertiesFromCustomList method, we can prevent our property being copied in the code below.

The code in InitPropertiesFromCustomList simply iterates over custom properties defined or editted in blueprints. We can stop our property from entering this 'list' in BlueprintGeneratedClass.cpp in the UBlueprintGeneratedClass::BuildCustomPropertyListForPostConstruction at line 376 we want to make the following change:
{% highlight cpp %} 
for (UProperty* Property = InStruct->PropertyLink; Property; Property = Property->PropertyLinkNext)
	{
		const bool bIsConfigProperty = Property->HasAnyPropertyFlags(CPF_Config) && !(OwnerClass && OwnerClass->HasAnyClassFlags(CLASS_PerObjectConfig));
		const bool bIsTransientProperty = Property->HasAnyPropertyFlags(CPF_Transient | CPF_DuplicateTransient | CPF_NonPIEDuplicateTransient);

		// Skip config properties as they're already in the PostConstructLink chain. Also skip transient properties if they contain a reference to an instanced subobjects (as those should not be initialized from defaults).
--->		if (!bIsConfigProperty && (!bIsTransientProperty || !Property->ContainsInstancedObjectProperty()) && !Property->HasAnyPropertyFlags(CPF_CDOOnly))
		{
			for (int32 Idx = 0; Idx < Property->ArrayDim; Idx++)
			{
{% endhighlight %}

### Conclusion
Following this we can add CDOOnly to our UPROPERTYs and at runtime only the CDO will hold any data for that property.
{% highlight cpp %}
UPROPERTY(EditDefaultsOnly, CDOOnly, Category="Test")
TArray<FMyStruct> SomeInformation;
{% endhighlight %}

Now if we need to access this property we can get the information from the CDO like so (should point out, **never edit these values**):
{% highlight cpp %}
const TArray<FMyStruct>& SomeInformationRef = GetClass()->GetDefaultObject<UMyObject>()->SomeInformation;
{% endhighlight %}