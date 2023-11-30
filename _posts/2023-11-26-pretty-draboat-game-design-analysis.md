---
date: 2023-06-26 20:15:03
layout: post
title: "Pretty Draboat!! - Game Design Analysis"
subtitle: Main game mechanism and its corresponding code implementation.
description: This analysis explores the design and code implementation of the Pretty Draboat!!, from its plot-driven and character-driven elements to its innovative Gacha system.
image: https://medias.wangruipeng.com/PrettyDrab-2-1.png
optimized_image: https://medias.wangruipeng.com/PrettyDrab-2-1.png
category: blog
tags:
 - Design
author: Wang Ruipeng
paginate: false
---
# Pretty Draboat!! - Game Design Analysis

[Click here to check the demo.](https://youtu.be/WRsrFzS_2HQ)

# About the game

In the game, players will assume the role of a dragon boat team leader, guiding a dragon boat team in competitions.

The story begins like this: 

> To promote traditional culture, the Chinese Provincial Education Bureau decides to hold a provincial-level high school dragon boat competition and mandates participation from every school. Yujun High School is inevitably tasked with forming both a boys' and a girls' dragon boat team. Due to an unexpected turn of events, the responsibility of preparing the girls' dragon boat team falls on a newcomer in the student council, who is the player. The player needs to recruit members to participate in the dragon boat race and win the ultimate provincial championship.
> 
> 
> Subsequently, the player, together with the 10 members already recruited, develops the Yujun High School girls' dragon boat team, completes daily training, and aims for the ultimate victory.
> 

![Untitled](https://medias.wangruipeng.com/PrettyDrab-2-1.png)

The game is a meticulously designed game with a core gameplay focused on character development. Throughout the game, players need to employ various strategies and methods to assemble and train their dragon boat team. These strategies and methods include, but are not limited to, attracting new team members by collecting resumes, or enhancing members' abilities and skills through the game's character development system. Additionally, players can boost the team's morale and increase the dragon boat's speed by accurately beating the drum during matches, thus securing more opportunities for victory. The character development system may include textbook-like learning materials, or upgrade items, used to enhance team members' theoretical knowledge and practical skills, thereby improving the team's competitiveness in races.

In the competition part of Pretty Draboat!! players take on the role of the dragon boat team's drummer. This role involves directing the boat's course and rhythm, ensuring the team generates maximum propulsion at the right moments. Players need to hit the drum at precise times to motivate the dragon boat team, thereby increasing the boat's speed and sprint ability. This gameplay emulates real dragon boat racing, offering players a unique and authentic experience.

# Gacha System

In designing the Gacha system for our game, our goal is to establish a unique mechanism that sets us apart from other games on the market. A key distinction of our game is that players need to deploy up to 10 characters in a match, compared to the 3-4 character setup common in other 2D Gacha games. This undoubtedly increases the complexity of character requirements and the volume of card-drawing resources in the game. Therefore, providing players with a more efficient way to acquire characters is a focal point for us, while also ensuring that the probability of obtaining the highest rarity level, Super Super Rare (SSR) characters, aligns with most games in the market. We set the draw probability for these super rare SSR characters at 1%, with corresponding probabilities for Super Rare (SR) and Regular (R) characters set at 10% and 89%, respectively. 

| Rarity | Probability |
| --- | --- |
| SSR | 1% |
| SR | 10% |
| R | 89% |

Additionally, we have conceptualized a probability bottom-up random enhancement mechanism to add flexibility to the card-drawing system. Based on the expected average 1% draw rate for SSR characters, we set the probability of drawing an SSR character in a single draw at 0.6% before the probability boost. Then, we will gradually increase the SSR draw probability from 0.6% to 100% linearly over 70 draws, ensuring at least one SSR character is drawn every 100 draws.

For the SR characters, our bottom-up mechanism guarantees at least one SR character in every 10 draws. Specifically, the probability is set at 5% for the first seven draws, 40% for the eighth, 70% for the ninth, and 100% for the tenth draw. This system aims to enhance the gaming experience and prevent player frustration from failing to draw high-star characters consecutively.

When both SSR and SR characters appear in the same draw, we process them in the order of SSR followed by SR, prioritizing the guarantee for SSR characters. The main logic is demonstrated here:

```csharp
using System;

public class CardDrawSystem
{
    private Random random = new Random();

    public int DrawSSR(int drawsSinceLastSSR, int drawsSinceLastSR)
    {
        double initialProbability = 0.6;
        double incrementPerDraw = (100 - initialProbability) / 70; 
        double currentProbability = drawsSinceLastSSR < 70 ? initialProbability + incrementPerDraw * drawsSinceLastSSR : 100;
        if (random.NextDouble() < currentProbability / 100)
            return 2; // SSR drawn
        else
            return DrawSR(drawsSinceLastSR); 
    }

    public int DrawSR(int drawsSinceLastSR)
    {
        double[] probabilities = { 5, 5, 5, 5, 5, 5, 5, 40, 70, 100 };
        double currentProbability = probabilities[Math.Min(drawsSinceLastSR, 9)];
        if (random.NextDouble() < currentProbability / 100)
            return 1; // SR drawn
        else
            return 0; // R drawn
    }
}

public class Gacha
{
    public static void Main()
    {
        CardDrawSystem cardDrawSystem = new CardDrawSystem();
        int drawsSinceLastSSR = 75; // Number of draws since the last SSR
        int drawsSinceLastSR = 9;   // Number of draws since the last SR

        int drawResult = cardDrawSystem.DrawSSR(drawsSinceLastSSR,drawsSinceLastSR);
        Console.WriteLine("Draw result: " + drawResult); // 0: R, 1: SR, 2: SSR
    }
}
```

Considering the substantial number of characters that can appear simultaneously in our game, we aim to provide players with more opportunities for card draws. However, this might result in an excessive number of Super Super Rare (SSR) characters. To address this, some games increase the minimum draw count for SSR characters to control their quantity, but this can make SSR characters feel more elusive, potentially reducing the game's appeal. Therefore, we seek to control the appearance rate of SSR characters while keeping the appearance rate of specific SSR characters similar to other games. Here is our designed card-drawing mechanism for determining which SSR characters to draw.

We plan to establish two SSR character pools: a permanent SSR pool and a limited SSR pool. Players' primary targets in the game are usually the limited SSR characters, as these cannot be obtained by other means. Within the same pool, there are two boosted limited SSR characters, each with a 30% chance of being drawn among all SSR characters. That is, once a card is identified as an SSR character, we further determine which specific SSR character it is. The card has a 30% chance of being one limited SSR character and another 30% chance of being a different limited SSR character, with the remaining 40% chance being a permanent SSR character.

We also wish for players to have a higher probability of drawing new SSR characters, thus necessitating adjustments to the card-drawing rules. We define two limited characters as Character A and Character B. If a player has already drawn `n` Character As and `m` Character Bs in the current pool, then the probability of Character A among all SSR characters will be 30% + 5%(m-n); similarly, the probability of Character B among all SSR characters will be 30% + 5%(n-m). The probabilities are capped at a maximum of 60% and a minimum of 0%. For instance, if a player has already drawn 6 Character As and 3 Character Bs, then the probability of drawing Character A in the next SSR draw would be 30% + (3-6)*5% = 15%, while the probability for Character B would be 30% + (6-3)*5% = 45%.

The main logic is as follows:

```csharp
using System;

public class SSRCharacterDrawSystem
{
    private Random random = new Random();

    public int DetermineSSRCharacter(int drawnA, int drawnB)
    {
        int baseProbabilityA = 30;
        int baseProbabilityB = 30;

        int adjustedProbabilityA = Math.Max(0, Math.Min(60, baseProbabilityA + 5 * (drawnB - drawnA)));
        int adjustedProbabilityB = Math.Max(0, Math.Min(60, baseProbabilityB + 5 * (drawnA - drawnB)));

        int draw = random.Next(1, 101);
        if (draw <= adjustedProbabilityA)
            return 1; // Character A
        else if (draw <= adjustedProbabilityA + adjustedProbabilityB)
            return 2; // Character B
        else
            return 0; // Other characters
    }
}

public class SSRCharacter
{
    public static void Main()
    {
        SSRCharacterDrawSystem ssrCharacterDrawSystem = new SSRCharacterDrawSystem();
        int drawnCharacterA = 6; // Number of Character As drawn
        int drawnCharacterB = 3; // Number of Character Bs drawn

        int ssrCharacterDrawn = ssrCharacterDrawSystem.DetermineSSRCharacter(drawnCharacterA, drawnCharacterB);
        Console.WriteLine("SSR Character drawn: " + ssrCharacterDrawn); // 1: Character A, 2: Character B, 0: Other SSR Characters
    }
}
```

# Character Design

In our game, the design of 2D character roles generally relies on two methods: character-driven and plot-driven. Let's first look at plot-driven design.

The plot-driven design pipeline often begins with the team members identifying a classic scene, or a "famous scene." These plot points often provide players with a key narrative memory, such as two best friends finding out they have feelings for the same boy, or a character who always finishes last in training eventually becoming the top of their class. These are often pivotal moments in the storyline that can influence a character's actions and even their personality later on. For example, in the design of the characters "Lan Si Xi," "Ye Qing Lan," and "Zhu Cang Liu," our team decided through brainstorming that the three characters would develop admiration for each other and set a scene where they eat together in the cafeteria, subtly testing each other. Based on this scenario, we needed to create character settings that make this scene plausible. We considered why the three would admire each other, focusing on each character having certain flaws and admiring others for their strengths in areas where they are weak.

![Untitled](https://medias.wangruipeng.com/PrettyDrab-2-2.png)

Character-driven design, on the other hand, relies on the character's own personality to shape the story. After selecting a character's personality, the main focus is on stimulating the player's affection and curiosity. A negative example would be describing the character's background extensively at the beginning of the game, which might bore the player. Thus, our character design should first spark the player's curiosity. Mystery has a strong attraction and can effectively stimulate a player's desire to continue playing. Detailed stories can be shared once the player becomes genuinely interested.

For a game, the opening scene should briefly mention the character's origin and motivation to spark players' curiosity about the character. Sometimes an illustration, an action, or a line of dialogue can be more effective than a lot of text. During character development, we need to understand how to appropriately restrain and leave blanks, providing space for players to explore and get to know the characters better. As the story progresses and players make their own choices, the character's mysterious veil should be lifted gradually.

In this process, characters should be like enigmatic puzzles, constantly drawing players to explore and understand them deeper. Every subtle revelation, whether through dialogue, action, or plot development, heightens the player's anticipation for the next secret to be unveiled, deepening their desire to understand the full picture of the character. Such a method of character development not only engages players more deeply in the game but also makes the characters themselves more intriguing and appealing.