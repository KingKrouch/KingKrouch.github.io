---  
layout: post  
title: Improving Neptunia Sisters Vs. Sisters with SvSFix  
tags: [Articles, Releases]  
excerpt: "Technical and PC port related improvements to Neptunia Sisters Vs. Sisters."  
feature-img: "https://ideafintl.com/neptunia-sisters/img/mainimg.jpg"  
thumbnail: "https://ideafintl.com/neptunia-sisters/img/mainimg.jpg"  
image: "https://ideafintl.com/neptunia-sisters/img/mainimg.jpg"
---  
***Table of Contents (So you can skip around and avoid the fluff if you absolutely cannot stand someone that leaves no stone unturned in explanations):***
* [Introduction](#introduction)
* [IL2CPP Shenanigans (Why it's bad, and how I managed to work around this)](#il2cpp_info)
* [Port Report (Since somebody has to do it)](#port_report)
* [How I fixed some stuttering problems (And why FixedUpdate is a looming threat with Unity games)](#fixed_update)
* [Changes made with this mod](#changes_made)



# Introduction {#introduction}

This marks the first time that a Compile Heart developed game is using the Unity engine. This is also seemingly also the first PC port handled in-house by CH themselves *(Beforehand, it wasn't unusual to see PC ports outsourced to PreApp Partners or Ghostlight, with mostly mixed/negative results)*. Later on, I will detail some areas that are being handled fine, while also detailing areas that need improvements made.

**Unity's modding support** *(Even accounting for games which were never designed with it from the start)* **is light-years ahead of Unreal Engine or a majority of licensed engines** *(Including PhyreEngine and Orochi/Mizuchi, both being game engines that CH have used in the past)*. Speaking from some experience with Unreal Engine games *(A majority of them share extremely similar issues, to where I've basically boycotted making mods/fixes for those games)*,  and even the Orochi-based games that CH has developed in the past *(Where I usually had to wade through x86 ASM code or manipulate D3D11 draw calls manually to make changes to things)*, it's actually surprisingly quick to get up and running with modifications using [BepInEx](https://github.com/BepInEx/BepInEx), a modding framework for Unity games. It's generally easier to extend engine functionality without needing massive engine source code overhauls (due to how modular the engine is, and it's C# nature), which makes fixing potential oversights and problems way easier, assuming the game wasn't packaged with IL2CPP.

If you are wondering why EnigmaFix *(my mod improving several IdeaFactory PC ports sharing the same game engine, including but not limited to the Death end re;Quest games)* is taking so long to finish, that's because of the slow process that it takes to figure everything out without source code *(it is also a pet project of mine for learning game modding, which ballooned in scope)*. I also figured that releasing a fix for a more recent game beforehand would generally be better for getting feedback addressed for future releases, and for generating a discussion.

**However, as I will divulge soon, this release suffers from some issues that have been plaguing a ton of Unity releases as of late** *(Whether that's due to poor documentation or stubborn design decisions)*, while not being as janky with handling certain issues as other game releases sharing the same engine *(Speaking from experience investigating some of them)*. **I will also discuss some enhancements and improvements that I've made, alongside some technicalities regarding certain features, that I feel like haven't really been specifically been discussed in certain circles properly** *(and in a way that doesn't make them seem like a colossal effort to do, and subsequently weaponized for excusing mediocrity, cough cough ultrawide support or any button prompts outside of Xbox controllers  x)*.

*So to address the massive elephant in the room (before we get to the port report)...*

# IL2CPP Shenanigans (Why it's bad, and how I managed to work around this) {#il2cpp_info}

**As a PSA: this game release uses Unity's IL2CPP packaging system, which makes fixing or improving things on the end-user side incredibly difficult, which honestly has been an increasingly more user-hostile trend that has been happening lately.**

**Boiling things down to three specific** *(something which is sorely lacking in online communities nowadays)* **reasons why** *(correct me if I say something incorrectly)*, **shipping a single-player game with IL2CPP is a bad idea on PC**:

1. **It makes it incredibly difficult to get BepInEx *(and other modding frameworks like MelonLoader)* working properly** above certain engine versions. Speaking from experience, many months later, and with some discussion with BIE maintainers, BepInEx 6 builds *(and plugins like UnityExplorer)* still have problems functioning with the game, even after changes have been made to their DLL unhollower system *(Which essentially creates proxy assemblies for IL2CPP functions)*. **Some IL2CPP games even require custom builds of UnityExplorer to properly investigate in-game objects, classes, and properties.**
2. **All functions are compiled/obfuscated to x86 Assembly code, which makes debugging and fixing problems on the end-user side difficult** without a proper development build and/or project files. **There is already enough of a problem with games on PC shipping without quality control and with easily preventable oversights and problems** *(Whether that's due to time or budgetary constraing is up for interpretation, but I'm personally tired of gambling my money on new game releases that never get patched or release in a barely tolerable state)*, and **unlike something like SteamStub** *(Which can be easily removed)*, which blocks hex editing or debuggers *(You specifically need to use the VEH debugger in Cheat Engine to prevent crashing)***, it cannot easily be worked around.**
3. For a single-player game, shipping with Mono runtimes is a non-issue. There's virtually next to nothing that needs to be protected to an insane degree *(e.g: server or security related functionality in multiplayer games)*. **Performance from my testing is nearly identical between an IL2CPP and Mono build on Windows**, so I personally don't buy the excuse of *"PeRfOrMaNcE"*, especially with how difficult it is to get anything but crackpot advice in online game development communities *(Because why ever bother asking how to make certain systems work better? Saying that sarcastically.)*. **The only justification that you can really make for shipping a single-player game with IL2CPP is with multi-platform releases, and even in that scenario, it's not uncommon to see multi-platform games ship with Mono on PC instead** *(the Overcooked games and Super Neptunia RPG come to mind immediately).*

**However**, due to a blessing in disguise that happened during the beta testing phase for the PC version, the *"_BackUpThisFolder_ButDontShipItWithYourGame"* folder was shipped alongside the game content on Steam for three or four patches before being removed. **This folder contains very useful debugging information and Mono binaries before they're compiled from IL to native binaries**, and as it turns out, [some games on Steam have included this in their release repositories accidentally](https://twitter.com/thexpaw/status/1393222955698728963). This was luckily near the end of beta testing, and not much was changed between the final build (outside of a smaller sized load/save selection screen) and the final accidentally shipped Mono runtimes (Being DLLs that can easily be viewed with an IL decompiler like [dnSpy](https://github.com/dnSpy/dnSpy)).
***However, I am feeling uneasy about potential changes to the game code in official updates that may break my patch, especially with DLC already being confirmed. If things break, I'm sorry, you will want to have a backup of the game handy in case an official update breaks something.***

**I also felt uneasy about bundling the needed runtimes with the mod for several reasons, but it seemed like a necessary evil considering the alternative (no mod, and issues that probably won't get addressed in a patch, judging by previous releases). Those are not covered under the same software license for the project for very obvious reasons.**

**After some work compiling a barebones Unity project with both Mono and IL2CPP runtimes, and comparing the files** *(Primarily comes down to a UnityPlayer.dll file that needs to be changed, some Unity related DLLs in the Managed folder being exchanged for fresh ones)*, **I managed to figure out how to get an IL2CPP game to load Mono runtimes instead. Unlike IL2CPP, BepInEx actually works properly this time!**

# Port Report (Since somebody has to do it) {#port_report}

So to sum up this release, it's a step in the right direction, but there are some growing pains with using a different game engine and doing in-house ports that need to be addressed.

Since IdeaFactory seemingly has been taking feedback more seriously *(Alongside NISA and PH3Games, good job fellas!)*, ~~which is more than I can say about a majority of Japanese publishers/developers and porting houses that release cashgrab ports in 2023 with enough corner cutting to make a certain type of [frozen PB&J sandwich](https://i5.walmartimages.com/asr/1c700394-9ee6-4915-ab8d-bb0a72a20938.402a81f87a1cb1bf72eac8d83bff86df.jpeg)~~, I figured that I would be a bit specific on where improvements need to be made *(Even if progress has been being made slowly, and members of the localization staff during testing have been transparent about being open for feedback). While I am certain a majority of these issues won't be addressed in official updates (as beta testing is done really close to the end of development, and I've already reported a few of these issues), it is a useful thing to document for future reference and future games.* _**As a wise man once said:**_

![wise_cracker.png](..%2Fassets%2Fimg%2Fposts%2Fsvs%2Fwise_cracker.png)

**To start:**

👍 **The game (mostly) properly supports arbitrary framerates with the only technical cap being VSync/Vertical-Sync. This is great from the standpoint that nothing is being hardcoded, and that you won't get uneven/inconsistent framepacing (Without FreeSync/GSync) with arbitrary refresh rates that don't line up with 60Hz. While this leaves a lot to be desired (for those that would rather have a framelimiter for power consumption reasons), I think I could offer a framelimiter that actually supports arbitrary refresh rates properly by simply having it operate on divisible factors of the current display refresh rate rather than using a limiter that operates on a hardcoded 30/60/120/144 integer value** ***(For those worrying about input lag, I am not using Unity's own VSync interval functionality to achieve this).*** If you are wondering why there's a discussion around console games locked at 30FPS "feeling/looking better" than PC games at the same target, it's because those games are often using the system's VBlank/VSync functions *(at half refresh rate)* rather than opting to manually implement a limiter *(that while has lower input latency compared to VSync has issues with proper framepacing *(cough cough Bloodborne))*. *It's worth mentioning that 40/45FPS framerate caps (before VRR was introduced) were not viable on consoles, due to rapid uneven fluctuations of images being presented between 33 and 16 miliseconds.* **Most Japanese PC ports seemingly handle framerate limiting incorrectly (resulting in rounding errors that cause the game to run a few frames less or more than intended), and it's generally bad practice when incorrectly done, which is my mindset on the situation of having VSync handle things. On the PC landscape, there's often monitors that don't evenly divide by 60Hz, which often makes hardcoded limits introduce more problems than they are supposed to fix.**

***NOTE:*** *If you are having problems with the custom framelimiter, you can actually disable it, and opt to use Special K or RTSS for your needs instead.*

*On a slightly related tangent, if you want my opinion on what release of the game to buy, the unlocked framerate on PC is really transformative in battles (Which plays similarly to action RPGs like the Tales series), and it's a bit of a shame that the PS4 and even PS5 releases are locked at 30FPS, when it's entirely possible to get double or triple that on PS5-equivalent PC hardware. This is probably the release of the game that I'm going to recommend as a result.*

👎 **The game is using FixedUpdate incorrectly (and without any sort of interpolation) for character movement (Which affects camera movement), which introduces unavoidable stuttering that a typical internal or external 60FPS framerate limit can't solve a majority of times.** *This issue is not prevalent in the PS4/PS5 release, due to those being locked at 30FPS.* Raising the FixedUpdate rate in the UnityEngine.Time class from Unity's default of 50Hz *(0.02)* to your desired framerate *(I.E: 1 ÷ 144)* is a simple workaround, but can sometimes introduce problems elsewhere in the game code that relies on deterministic timings or a fixed tickrate *(Samurai Maiden is an example of a recent release where physics handling on jumping for both player and AI enemies are being done at a fixed 60Hz rate, which makes a proper fix for the camera stuttering above 60FPS next to impossible without access to the actual game code)*. **Hence why I had to dig into the game's code and find a proper workaround for this issue, more details on this later.**

***NOTE:***  *You can choose to interpolate game logic that operates on a fixed tickrate to work around this problem, but for some technical reasons that I'll get into later, I don't think I can achieve a fix for the camera stuttering through this method.*

👍 **Nothing major resolution-wise is being hardcoded, aside from some UI elements** *(Which I will explain how I went about fixing them)*, **and inappropriate usages of physical cameras during cutscenes** that actually cuts the sides with resolutions narrower than 16:9 *(I.E: 16:10 and 4:3)*, and decreases the FOV with resolutions wider than 16:9 *(I.E: ultrawide)*.

**You can freely resize or adjust the game window** *(and ignore the in-game hardcoded resolution options)*, **and still have 3D rendering and post-processing render correctly** *(Which some Unity games from Atlus such as SMT III: Nocturne HD or Soul Hackers 2 did incorrectly by using hardcoded render targets attached to UI elements rather than the engine's built-in functionality).*

👍 **Standard gameplay uses Hor+ FOV scaling** *(so the vertical space doesn't decrease the wider the aspect ratio)*, and **while this is a good starting point** *(Better than any Unreal Engine game not being bothered to switch the poor default of Vert- for FOV scaling to Hor+, and then resulting to pillarbox during testing due to the editor viewport actually zooming in during testing)* **I think this can be improved in a way that doesn't screw 16:10/4:3 display users over.**

👍 **Due to Unity's NewInput system, there is native PlayStation and Switch controller support** *(Albeit with Xbox's Back button being bound to the PS Share button rather than the trackpad)* **outside of the game only officially supporting Xbox button prompts.**

## ***How I fixed some stuttering problems (And why FixedUpdate is a looming threat with Unity games) :*** {#fixed_update}

Arguably the biggest issue that I managed to (mostly) fix with this mod is the FixedUpdate stuttering. I figured I'd share some information regarding that, which I feel like would be a good warning sign on how oversights can absolutely affect the end product.

<video src="https://github.com/KingKrouch/kingkrouch.github.io/raw/master/assets/img/posts/svs/FixedUpdate.webm" controls="controls" style="max-width: 720px;">
</video>

This does not appear smooth like what the framerate overlay on Steam claims it is, and if a game is using IL2CPP, this will usually be impossible to fix unless you cap your framerate below 50FPS.

I have also seen several games using Unity (Samurai Maiden, Super Neptunia RPG, Overcooked, Deadly Premonition 2, Monochrome Mobius, and Rune Factory 5 are examples) that use FixedUpdate for camera/player movement updating. This can present extremely apparent problems with stuttering especially when it's used without any sort of transform interpolation. Unlike Unreal's PSO caching problems, this is something that can be taken into account.

I've already heard several people (and some streamers) mention the stuttering problems, and it's even more apparent when on a high refresh rate monitor, so I decided to investigate this.

During testing, I started by adjusting the GlobalGameManager file's Time.FixedDeltaTime property using Unity Asset Bundle Extractor (UABE). Adjusting it to something high (like a fixed delta time of 1ms (100FPS) or 4.16ms (240FPS)) seemed to have worked around the problem, but at the cost of increased CPU utilization. We also don't entirely know what else is using FixedUpdate, so this method of working around the problem is not ideal. I've used this method of working around FixedUpdate problems in games (that use IL2CPP) before without much problem, but there are some games (Samurai Maiden comes to mind), where it's being used to handle the velocity and handling of player/enemy jumping, and that can cause softlocking in some places. But I digress...

From my time attempting a fix, I managed to narrow down the issue, not to the camera itself, but what was controlling the player and enemy positioning. The `MapUnitCollisionRigidbodyComponent` and `MapUnitCollisionCharacterControllerComponent` scripts respectively.
I decided to temporarily disable the component to see if it was the source of the problem. Judging by the player only being able to rotate, and the camera being much smoother, it was indeed the source of the problem.

<video src="https://github.com/KingKrouch/kingkrouch.github.io/raw/master/assets/img/posts/svs/FixedUpdateToggle.webm" controls="controls" style="max-width: 720px;">
</video>

Before we get to a discussion about an appropriate fix, apparently criticizing games tying logic to the framerate is a controversial statement in some circles (Whether that's from fanboys defending the trend of bad cashgrab PC ports, or from certain development communities that still somehow are fine with bad engine defaults (_cough cough Unreal Engine still using Vert- FOV Scaling as the default and not having working DirectInput/SteamInput support, and the people on Unreal Slackers defending that_)).
I wanted to preface what I'm going to say with me saying that I'm not gonna go with the _"muh lazy developer"_ or _"throw out your good hardware and use a 1080p 60Hz monitor taken from a school, and a MadCats Xbox 360 controller from Gamestop's dumpster"_ Steam forums shtick here, because I'm not mentally five years old. I have also seen particular types of people peddle the whole "game physics" talking point as a defense for games not shipping with support for arbitrary framerates without actually investigating the issue using debugging tools _(So it's a similar misinformation/speculation scapegoat for issues like what Denuvo is towards stuttering, when the games that do have a problem with it are innappropriately using anti-debugging functionality (like with Capcom's games), polling SteamAPI redundantly (Final Fantasy XV comes to mind), or have framelimiters that cause desynchronization/stutter issues that disappear as soon as I disable the limit using Cheat Engine and employ my own (Sega and Atlus games come to mind))_.

There are some valid reasons why some would go with having game logic run on a fixed tickrate _(For example, 30 or 60Hz)_. Whether this is down to stuff regarding game collision, or physics related functionality, where having that stuff deterministic is ideal. That said, there are ways to where you can run a game with a fixed tickrate without tying it to rendering _(This is something that games from Compile Heart, and for a more topical example Game Freak's games are no stranger to)_, and still having it appear smooth. In a lot of cases with source code access, opting to use interpolation can be less work for a similar effect. I've seen my fair share of ports and source ports that use it _(The Halo games run internally at 60Hz, Catherine Classic runs internally at 30Hz, the Ship of Harkinian Decompilation for Zelda OOT runs internally at 20Hz, all modern source ports of Doom opt to use interpolation to hide the fact that it runs internally at 35FPS)_, and it's not a bad solution _(Although this does have a cost of 1 frame of latency)_. Retaining smooth framepacing on a wide range of setups is key.

However, from my testing, while making sure to reference a really good reading on FixedUpdate _(I'd actually highly recommend reading [KinematicSoup's article on fixed timesteps in Unity](https://www.kinematicsoup.com/news/2016/8/9/rrypp5tkubynjwxhxjzd42s3o034o8))_, I found myself facing rubber-banding problems with character movement, so I chose not to use this solution _(Also due to how you have to load some of the interpolation scripts in a specific order through the project settings, which is not viable in the case of a BepInEx plugin)_.

Taking a look at some of the code in the FixedUpdate method for this function, it seems obvious on what I should try. Specifically these two lines of code:

```csharp
if (!this.collision_.UpdatePrevious(ref zero1, ref zero2, out ground_height, GameTime.FixedDeltaTime))
    return;
this.collision_.GravityGroundCapsule(ref zero2, in ground_height, GameTime.FixedDeltaTime);
```


The GameTime class basically returns an appropriate Fixed/UnFixed Delta Time (the time between the previous and current frame) that also takes pausing into consideration. I wouldn't get too concerned about the semantics. Do note that there's plenty of gameplay logic that doesn't take the current time dilation into account (Essentially `Time.DeltaTime * Time.TimeScale`) unless alt-tabbing occurs (Where most of the internal game logic is paused, but Unity is still technically rendering), but this information shouldn't be too relevant unless you wanted to put together a turbo mode.

From my time messing with the game assembly files in dnSpy (an IL decompiler), I found that renaming the method from FixedUpdate to Update, and then replacing GameTime.FixedDeltaTime with GameTime.DeltaTime did the trick. But patching the game assemblies directly is not smart.

My solution for this was to create classes that derived from these two classes, and then directly patch the FixedUpdate method to not run (using a HarmonyX Prefix patch with a boolean type that always returned false). The problem with patching the functions directly is that you can't create new methods using HarmonyX. You also cannot access private variables. And, for those two classes in particular, you can't call an __instance of the class to replace the components directly, so I had to run a for each loop that grabbed every component in the array, and do the replace them that way.

And here is the results:

<video src="https://github.com/KingKrouch/kingkrouch.github.io/raw/master/assets/img/posts/svs/MovedToUpdate.webm" controls="controls" style="max-width: 720px;">
</video>

This is just one of the changes that I've made. Not counting any of the changes that I've done to UI elements, the new framerate limiter, or anything of that sorts. Maybe I'll expand upon this blogpost and add details regarding that.

## ***Changes made with this mod:*** {#changes_made}

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

There are probably other things that I'm missing, but as usual, the mod can be grabbed from here:

