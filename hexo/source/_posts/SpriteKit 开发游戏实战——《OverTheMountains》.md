---
title: SpriteKit 开发游戏实战——《OverTheMountains》
categories:
  - iOS
date: 2019-04-02 18:20:35
tags:
  - SpriteKit
  - Game
---
## TL;DR

SpriteKit 早在 iOS 7 就已推出，但由于我的精力/水平有限，直到最近才开始学习它。

文章中的游戏《OverTheMountains》（翻山越岭）是我从零开始自学 SpriteKit 时，用了不到三天时间自己开发的一个小型 demo，并没有参考别人的代码，所以可能存在很多不合理的地方，还望各位不吝赐教，我将非常感激。

项目工程的 [GitHub 地址请戳我](https://github.com/iCodeWoods/OverTheMountains)。

<!-- more -->

## 游戏介绍

其实这类游戏在 AppStore 上已有很多，无非都是有一个小黑人，使用一根能不断伸长的棍子，看看最终能走多远。

而本作的游戏背景是，男主人公经过长途跋涉，翻山越岭地去见女主人公。（瞬间不一样了有木有😁）

![OverTheMountains](OverTheMountains.gif)

## 一些实现思路

尽管游戏本身很简单，但如果一开始没有一个良好的设计，很容易写着写着就乱了。

下面就粗略地说一下我的实现思路与步骤：

1. 添加两个矩形（也就是所谓的“山”）（左边的记为起点，右边的记为终点）
2. 将起点向左移出屏幕，将终点向左移至起点的位置，从屏幕右侧生成一个新的终点并移至合适的位置
3. 点击屏幕可以在起点处生成一个棍子，且棍子会随着按屏幕的时间增长而增长
4. 松手后棍子会顺时针旋转45度，并计算是否与终点相交
5. 在起点添加一个玩家，并可以实现双脚交替走路的动画
6. 当棍子旋转完毕后，让玩家走至棍子末端。如果相交，则进入下一关，也就是上面所说的第二步，同时让玩家也随着一起移动，并把棍子移除。如果不相交，则棍子旋转至垂直方向，同时玩家坠落悬崖~
7. 坠落后，进入开始界面，添加一个按钮，用户点击后能够再进入游戏界面，重新开始游戏
8. 计算得分，添加一个 Label 用于显示分数
9. 添加一个红心，如果命中红心可以获得额外的分数，计算棍子与红心是否相交
10. ……and so on

不难看出，整个游戏最复杂的部分在于第六步。但由于我们在此之前已经实现并且测试过了第二步，所以第六步的难度大幅下降。

其实从第一步开始，一点点的实现，即使不参照任何其他 demo，我们也能轻松的实现整个游戏。相反，如果一开始就把“起点”、“终点”、“棍子”、“玩家”、“红心”等元素全加上，反而会导致一片混乱。

## 一些实现细节

### 棍子的伸长

也就是上面所说的第三步。由于我刚开始学习 SpriteKit，所以思维还没有转变过来，一开始的想法还是使用一个 NSTimer 去处理。（demo 暂且先不考虑 timer 不会释放的问题）

```objective-c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    self.stickTimer = [NSTimer scheduledTimerWithTimeInterval:0.05 target:self selector:@selector(growStick) userInfo:nil repeats:YES];
    [[NSRunLoop mainRunLoop] addTimer:self.stickTimer forMode:NSRunLoopCommonModes];
}

- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self.stickTimer invalidate];
    self.stickTimer = nil;
}

- (void)growStick {
    self.stickNode.size = CGSizeMake(self.stickNode.size.width, self.stickNode.size.height + kStickGrowSpeed);
}
```

后来发现可以使用 `repeatActionForever:` 与 `runAction:withKey:` 方法实现，好处不仅是可以少声明一个属性（stickTimer）和一个方法（growStick），更重要的是现在它真正的融入 SpriteKit 的世界了，它的运行与结束都会由 SKNode 来替我们处理，我们不再需要关心它是否没有释放或者是不准之类的了。

在添加时指定一个 key，当想移除时，只需要移除指定 key 的 Action 即可。代码如下：

```objective-c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    SKAction *growAction = [SKAction repeatActionForever:[SKAction resizeByWidth:0 height:kStickGrowSpeed duration:0.1]];
    [self.stickNode runAction:growAction withKey:kStickGrowActionName];
}

- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self.stickNode removeActionForKey:kStickGrowActionName];
}
```

### 玩家的生成与双脚交替行走

也就是上面所说的第五步。其实就是将一组图片放入以`.atlas`结尾的文件夹中，然后使用苹果爸爸为我们封装好的 [SKTextureAtlas](https://developer.apple.com/documentation/spritekit/sktextureatlas?language=objc)。

![Atlas](Atlas.png)

```objective-c
- (PlayerNode *)playerNode {
    if (!_playerNode) {
        _playerNode = [[PlayerNode alloc] initWithTexture:self.playerWalkingFrames[0]];
    }
    return _playerNode;
}

- (NSMutableArray *)playerWalkingFrames {
    if (!_playerWalkingFrames) {
        _playerWalkingFrames = [NSMutableArray array];
        SKTextureAtlas *playerAnimatedAtlas = [SKTextureAtlas atlasNamed:kPlayerTextureAtlasKey];
        for (int i = 1; i <= playerAnimatedAtlas.textureNames.count; i++) {
            SKTexture *texture = [playerAnimatedAtlas textureNamed:[NSString stringWithFormat:@"player%d", i]];
            [_playerWalkingFrames addObject:texture];
        }
    }
    return _playerWalkingFrames;
}
```

### 棍子、起点、终点、玩家的组动画

也就是上面所说的第六步。这里涉及到很多并发执行的动画。

上面说实现思路与步骤的时候只是一个大概，没有完整地说。事实上这里面有一个问题，就是当我们按住屏幕伸长棍子，然后各种元素执行动画时，此时用户如果再点击屏幕，就不能再产生另一根棍子并执行一堆动画了。因此我们加了一个标志位，在 `touchesBegan` 和 `touchesEnd` 中进行判断，相当于加了一个锁吧。

```objective-c
#pragma mark - Touch Event

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    if (self.isPlaying) {
        return;
    }
    // do something...
}

- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    if (self.isPlaying) {
        return;
    }
    self.isPlaying = YES;
    // do something...
}
```

然后在所有的棍子、起点、终点、玩家等一系列元素全部执行完动画后，我们再重置 `isPlaying` 标志位。SKNode 执行 Action 动画是异步的，我们如何准确得知所有动画执行完的时机？由于我初学 SpriteKit，尚不知是否有简便方法，因此还是使用了平时 iOS 开发中所用到的 `dispatch_group_t` 来实现。当 group 里的操作全部执行完毕后，通过 `dispatch_group_notify` 来获知并重置标记位。代码如下：

```objective-c
- (void)overTheMountains:(BOOL)success {
    self.currentLevel++;
    
    // 玩家执行双脚交替行走的动画并前进
    [self.playerNode runAction:[SKAction repeatActionForever:[SKAction animateWithTextures:self.playerWalkingFrames timePerFrame:0.3f resize:NO restore:YES]] withKey:kPlayerWalkingActionKey];
    SKAction *forwardPlayerAction = [SKAction moveToX:endPointX duration:kPlayerMoveActionDuration];
    [self.playerNode runAction:forwardPlayerAction completion:^{
        [self.playerNode removeActionForKey:kPlayerWalkingActionKey];
        
        dispatch_group_t group = dispatch_group_create();
        
        // 移除起点
        dispatch_group_enter(group);
        SKAction *removeStartPillarAction = [SKAction moveToX:-CGRectGetWidth(self.startPillarNode.frame) duration:0.1];
        [self.startPillarNode runAction:removeStartPillarAction completion:^{
            [self.startPillarNode removeFromParent];
            dispatch_group_leave(group);
        }];
        
        // 把终点移至起点的位置
        dispatch_group_enter(group);
        [self.endPillarNode runAction:[SKAction moveToX:0 duration:kPillarMoveActionDuration] completion:^{
            self.startPillarNode = self.endPillarNode;
            self.endPillarNode = [self generateEndPillarNodeWithLevel:self.currentLevel];
            dispatch_group_leave(group);
        }];
        
        // 把玩家移至起点的位置
        dispatch_group_enter(group);
        [self.playerNode runAction:[SKAction moveToX:CGRectGetWidth(self.endPillarNode.frame) - CGRectGetWidth(self.playerNode.frame) - kStickWidth duration:kPillarMoveActionDuration] completion:^{
            dispatch_group_leave(group);
        }];
        
        // 移除棍子
        dispatch_group_enter(group);
        SKAction *removeStickAction = [SKAction moveToX:-CGRectGetWidth(self.stickNode.frame) duration:0.3];
        [self.stickNode runAction:removeStickAction completion:^{
            [self.stickNode removeFromParent];
            self.isPlaying = NO;
            dispatch_group_leave(group);
        }];
        
        // 所有动画完成后的通知，在此重置标记位
        dispatch_group_notify(group, dispatch_get_main_queue(), ^{
            NSLog(@"Group finished...");
            self.isPlaying = NO;
        });
    }];
}
```

### 游戏场景的切换

也就是上面所说的第七步。其实很简单，使用现成的 `presentScene:transition:` 方法，只需要选择我们自己喜欢的 Transaction 就好了~

```objective-c
// 游戏界面，闯关失败后进入开始界面
SKTransition *transition = [SKTransition revealWithDirection:SKTransitionDirectionDown duration:0.5];
[self.view presentScene:[BeginScene sceneWithSize:self.frame.size]; transition:transition];

// 开始界面，点击开始之后进入游戏界面
SKTransition *transition = [SKTransition doorsOpenHorizontalWithDuration:0.5];
[self.scene.view presentScene:[GameScene sceneWithSize:self.scene.view.frame.size] transition:transition];
```


## 一些未来的规划

上文所提及的都是一些非常基础的功能，实现并不复杂，但可玩性其实不高，无法持续吸引用户。

其实真正上手实践了之后，就会发现其实很多比较火的小游戏，难点不在于技术实现，而在于一个好的创意/想法/策划。

如果在 AppStore 上搜索“Stick Hero”，会发现已经有一大堆类似的应用，功能都差不多。这也是我为什么没有上架这款游戏的原因。想要脱颖而出，就得做出一些不同的点来。

- 既然游戏名叫《Over The Mountains》（翻山越岭），那山在哪呢喂！😂😂 所以未来有打算把现在的矩形“柱子”（PillarNode）换成“⛰”，山可能会有不同的形状、高度。
- 考虑加上一些符合故事背景的音乐，🌰：《越过山丘》之类的（不过应该还是轻音乐一点比较好。。。）
- 考虑加上四季与背景图、山的变化，例如场景会自动切换到冬天，然后⛰会变成雪山等。
- 考虑加上金币功能，命中红心、每日登陆、分享、看广告视频等都会获得金币，金币可以用于购买不同的人物形象/场景等。
- 翻过一定数量的山后（比如100座），男主角与女主角终于团聚（泪奔~~┭┮﹏┭┮）
- ……（其实能做的蛮多的）

等我有时间的时候，会继续完善这个游戏，并最终上线 AppStore（但愿我有空。。。😂）

The End.

