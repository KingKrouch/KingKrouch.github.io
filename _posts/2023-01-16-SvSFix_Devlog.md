---  
layout: post  
title: Improving Neptunia Sisters Vs. Sisters with SvSFix  
tags: [Articles, Releases]  
excerpt: "Technical and PC port related improvements to Neptunia Sisters Vs. Sisters."  
feature-img: "https://image.api.playstation.com/vulcan/ap/rnd/202208/1800/vrR9YAmG32iUwoVeQZViCWDQ.jpg"  
thumbnail: "https://image.api.playstation.com/vulcan/ap/rnd/202208/1800/vrR9YAmG32iUwoVeQZViCWDQ.jpg"  
image: "https://image.api.playstation.com/vulcan/ap/rnd/202208/1800/vrR9YAmG32iUwoVeQZViCWDQ.jpg"
---  
***Table of Contents (So you can skip around and avoid the fluff if you absolutely cannot stand someone that leaves no stone unturned in explanations):***
* [Introduction](#introduction)
* [IL2CPP Shenanigans (Why it's bad, and how I managed to work around this)](#il2cpp_info)
* [Port Report (Since somebody has to do it)](#port_report)
* [How I fixed some stuttering problems (And why FixedUpdate is a looming threat with Unity games)](#fixed_update)
* [Changes made with this mod and Links to Download and Source Code](#changes_made)



# Introduction {#introduction}

**This marks the first time that a Compile Heart developed game is using the Unity engine. This is also seemingly also the first PC port handled in-house by CH themselves** *(Beforehand, it wasn't unusual to see PC ports outsourced to PreApp Partners or Ghostlight, with mostly mixed/negative results)*.
<br>**Later on, I will detail some areas that are being handled fine, while also detailing areas that need improvements made. But for some semantics on my thoughts on Unity modding for those curious:**

**Unity's modding support** *(Even accounting for games which were never designed with it from the start)* **is light-years ahead of Unreal Engine or a majority of licensed engines**.
<br>Speaking from some experience with Unreal Engine games *(A majority of them share extremely similar and preventable issues, to where I've boycotted making mods/fixes for those games)*, and even the Orochi-based games that CH has developed in the past *(Where I usually had to wade through x86 ASM code or manipulate D3D11 draw calls manually to make changes to things)*, it's actually surprisingly quick to get up and running with modifications using [BepInEx](https://github.com/BepInEx/BepInEx), a modding framework for Unity games.
It's generally easier to extend engine functionality without needing massive engine source code overhauls _(due to how modular the engine is)_, which makes fixing potential oversights and problems way easier, assuming the game wasn't packaged with IL2CPP. <br>**For example, I think supporting SteamInput, custom shading models _(like flexible toon shading that also takes shadows into consideration)_, and other systems is far easier on Unity compared to Unreal Engine _(although I do admit that Unity's UI framework needs work)_. Good luck adding any of those to Unreal without a low-level engine developer on-board or without some sort of limitation that isn't good enough. You can do both of those things without tearing apart the engine in Unity. This is what I think is completely disregarded in terms of the whole Unity vs Unreal discourse.**

If you're wondering why EnigmaFix *(my mod improving several IdeaFactory PC ports sharing the same game engine, including but not limited to the Death end re;Quest games)* is taking long to finish, that's because of the slow process that it takes to figure everything out just with x86 Assembly code without any sort of .PDB debug symbols *(it is also a pet project of mine for learning game modding, which unfortunately suffered from scope-creep)*.
**I figured that releasing a fix for a more recent game beforehand would generally be better for getting feedback addressed for future releases, and for generating a discussion.**

**However, as I will divulge soon, this release suffers from some issues that have been plaguing a ton of Unity releases as of late** *(Whether that's due to poor documentation or stubborn design decisions)*, and I figured I'd share my findings (as I've seen plenty of other games share similar issues).
*So to address the massive elephant in the room (before we get to the port report)*:

# IL2CPP Shenanigans _(Why it's bad, and how I managed to work around this)_ {#il2cpp_info}

**As a PSA: this game release uses Unity's IL2CPP packaging system, which makes fixing or improving things on the end-user side incredibly difficult, which honestly I have been noticing a pattern lately in regards to half-baked releases on PC that use Unity.**
~~There's a lot of crap that people claim online that I feel like I have to unpack.~~

### This may be somewhat opinionated _(and very blunt)_, but... **Boiling things down to three specific reasons why shipping a single-player game with IL2CPP is generally a bad idea on PC**:

1. **It makes it incredibly difficult to get BepInEx *(and other modding frameworks like MelonLoader)* working properly** above certain engine versions.
Speaking from experience, many months later, and with some discussion with BIE maintainers, BepInEx 6 builds *(and plugins like UnityExplorer)* still have problems functioning with the game, even after changes have been made to their DLL unhollower system *(Which essentially creates proxy assemblies for IL2CPP functions)*.
**Some IL2CPP games even require custom builds of UnityExplorer _(as it's a tool that hasn't been maintained in a bit)_ to properly investigate in-game objects, classes, and properties.**
<br><br>2. **All functions are compiled to x86 Assembly code, which essentially obfuscates how the game functions, and makes debugging and fixing problems on the end-user side difficult** without a proper development build and/or project files.
**There is already enough of a problem with games on PC shipping without quality control and with easily preventable oversights and problems** *(Whether that's due to time or budgetary constraints is up for interpretation, but I'm personally tired of gambling my money on new game releases that never get even basic issues patched or release in a barely tolerable state)*, and **unlike something like SteamStub** _(Which can be easily removed),_ **IL2CPP cannot easily be worked around.**
<br><br>3. **For a single-player game without any form of anti-tamper (like Denuvo), shipping with Mono runtimes is a non-issue.**
**There's virtually next to nothing that needs to be protected to an insane degree** *(e.g: server or security related functionality in multiplayer games)*. Most of the justification that I see online in regards to this is unfounded delusions from amateur developers on online forums _(possibly with undiagnosed schizophrenia)_ that anyone _(Could be CIA fellas that glow in the dark)_ is gonna bother ripping content from your game. Considering there's entire online forums that will find a way to extract game content anyways _(Either for modding or datamining, hello Xentax/Xenhax and Undertow forums!)_, and asset-flipping content from existing games isn't really a thing, it's safe to say this is completely unfounded.
**Security through obfuscation doesn't work**, and according to the logic of people defending the lack of fighting game beta tests on PC _(as an example of what I mean by this)_, Switch games shouldn't have beta tests either because someone can dump the ROM onto a PC and datamine it.
<br><br>**Performance from my testing is nearly identical between an IL2CPP and Mono build on Windows**, so I personally don't buy the excuse of *"performance"* at least on PC, especially with how difficult it is to get anything but crackpot advice and complacency in online game development communities *(Why ever bother asking how to make certain systems work better? Rhetorical Question.)*
**The only justification that you can really make for shipping a single-player game with IL2CPP is with multi-platform releases, and even in that scenario, it's not uncommon to see multi-platform games ship with Mono on PC instead** *(the Overcooked games and Super Neptunia RPG come to mind immediately).*

***However***, due to a blessing in disguise that happened during the beta testing phase for the PC version, the `_BackUpThisFolder_ButDontShipItWithYourGame` folder was shipped alongside the game content on Steam for three or four patches before being removed. **This folder contains very useful debugging information and Mono binaries before they're compiled from IL to native binaries**, and as it turns out, [some games on Steam have included this in their release repositories accidentally](https://twitter.com/thexpaw/status/1393222955698728963).
This was luckily near the end of beta testing, and not much was changed between the final build _(outside of a smaller sized load/save selection screen, which is simply an asset modification)_ and the final accidentally shipped Mono runtimes _(Being DLLs that can easily be viewed with an IL decompiler like [dnSpy](https://github.com/dnSpy/dnSpy))_.
***I am feeling uneasy about potential changes to the game code in official updates that may break my patch, especially with DLC already being confirmed. If things break, I'm sorry, you will want to have a backup of the game handy in case an official update breaks something.***

**I also am feeling uneasy about bundling the needed runtimes with the mod for several reasons _(Including but not limited to NDA related stuff)_, but it seemed like a necessary evil considering the alternative _(No mod, and issues that probably won't get addressed in a patch, judging by previous releases)_. Those are not covered under the same software license for the project for very obvious reasons.**
***But if I looked at things through a pragmatic standpoint, I'd be willing to take down the mod, mono runtimes, and source code if I was given an DMCA takedown, as harm is not my personal intent.***

**After some work compiling a barebones Unity project with both Mono and IL2CPP builds, and comparing the files, I managed to figure out how to get an IL2CPP game to load Mono runtimes instead. Unlike IL2CPP, BepInEx and UnityExplorer actually works properly this time!**
<br>_(This primarily comes down to a UnityPlayer.dll file that needs to be changed, and some Unity related DLLs in the Managed folder being exchanged for fresh ones, for those wondering on what to do given a `_BackUpThisFolder_ButDontShipItWithYourGame` folder)_

# Port Report (Since somebody has to do it) {#port_report}

So to sum up this release, **it's a step in the right direction, but there are some growing pains with using a different game engine and doing in-house ports that need to be addressed.**

Since IdeaFactory seemingly has been taking feedback more seriously *(Alongside NISA and PH3Games, good job fellas!)*, I figured that I would be a bit specific on where improvements need to be made *(Even if progress has been being made slowly, and members of the localization staff during testing have been transparent about being open for feedback).

While I am certain a majority of these issues won't be addressed in official updates _(as beta testing is done really close to the end of development, I've already reported a few of these issues, and IdeaFactory doesn't particularly have a good track record with fixing problems in their releases post-launch)_, it's a useful thing to document for future reference and future games.* _**As a wise man once said:**_

<figure>
  <img src="/assets/img/posts/svs/wise_cracker.png" alt="Words of Wisdom"/>
  <figcaption>(This perfectly encapsulates my point about how feedback that isn't specific enough can absolutely result in nothing properly being addressed. For example, 3440x1440 being added to a hardcoded list doesn't really fix the problem with how options are handled in a lot of mediocre ports.)</figcaption>
</figure>

**To start:**

👍 **The game _(mostly)_ properly supports arbitrary framerates with the only technical cap being VSync/Vertical-Sync and some FixedUpdate shenanigans _(Which I'm doing to talk about soon)_.
This is great from the standpoint that nothing is being hardcoded, great in terms of being a step up from previous games _(Where a lot of stuff was bound to a fixed tickrate and required assembly opcode patches to slow down/speed up based on the current framerate)_ and that you won't get uneven/inconsistent framepacing _(Without FreeSync/GSync)_ with arbitrary refresh rates that don't line up with 60Hz.
<br><br>While this leaves a lot to be desired _(for those that would rather have a framelimiter for power consumption reasons)_, I think a framelimiter that actually supports arbitrary refresh rates properly by simply having it operate on divisible factors of the current display refresh rate (rather than using a limiter that operates on a hardcoded 30/60/120/144 integer value) would do wonders.**

If you are wondering why there's a discussion around console games locked at 30FPS "feeling/looking better" than PC games at the same target, it's because those games are often using the console's VBlank/VSync functions *(at half refresh rate)* rather than opting to manually implement a limiter _(like with a lot of ports on PC, or in the case of Bloodborne on PS4, which often occurs with mixed/bad results)_. Do note that a framerate cap usually has a lower input latency compared to VSync _(In case anyone asks "Why not just do something similar on PC")_, but often has issues with proper framepacing. *40/45FPS framerate caps (before VRR was introduced) were not viable on consoles, due to rapid uneven fluctuations of images being presented between 33 and 16 miliseconds, why would that be any different with a 75Hz or 144Hz panel (without VRR) on PC? The Steam Deck had to somewhat work around this by offering per-game refresh rate overrides (Which is something that isn't handled properly on Windows, especially without Fullscreen Exclusive modes)*

I've also seen my fair share of framelimiters on PC that have rounding issues, desyncing, or stutter, and hardcoded 30/60/90/120/144 options or an arbitrarily decided limit aren't exactly ideal for many reasons _(and can actually introduce more problems than they attempt to solve)_, so I'm opting to take this in my own hands by taking a similar approach to how Crash N'Sane Trilogy handles the problem _(But without using VSync)_.

***NOTE:*** ***If you are having problems with the custom framelimiter, you can actually disable it (change the frame interval setting in SvSFix.cfg to "0", and opt to use Special K or RTSS for your needs instead.***

*On a slightly related tangent, if you want my opinion on what release of the game to buy, the unlocked framerate on PC is really transformative in battles (Which plays similarly to action RPGs like the Tales series), and it's a bit of a shame that the PS4 and even PS5 releases are locked at 30FPS, when it's entirely possible to get double or triple that on PS5-equivalent PC hardware. This is probably the release of the game that I'm going to recommend as a result.*

👎 **The game is using FixedUpdate incorrectly _(and without any sort of interpolation)_ for character movement _(Which does end up affecting camera movement)_, which introduces unavoidable stuttering that a typical internal or external 60FPS framerate limit can't solve a majority of times.** *This issue is not prevalent in the PS4/PS5 release, due to those being locked at 30FPS.* Raising the FixedUpdate rate in the UnityEngine.Time class from Unity's default of 50Hz *(0.02)* to your desired framerate *(I.E: 1 ÷ 144)* is a simple workaround, but can sometimes introduce problems elsewhere in the game code that relies on deterministic timings or a fixed tickrate *(Samurai Maiden is an example of a recent release where physics handling on jumping for both player and AI enemies are being done at a fixed 60Hz rate, which makes a proper fix for the camera stuttering above 60FPS next to impossible without access to the actual game code)*. **Hence why I had to dig into the game's code and find a proper workaround for this issue, more details on this later.**

👍 **Nothing major resolution-wise is being hardcoded, aside from some UI elements, and inappropriate usages of physical cameras during cutscenes.**

**You can freely resize or adjust the game window** *(and ignore the in-game hardcoded resolution options)*, **and still have 3D rendering and post-processing render correctly** *(Which some Unity games from Atlus such as SMT III: Nocturne HD or Soul Hackers 2 did incorrectly by using hardcoded render targets attached to UI elements rather than the engine's built-in functionality).*

👍 **Due to Unity's NewInput system, there is native PlayStation and Switch controller support** *(Albeit with Xbox's Back button being bound to the PS Share button rather than the trackpad)* **outside of the game only officially supporting Xbox button prompts.**

👎 **The game has a hardcoded set of 12 resolution options** _(and they're seemingly using enumerators and storing the resolution in an integer index in the system save file)._ **There's limited scalability/graphics options** _(Something which would come in handy for low-end/entry-level systems or the Steam Deck)_. <br><br>**Resolutions and graphics options should really be handled through config files or even just Unity's default of the registry instead of save files that easily synchronize over to other hardware _(Something which is annoying when picking up and playing on the Steam Deck and then going back to my main systems)_.** _I am working on figuring out a solution for the resolution issue (And I even have a proof of concept list class of sorts made for this purpose, if anyone wants to look at how I plan on approaching the problem)._

👍 **There are action based rebinding for controllers _(A first for a Compile Heart game)_, and Keyboard _(Although it's sorely lacking a secondary bindings, mouse rebinding, and some actions that are seemingly hardcoded like resetting the camera rotation)_.** **There are also strange design decisions like allowing the character to switch between characters by clicking on certain UI elements, and the camera refusing to turn unless you explicitly hold down the right mouse button, this doesn't behave in a similar manner to controller input.** In previous ports handled by PreApp Partners, rebinding was handled on a button-based context rather than actions, which affected menu navigation and even resulted in weird stuff happening when you opted for a western layout when starting the game for the first time, like B (on an Xbox controller) being used for jumping while the Japanese/Switch versions used the proper button layout.

👎 **The lack of support for button prompts for anything outside of Xbox controllers. Unity's NewInput system actually supports more than just Xbox controllers out of the box without relying on SteamInput _(Which is something I can't say for Unreal Engine)_, and it's not that challenging to get SteamInput support working in a Unity game _(Although it's been a learning experience)_.** Licensing issues from native support aside, I think that they could have opted towards using SteamInput's [`GetActionOriginFromXboxOrigin`](https://partner.steamgames.com/doc/api/isteaminput#GetActionOriginFromXboxOrigin) functionality for this, as the game technically already opts PlayStation and Switch controllers into using SteamInput.

👍 **Standard gameplay uses Hor+ FOV scaling** *(so the vertical space doesn't decrease the wider the aspect ratio)*, and **while this is a good starting point** *(Better than any Unreal Engine game not being bothered to switch the poor default of Vert- for FOV scaling to Hor+, and then resulting to pillarbox the camera during testing due to the editor viewport actually zooming in)* **I think this can be improved in a way that doesn't screw 16:10/4:3 display users over.** _On that topic:_

👎 **Cutscenes are seemingly using Unity's physical cameras incorrectly**, causing cropping on anything that isn't 16:9. Gameplay cameras are using Hor+ scaling, which while is good for 16:9 or wider, isn't great for anything narrower. I have managed to fix this by using an Overscan GateFit, and in the case of gameplay with a 16:9 sensor size _(So it approximates Major Axis FOV Scaling)_. While Vert- behavior in Unreal Engine is fine for anything narrower than 16:9, it also completely screws over anyone with a resolution wider than 16:9 _(and makes it look like tunnel vision, and needing to increase the camera FOV to compensate)_, and with Hor+, this is the opposite effect. So I decided that changing this _(Which also comes in handy for when you need to letterbox or pillarbox things like cutscenes)_ and offering a balance would be a good idea. 

👎 **There is some really janky stuff going on with UI element anchoring and pivoting that sadly gets in the way of proper _(or at least effortless)_ arbitrary resolution support** _(at least out of the box)_. When attempting to use 16:10 resolutions _(as an example)_, UI elements also cut off rather than scaling correctly. There are also portions like the title screen and end credits that are using AspectRatioFitter components incorrectly _(opting to zoom in rather than fit to the center of the screen)_. <br><br>If there was a letterbox/pillarbox actor component _(For hiding stuff with 16:9 fullscreen UI elements (like backgrounds) that weren't designed for anything different)_, and if UI elements were handled using AspectRatioFitters, I'd that would make supporting 16:10 and ultrawide configurations far easier.

**PS (Some free advice if you will):** _If anyone designing UI elements for games are bothering to read this, trust me when I say that anchoring UI elements correctly alongside using AspectRatioFitters goes a long way in making sure that UI elements are presentable with different resolutions and aspect ratios. There are also some scenarios where the player would prefer the UI to be restricted to a portion of the center of the screen (I.E: 32:9 displays), which makes using an AspectRatioFitter useful for both gameplay UI and stuff that needs to stay in a certain portion.
<br><br>That's probably one of the bigger points of contention that I have with how I think things are being handled in this game outside of the FixedUpdate shenanigans (Note: I've seen plenty of other games that don't do this stuff particularly well either), as coming up with math formulas to calculate the correct anchor/pivot points for certain UI elements that aren't anchored properly is painful.
<br><br>I've seen some people that port games to PC for a living (e.g: Durante), claim that most of the work in supporting arbitrary aspect ratios comes down to making [sure UI elements display properly](https://www.reddit.com/r/Games/comments/eq6qpv/comment/feog7l8/), and that's not a wrong statement after what I've seen. However, I do think that in the case of most multiplatform engines, this can be accounted for rather easily.<br> For those using Unreal Engine (for example), you can approximate something similar using a panel inside of both a Size and ScaleBox in the UMG editor._

## ***How I fixed some stuttering problems (And why FixedUpdate is a looming threat with Unity games) :*** {#fixed_update}

Arguably the biggest issue that I managed to _(mostly)_ fix with this mod is the FixedUpdate stuttering. I figured I'd share some information regarding that, which I feel like would be a good warning sign on how blatant oversights can absolutely affect the end product to such a massive degree.

<video src="https://github.com/KingKrouch/kingkrouch.github.io/raw/master/assets/img/posts/svs/FixedUpdate.webm" controls="controls" style="max-width: 720px;">
</video>

This does not appear smooth like what the framerate overlay on Steam claims it is, and if a game is using IL2CPP, this will usually be impossible to fix unless you cap your framerate below 50FPS.

I have also seen several games using Unity _(Samurai Maiden, Super Neptunia RPG, Overcooked, Deadly Premonition 2, Monochrome Mobius, and Rune Factory 5 are examples)_ that use FixedUpdate for camera/player movement updating. This can present extremely apparent problems with stuttering especially when it's used without any sort of transform interpolation. Unlike Unreal's PSO caching problems, this is something that can be taken into account.

I've already heard several people _(and some streamers)_ mention the stuttering problems, and it's even more apparent when on a high refresh rate monitor, so I decided to investigate this.

During testing, I started by adjusting the `GlobalGameManager` file's `Time.FixedDeltaTime` property using Unity Asset Bundle Extractor (UABE). Adjusting it to something high _(like a fixed delta time of 1ms (1000FPS) or 4.16ms (240FPS))_ seemed to have worked around the problem, but at the cost of increased CPU utilization. We also don't entirely know what else is using FixedUpdate, so this method of working around the problem is not ideal. I've used this method of working around FixedUpdate problems in games _(that use IL2CPP)_ before without much problem, but there are some games, like Samurai Maiden, where it's being used to handle the velocity and handling of player/enemy jumping, and that can cause softlocking in some places. But I digress...

From my time attempting a fix, I managed to narrow down the issue, not to the camera itself, but what was controlling the player and enemy positioning. The `MapUnitCollisionRigidbodyComponent` and `MapUnitCollisionCharacterControllerComponent` scripts respectively.
I decided to temporarily disable the component to see if it was the source of the problem. Judging by the player only being able to rotate, and the camera being much smoother, it was indeed the source of the problem.

<video src="https://github.com/KingKrouch/kingkrouch.github.io/raw/master/assets/img/posts/svs/FixedUpdateToggle.webm" controls="controls" style="max-width: 720px;">
</video>

Before we get to a discussion about an appropriate fix, apparently criticizing games tying logic to the framerate is a controversial statement in some circles _(Whether that's from fanboys defending the trend of bad cashgrab PC ports, or from certain development communities that still somehow are fine with bad engine defaults (_cough cough Unreal Engine still using Vert- FOV Scaling as the default, not having working DirectInput/SteamInput support, and the people on Unreal Slackers defending every shortcoming the engine has_))_.
<br><br>I wanted to preface what I'm going to say with me saying that I'm not gonna go with the _"muh lazy developer"_ or _"throw out your good hardware and use a 1080p 60Hz monitor taken from a school, and a MadCats Xbox 360 controller from Gamestop's dumpster"_ Steam forums shtick here, because I'm not mentally five years old. I have also seen particular types of people peddle the whole _"game physics"_ talking point as a defense for games not shipping with support for arbitrary framerates without actually investigating the issue using debugging tools _(So it's a similar misinformation/speculation scapegoat for issues like what Denuvo is towards stuttering, when the games that do have a problem with it are innappropriately using anti-debugging functionality (like with Capcom's games), polling SteamAPI redundantly (Final Fantasy XV comes to mind), or have framelimiters that cause desynchronization/stutter issues that disappear as soon as I disable the limit using Cheat Engine and employ my own (Sega and Atlus games come to mind))_.

**There are some valid reasons why some would go with having game logic run on a fixed tickrate** _(For example, 30 or 60Hz)_. **Whether this is down to stuff regarding game collision, or physics related functionality, where having that stuff deterministic is ideal. That said, there are ways to where you can run a game with a fixed tickrate without tying it to rendering** _(This is something that games from Compile Heart, and for a more topical example Game Freak's games are no stranger to)_, and still having it appear smooth. **In a lot of cases with source code access, opting to use interpolation can be less work for a similar effect. I've seen my fair share of ports and source ports that use it** _(The Halo games run internally at 60Hz, Catherine Classic runs internally at 30Hz, the Ship of Harkinian Decompilation for Zelda OOT runs internally at 20Hz, all modern source ports of Doom opt to use interpolation to hide the fact that it runs internally at 35FPS)_, **and it's not a bad solution _(Although this does have a cost of 1 frame of latency)_.** **Retaining smooth framepacing on a wide range of setups is key.**

**However, from my testing, while making sure to reference a really good reading on FixedUpdate** _(I'd actually highly recommend reading [KinematicSoup's article on fixed timesteps in Unity](https://www.kinematicsoup.com/news/2016/8/9/rrypp5tkubynjwxhxjzd42s3o034o8))_, **I found myself facing rubber-banding problems with character movement, so I chose not to use this solution** _(Also due to how you have to load some of the interpolation scripts in a specific order through the project settings, which is not viable in the case of a BepInEx plugin)_.

Taking a look at some of the code in the FixedUpdate method for this function, it seems obvious on what I should try. Specifically these two lines of code:

```csharp
if (!this.collision_.UpdatePrevious(ref zero1, ref zero2, out ground_height, GameTime.FixedDeltaTime))
    return;
this.collision_.GravityGroundCapsule(ref zero2, in ground_height, GameTime.FixedDeltaTime);
```

The GameTime class basically returns an appropriate Fixed/UnFixed [Delta Time](https://en.wikipedia.org/wiki/Delta_timing) (the time between the previous and current frame) that also takes pausing into consideration. I wouldn't get too concerned about the semantics. Do note that there's plenty of gameplay logic that doesn't take the current time dilation into account (Essentially `Time.DeltaTime * Time.TimeScale`) unless alt-tabbing occurs (Where most of the internal game logic is paused, but Unity is still technically rendering), but this information shouldn't be too relevant unless you wanted to put together a turbo mode. **From what I see, a majority of the gameplay logic is using DeltaTime appropriately, and doesn't have the same issue.**

From my time messing with the game assembly files in dnSpy (an IL decompiler), I found that renaming the method from FixedUpdate to Update, and then replacing GameTime.FixedDeltaTime with GameTime.DeltaTime did the trick _(although there may still be edge case scenarios, such as when you are looking upwards while jumping forward, or when looking upwards while climbing ladders)_. But patching the game assemblies directly is not smart.

My solution for this was to create classes that derived from these two classes, and then directly patch the FixedUpdate method to not run _(using a HarmonyX Prefix patch with a boolean type that always returned false, which essentially acts like a detour that never returns to the original function)_. The problem with patching the functions directly is that you can't create new methods using HarmonyX. You also cannot access private variables. And, for those two classes in particular, you can't call an __instance of the class to replace the components directly, so I had to run a for each loop that grabbed every component in the array, and do the replace them that way. **Personally, if I had more time to approach this issue, I would have probably opted towards interpolation instead.**

And here is the results:

<video src="https://github.com/KingKrouch/kingkrouch.github.io/raw/master/assets/img/posts/svs/MovedToUpdate.webm" controls="controls" style="max-width: 720px;">
</video>

This is just one of the changes that I've made. Not counting any of the changes that I've done to UI elements, the new framerate limiter, or anything of that sorts. Maybe I'll expand upon this blogpost and add details regarding that.

## ***Changes made with this mod, alongside Links to Download and Source Code:*** {#changes_made}

***UI Modifications:***

- **Several tweaks to UI elements to make them work better with arbitrary aspect ratios, and added some math that allows for correct positioning relative to 16:9** ***(Although you can opt to center all of the UI elements to a 16:9 aspect ratio, if you are playing with 32:9 (super-ultrawide) resolutions or other aspect ratios where a center focal-point for UI is desired.***
- ***(Mostly)*** **unhardcoded the resolution used by the in-game minimap rendering** ***(By default, there is a matrix function being used for minimap rendering that is using a hardcoded 3840x2160 resolution, I adjusted this to do some math to use a 2160p/4K equivalent of the current aspect ratio, although there are still problems that need to be resolved with resolutions narrower than 16:9 causing the sides to get cut off).***
- **Introduced a custom letterbox/pillarbox UI overlay for fullscreen UI elements originally designed with 16:9 in mind** ***(So you don't see anything past those bounds that you aren't supposed to).***
- **Added Aspect Ratio Fitter elements to the Disc Development and Status menu, so they display at the proper aspect ratio when non-16:9 resolutions are being used.**
- **Fixed some fade-in/fade-out effects** ***(Such as during the load/save screen)*** **to take up the entire screen space to work around a fixed 16:9 size.**

***Graphics Modifications:***
- **Added scalability settings for Draw Distance (lodBias), Model LOD, Texture Quality (loads lower quality Mipmaps on textures the higher the value), Resolution Scaling (So you can downscale or upscale while having UI run at native resolution), MSAA, and Post-Process Anti-Aliasing (FXAA, SMAA).**

***Camera Modifications:***

- **Added some adjustments to the gameplay camera and cutscenes to use something approximated to Major Axis FOV scaling** ***(Vert- behavior below 16:9, Hor+ behavior above 16:9)*** **by using a Overscan GateFit on the camera, alongside a sensor size of 16x9. This has the benefit to where the user can opt to have pillarboxing/letterboxing during cutscenes if desired** ***(if you absolutely live by the idea of absolute artistic intent no matter how arbitrary).*** **This is similar to how Elden Ring approached to handle things** ***(Even if the game has letterboxes/pillarboxes as an overlay during gameplay, and FromSoftware could have shipped functional arbitrary aspect ratio support given some UI tweaks).***

***Input Modifications:***

- **Worked around the button prompt problem** ***(mostly),*** **by extending the in-game button prompts to take SteamInput** ***(Custom prompts for those who would rather not rely on SteamInput is coming soon)*** **based on the currently used input method into account. The former is a quick work-around for the "licensing" problem with offering non-Xbox prompts on PC, and the latter is technically feasible due to the game still having PlayStation 4 and 5 button prompts and tutorial images in the game files** ***(I just have to account for PS3 and Switch controllers).***
- **You can now pick your desired input method** ***(Keyboard/Mouse, Controller, or Automatic (the game's default functionality which switches between them based on the last pressed input))*** **in case you are using a simultaneous Controller + KB/M setup** ***(Such as the Steam Controller or Steam Deck's trackpads, or an analog keypad such as the Razer Tartarus Pro).***
- **Created a more appropriate [SteamInput layout for PlayStation controllers](steam://controllerconfig/1932160/2913919532) with the game** ***(Trackpad is used for opening the minimap,  and share is used for screenshots + instant replays just like on PS4/PS5).***

***Framerate Modifications:***

- **Implemented a framelimiter that operates on divisible factors of the display refresh rate rather than opting to use a hardcoded value, opting to use Unity's inaccurate limiter** ***(that only operates on integer values and doesn't work well with odd divisible factors (I.E: half-refresh 175Hz)).***
- **Fixed some leftover internal gameplay logic that caps the game to 60FPS if VSync is disabled for whatever reason. You can now disable VSync** ***(to reduce latency)*** **and employ a framerate cap using the new framelimiter, or you can disable both for benchmarking purposes.**
- **Fixed FixedUpdate induced stuttering with Player/Party Member movement that was causing camera stuttering, by preventing the FixedUpdate method from running, and replacing the offending component with one that has an Update method and takes Delta Time into account instead.**

***Photo Mode Modifications:***

- **Added a Steam API screenshot hook to Photo Mode's screenshot capturing system** ***(As the game saves photos into "Documents/My Games" rather than the LocalAppData folder which game saves reside in).***
  <br>**NOTE: I am still working on ironing out some kinks with how screenshots are capturing backwards and with UI elements visible.**

There are probably other things that I'm missing.

[The mod can be grabbed from here](https://github.com/KingKrouch/SvSFix) and [source code can be found here](https://github.com/KingKrouch/SvSFix).
