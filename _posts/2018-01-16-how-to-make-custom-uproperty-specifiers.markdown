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
Firstly this is an engine modification, if you are not comfortable doing engine modifications then this post might not be for you. If you are interested but don't know how <a href="https://docs.unrealengine.com/latest/INT/GettingStarted/DownloadingUnrealEngine/" target="_blank">look here</a>.

Have you ever wondered how to make your own UPROPERTY specifiers? I did recently when I came across a problem in my project, I wanted a class derived in blueprint to have properties that I could edit in editor, but I didn't want all instances to hold copies of this data. As my project holds hundreds of instances at any time, it is a huge waste of memory. But I needed access to these defaults (I did not need them to change ever), I first tried the Transient property but this didn't work and my instances were still copying over data, I then tried with the Instanced property which worked! However this only works if the property is a UObject (which is required by the Instanced property), which not all of my properties were.

So I decided to implement my own UPROPERTY specifier that would store the defaults in the CDO but not any instances that were spawned from it.

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

Back to the UnrealHeaderTool, now we need to edit the Private/HeaderParserTool.cpp file, this is where your headers get parsed, things that check that you haven't forgotten your .generated.h files or that you have used the right UCLASS or USTRUCT specifiers. It all goes here. We want to go the void FHeaderParser::GetVarType function under the huge switch statement, we want to put our new property specifier at the bottom, or just search for case EVariableSpecifier::SkipSerialization: and put the following below it:
{% highlight cpp %}
case EVariableSpecifier::CDOOnly:
{
	Flags |= CPF_CDOOnly;
}
break;
{% endhighlight %}

That's it, you now have made a custom UPROPERTY specifier, that does nothing. From here, you could check that the specifier is being used properly (further down), or you could move onto the engine side implementation.

From here it really depends what you want your UPROPERTY specifier to do, I will continue how I added my implementation but this is just for my implementation.

The way my specifier works is that I need to prevent the duplication of the CDO properties, this happens in 2 places, ObjectGlobals.cpp and BlueprintGeneratedClass.cpp.

In ObjectGlobals.cpp under FObjectInitializer::InitProperties we want to change a single line from line 3026 (4.17)
{% highlight cpp %}
for (UProperty* P = bCanUsePostConstructLink ? Class->PostConstructLink : Class->PropertyLink; P; P = bCanUsePostConstructLink ? P->PostConstructLinkNext : P->PropertyLinkNext)
{
	if (bNeedInitialize)
	{		
		bNeedInitialize = InitNonNativeProperty(P, Obj);
	}

	bool IsTransient = P->HasAnyPropertyFlags(CPF_Transient | CPF_DuplicateTransient | CPF_NonPIEDuplicateTransient);
	if (!IsTransient || !P->ContainsInstancedObjectProperty() && !P->HasAnyPropertyFlags(CPF_CDOOnly))
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
{% endhighlight %}

Lastly in BlueprintGeneratedClass.cpp in the UBlueprintGeneratedClass::BuildCustomPropertyListForPostConstruction at line 376 we want to change the line to:
{% highlight cpp %} 
if (!bIsConfigProperty && (!bIsTransientProperty || !Property->ContainsInstancedObjectProperty()) && !Property->HasAnyPropertyFlags(CPF_CDOOnly))
{% endhighlight %}

### Conclusion
Following this we can add CDOOnly to our UPROPERTYs and at runtime only the CDO will hold any data for that property.