---
date: 2023-06-26 01:31:07
layout: post
title: "Pretty Draboat!! - Backend System"
subtitle: Exploring the Technical Foundations of my Pretty Draboat!!
description: This post use `GameDatabase` class as an example to demonstrate how does how Pretty Draboat!! this game was coded in backend. This class effectively demonstrates several key software engineering and game development concepts.
image: https://medias.wangruipeng.com/PrettyDrab-2-1.png
optimized_image: https://medias.wangruipeng.com/PrettyDrab-2-1.png
category: blog
tags: 
 - Code
 - blog
author: Wang Ruipeng
paginate: false
---
# Pretty Draboat!! - Backend System

This post use `GameDatabase` class as an example to demonstrate how does how Pretty Draboat!! this game was coded in backend. This class effectively demonstrates several key software engineering and game development concepts.

This unity project is made open-source on the unity version control system:

[wangrp/Pretty Draboat - Pretty Draboat - Plastic Hub (unity.cn)](https://plastichub.unity.cn/wangrp/Pretty-Draboat)

### **1. Understanding of Object-Oriented Concepts**

**Singleton Pattern**:

```csharp
private static GameDatabase instance;

private GameDatabase() {
    lastUpdate = System.DateTime.Now;
    baseSeed = lastUpdate.GetHashCode();
}

public static void Initialize() {
    if(instance == null)
        instance = new GameDatabase();
    // ... rest of the initialization code ...
}
```

This snippet demonstrates the Singleton pattern. The private constructor prevents external instantiation, and the static **`instance`** variable ensures only one **`GameDatabase`** object exists. The **`Initialize`** method creates the singleton instance if it's not already created, embodying the Singleton design pattern.

**Method Overloading and Polymorphism**:

```csharp
public static IReadOnlyList<T> GetItemsByInterface<T>() where T : IItem {
    // ... implementation ...
}

public static IReadOnlyList<T> GetItemsByItemType<T>() where T : BaseItem {
    // ... implementation ...
}
```

These methods showcase polymorphism and generic programming. They allow different types of item collections to be retrieved based on the item interface **`IItem`** or a specific base class **`BaseItem`**. This approach demonstrates flexibility and reusability in code design.

### **2. Modularity and Code Organization**

**Data Loading and Management**:

```csharp
c
public static void Initialize() {
    // ... initialization checks ...

    var templates = Resources.LoadAll<ItemTemplate>("Items/");
    // ... item loading logic ...

    var characterTemplates = Resources.LoadAll<CharacterDefine>("Characters/");
    // ... character loading logic ...

    var skillTemplates = Resources.LoadAll<SkillDefine>("Skills/");
    // ... skill loading logic ...
}
```

This **`Initialize`** method is a great example of modularity. It clearly separates the loading logic for different types of game data, such as items, characters, and skills. Each section is responsible for a specific aspect of game data initialization, demonstrating well-organized and modular code structure.

**Separation of Concerns**:

The same **`Initialize`** method and other parts of the **`GameDatabase`** class also illustrate separation of concerns. Each method is responsible for a distinct functionality, such as loading data, retrieving items, or handling character data, contributing to a clean and maintainable codebase.

### **3. Algorithm and Data Structure Usage**

**Gacha System Implementation**:

```csharp
public static CharacterDefine PickupCharaOnce(PickupPool pool, out bool newCh) {
    // ... implementation ...
}

public static void PickupCharaMultiTimes(PickupPool pool, ref CharacterDefine[] chRes, ref bool[] newCh, int times = 10) {
    // ... implementation ...
}
```

These methods demonstrate the use of algorithms in implementing a gacha system. They involve randomness and probability calculations to determine which characters a player receives. This part of the code highlights the understanding of algorithms and their application in game mechanics.

**Efficient Data Retrieval**:

```csharp
private Dictionary<int, BaseItem> items;
private Dictionary<int, CharacterDefine> characters;
private Dictionary<int, SkillDefine> skills;
```

The use of dictionaries for managing items, characters, and skills showcases efficient data storage and retrieval. It reflects the understanding of using appropriate data structures for optimal performance in a game environment.

Full code is attached for further reference:

```csharp
using Newtonsoft.Json;
using Sirenix.Utilities;
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using System.Runtime.CompilerServices;
using Unity.VisualScripting;
using UnityEditor;
#if UNITY_EDITOR
using UnityEngine;
#endif

namespace PrettyDraboat {

    /// <summary>
    /// 调用前需要获得Instance实例
    /// </summary>
    public class GameDatabase {

        private GameDatabase() {
            UnityEngine.Debug.Log("游戏数据库初始化");
            lastUpdate = System.DateTime.Now;
            baseSeed=lastUpdate.GetHashCode();
        }

        private static GameDatabase instance;

#if UNITY_EDITOR
        [InitializeOnLoadMethod]
#endif
        public static void Initialize() {
            //instance = new();
            if(instance==null)
                instance = new GameDatabase();
            var ins = instance;
            if (ins.hasInitialize)
                return;
            ins.hasInitialize = true;
            var templates = Resources.LoadAll<ItemTemplate>("Items/");
            int length=templates.Length;
            ins.items = new(length);
            for(int i = 0; i < length; ++i) {
                var item = templates[i].item;
                ins.items.Add(item.id,item);
            }

            var characterTemplates = Resources.LoadAll<CharacterDefine>("Characters/");
            length = characterTemplates.Length;
            ins.characters=new(length);
            for (int i=0;i < length; ++i) {
                var ch = characterTemplates[i];
                if (ch == null) {
                    UnityEngine.Debug.Log("Comes Null");
                }
                ins.characters.Add(ch.CharaID, ch);
            }

            ins.skills = new();
            var skillTemplates = Resources.LoadAll<SkillDefine>("Skills/");
            length = skillTemplates.Length;
            for(int i = 0; i < length; ++i) {
                var sk = skillTemplates[i];
                ins.skills.Add(sk.SkillID, sk);
            }
#if UNITY_EDITOR
            var json=System.IO.File.ReadAllText(Application.dataPath + "/Resources/playerInfo.json");
            PlayerInfo playerInfo = JsonConvert.DeserializeObject<PlayerInfo>(json);
            PlayerInfo.Initialize(playerInfo);
#endif
            PlayerInfo pInfo = JsonConvert.DeserializeObject<PlayerInfo>(jjson);
            PlayerInfo.Initialize(playerInfo);
            var charaHeadsSprites = Resources.LoadAll<Sprite>("Characters/characters_head");
            length = charaHeadsSprites.Length;
            for (int i = 0;i < length; ++i) {
                var chs = charaHeadsSprites[i];
                ins.character_heads.Add(chs.name, chs);
            }
        }

        /// <summary>
        /// 获得某一类物品的总数
        /// </summary>
        /// <typeparam name="T">T是为接口，不会影响功能的实现</typeparam>
        /// <returns>因为被缓存起来了，所以用了IReadOnlyList，不影响从中获得元素</returns>
        public static IReadOnlyList<T> GetItemsByInterface<T>() where T:IItem {
            if(GetItemsCache.TryGetValue(typeof(T),out var value)) {
                return value as List<T>;
            }
            var allItem=instance.items.Values;
            List<T> list = new();
            foreach(var item in allItem) {
                if(item is T) {
                    list.Add((T)(IItem)item);//太sb了这一坨东西
                }
            }
            GetItemsCache.Add(typeof(T),list);
            return list;
        }

        /// <summary>
        /// 获得某一类物品的数量
        /// </summary>
        /// <typeparam name="T">T是为BaseItem的派生类</typeparam>
        /// <returns>只读列表，会被缓存起来，故不方便被修改</returns>
        public static IReadOnlyList<T>GetItemsByItemType<T>()where T : BaseItem {
            if(GetItemsCache.TryGetValue(typeof(T), out var value)) {
                return value as List<T>;
            }
            var allItem=instance.items.Values;
            List<T> list = new();
            foreach(var item in allItem) {
                if(item is T) {
                    list.Add( (T)item);
                }
            }
            GetItemsCache.Add(typeof(T), list);
            return list;
        }

        /// <summary>
        /// 根据id获得角色的基础信息
        /// </summary>
        /// <param name="id">角色的id</param>
        /// <returns>该角色的基础定义，若id不存在，则返回null</returns>
        public static CharacterDefine GetCharaByID(int id) {
            if(instance.characters.TryGetValue(id, out var character)) {
                return character;
            }
            return null;
        }

        /// <summary>
        /// 获取所有角色的基础信息
        /// </summary>
        /// <returns>一个只读集合，用foreach去迭代吧</returns>
        public static IReadOnlyCollection<CharacterDefine> GetAllCharacterTemplates() {
            return instance.characters.Values;
        }

        /// <summary>
        /// 进行一次抽卡
        /// </summary>
        /// <param name="pool">抽取的卡池</param>
        /// <returns></returns>
        public static CharacterDefine PickupCharaOnce(PickupPool pool,out bool newCh) {
            GenerateNewSeed();
            var res = PickupCharacter(pool);//抽出的角色
            //while (res.star != CharacterDefine.InitStar.THREE) {
            //    res = PickupCharacter(pool);
            //}
            newCh = PlayerInfo.AddCharacter(res);//如果newCh为true，则代表抽到新角色         
            PlayerInfo.PickCost(pool.PickOnceCost);
            return res;
        }

        /// <summary>
        /// 进行多次抽卡
        /// </summary>
        /// <param name="pool">要抽的卡池</param>
        /// <param name="chRes">作为获得角色返回结果的数组</param>
        /// <param name="newCh">作为是否新角色返回结果的数组</param>
        /// <param name="times">抽卡次数，默认为10</param>
        /// <remarks>之所以使用这种特殊的传值方式，是因为返回数组难免要产生GC，所以建议是调用方使用ArrayPool创建共享数组以避免GC。不过反正也是我亲自写调用代码...</remarks>
        public static void PickupCharaMultiTimes(PickupPool pool,ref CharacterDefine[]chRes,ref bool[] newCh, int times=10) {
            GenerateNewSeed();
            for(int i = 0; i < times; ++i) {
                var res=PickupCharacter(pool);
                //while (res.star != CharacterDefine.InitStar.THREE) {
                //    res = PickupCharacter(pool);
                //}
                chRes[i] = res;
                newCh[i] = PlayerInfo.AddCharacter(res);
            }
            PlayerInfo.PickCost(pool.PickTensCost);
            return;
        }

        private static CharacterDefine PickupCharacter(PickupPool pool) {
            var ins = PlayerInfo.Instance;
            --ins.lastThreeStar;
            --ins.lastThreeUp;
            CharacterDefine res;
            //GenerateNewSeed();
            float rNumber; //Random.value;
            if (ins.lastThreeUp == 0 || ins.lastThreeStar == 0) {
                rNumber = 1.0f;
            }
            else {
                rNumber = Random.value;
                //GenerateNewSeed();
            }
            
            float x = pool.OneStarProbability;
            if (rNumber < x || rNumber == x) {//抽出一星
                if(pool.EnableOneStarUp) {
                    var prob = pool.OneStar.upProbability;
                    rNumber = Random.value;
                    GenerateNewSeed();
                    if (rNumber < prob || rNumber == prob) {//up
                        res=PickUpChara(pool.OneStar.up);
                    }
                    else {
                        res=PickNormalChara(pool.OneStar.normal);
                    }
                    return res;
                }
                res = PickNormalChara(pool.OneStar.normal);
                return res;
            }
            x += pool.TwoStarProbability;
            if(rNumber<x|| rNumber == x) {//抽出二星
                rNumber = Random.value;
                //GenerateNewSeed();
                var prob=pool.TwoStar.upProbability;
                if(rNumber < prob || rNumber == prob) {//UP
                    res=PickUpChara(pool.TwoStar.up);                   
                }
                else {
                    res = PickNormalChara(pool.TwoStar.normal);
                }
                return res;
            }
            //三星
            rNumber=Random.value;
            //GenerateNewSeed();
            var upProb = pool.ThreeStar.upProbability;
            ins.lastThreeStar = GameDefine.MaxThreeGuarantee;
            if(rNumber<upProb || upProb == rNumber) {
                ins.lastThreeUp = GameDefine.MaxUpGuarantee;
                res = PickUpChara(pool.ThreeStar.up);
            }
            else {
                res = PickNormalChara(pool.ThreeStar.normal);
            }
            return res;

            //为了实现不等分概率的PickUp
            static CharacterDefine PickUpChara(List<PickupPool.CharaProb> lists) {
                int length=lists.Count;
                var prob = Random.value;
                //GenerateNewSeed();
                var basic = 0f;
                CharacterDefine define = null;
                for(int i = 0; i < length; ++i) {
                    basic += lists[i].probability;
                    if (prob < basic || prob == basic) {
                        define = lists[i].character;
                        break;
                    }
                }
                return define;
            }

            static CharacterDefine PickNormalChara(List<CharacterDefine> lists) {
                int length = lists.Count;
                int index=Random.Range(0, length);
                //GenerateNewSeed();
                return lists[index];
            }
        }

        public static SkillDefine GetSkillByID(int id) {
            if (instance.skills.ContainsKey(id))
                return instance.skills[id];
            return null;
        }

        public static T GetSkillByID<T>(int id) where T:SkillDefine {
            instance.skills.TryGetValue(id, out var skill);
            return skill as T;
        }

        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public static void GenerateNewSeed() {
            var now = System.DateTime.Now;
            var timespan = now-lastUpdate;
            int random = Random.Range(0, int.MaxValue);
            Random.InitState((int)timespan.Ticks^baseSeed ^ random);
            lastUpdate = now;
            return;
        }
        
        // 使用角色名称获得头像, 垃圾代码
        public static Sprite getCharacterHead(string charaName) {
            instance.character_heads.TryGetValue(charaName, out var charaSprite);
            return charaSprite;
        }

        private bool hasInitialize = false;

        private static Dictionary<System.Type, object> GetItemsCache = new();

        private Dictionary<int,BaseItem> items;

        private Dictionary<int, CharacterDefine> characters = new();

        private Dictionary<int,SkillDefine> skills;

        private static System.DateTime lastUpdate;

        private static int baseSeed;

        private Dictionary<string, Sprite> character_heads = new();

        private static string jjson = "{\r\n  \"id\": 1,\r\n  \"budget\": 999999,\r\n  \"Name\": \"制作人\",\r\n  \"Characters\": [\r\n    {\r\n      \"id\": 25,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 1,\r\n      \"tutorID\": -1,\r\n      \"Star\": 3,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 2,\r\n      \"tutorID\": -1,\r\n      \"Star\": 3,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 23,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 3,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 4,\r\n      \"tutorID\": -1,\r\n      \"Star\": 3,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 5,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 6,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 7,\r\n      \"tutorID\": -1,\r\n      \"Star\": 3,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 8,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 24,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 27,\r\n      \"tutorID\": -1,\r\n      \"Star\": 3,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 9,\r\n      \"tutorID\": -1,\r\n      \"Star\": 1,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 10,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 11,\r\n      \"tutorID\": -1,\r\n      \"Star\": 1,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 12,\r\n      \"tutorID\": -1,\r\n      \"Star\": 1,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 13,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 22,\r\n      \"tutorID\": -1,\r\n      \"Star\": 1,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 14,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 15,\r\n      \"tutorID\": -1,\r\n      \"Star\": 1,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 26,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 16,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 17,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 18,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 19,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 20,\r\n      \"tutorID\": -1,\r\n      \"Star\": 2,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    },\r\n    {\r\n      \"id\": 21,\r\n      \"tutorID\": -1,\r\n      \"Star\": 1,\r\n      \"Level\": 1,\r\n      \"Exp\": 0\r\n    }\r\n  ],\r\n  \"Items\": {\r\n    \"100\": 999999,\r\n    \"101\": 999999,\r\n    \"102\": 999999,\r\n    \"103\": 999999\r\n  },\r\n  \"registDate\": \"0001-01-01T00:00:00\",\r\n  \"lastLoginTime\": \"0001-01-01T00:00:00\",\r\n  \"thisLoginTime\": \"0001-01-01T00:00:00\",\r\n  \"lastThreeStar\": 80,\r\n  \"lastThreeUp\": 160,\r\n  \"normalVoucher\": 999999,\r\n  \"deluxeVoucher\": 999999,\r\n  \"formation\": [1,2,3,4,5,6,7,8,9,10],\r\n  \"pickTicket\": 999999\r\n}";
    }

}
```